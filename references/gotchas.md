# Modular Avatar gotchas

Hard-won pitfalls discovered while working with MA 1.18 / NDMF in real projects. Read this before adding MA components, rebuilding menus, or troubleshooting silent failures.

## 1. ObjectToggle: `referencePath` must be set

**Symptom**: An MA Object Toggle is added and the menu item shows up, but the toggle silently does nothing.

**Cause**: `ModularAvatarObjectToggle.Objects[i].Object.referencePath` was left empty. MA's `AvatarObjectReference.Get()` returns `null` when `referencePath` is empty, so the build generates a no-op.

**Fix**: Always set the full avatar-relative path:

```csharp
objRef.FindPropertyRelative("referencePath").stringValue = "Clothes/Default/Cardigan";
objRef.FindPropertyRelative("targetObject").objectReferenceValue = target.gameObject;
```

**Verify** after setting: read the component back and confirm `Objects[0].Object.referencePath` equals the path, not `""`.

## 2. ObjectToggle must be under (or on) its MenuItem

**How MA binds the parameter**: a Reactive component (`ObjectToggle`/`ShapeChanger`) walks up the parent chain from its own GameObject to the nearest `ModularAvatarMenuItem`, and uses that item's parameter as the trigger condition (see `ReactiveObjectAnalyzer.LocateReactions.BuildConditions`).

**Consequence**:
- The `MenuItem` and `ObjectToggle` must live on the **same GameObject** or the ObjectToggle must be a **child** of the MenuItem's GameObject.
- The menu-tree node is separate from the clothing node. Put `MenuItem` + `ObjectToggle` on the menu-tree node, and let `ObjectToggle.Objects[]` reference the real clothing node by path. Do not put MA components directly on the clothing node.

## 3. `_OFF` parameters are inverted: set `Active = false`

**Symptom**: Toggle appears to do nothing, or the object is on when it should be off (and vice versa).

**Cause**: Parameters named like `Cardigan_OFF`, `Bottoms_OFF`, `Crown_OFF`, `Slippers_OFF` mean "param = 1 ⇒ hide". But `ObjectToggle.Active = true` means "when the parameter triggers, set the object active". These are opposite.

**Fix**: For OFF-style parameters set `Active = false`:

```csharp
elem.FindPropertyRelative("Active").boolValue = false;
```

Semantics: param 0 (default) → object keeps scene state (shown); param 1 (menu clicked) → object set inactive (hidden).

## 4. Moving nodes breaks FX animation root paths

**Symptom**: After grouping nodes under `Clothes/Default/...`, `Hair/Default/...`, etc., menu sliders/toggles stop working and Console shows missing binding warnings.

**Cause**: FX anim clips bind by root-relative path, e.g. `Cardigan.anim` uses `path: Cardigan`. Moving the node to `Clothes/Default/Cardigan` makes the binding resolve to nothing.

**Mitigations**:
- Toggle-style controls (m_IsActive on a single object): rebuild with MA ObjectToggle (this skill's approach).
- BlendShape sliders (BreastSize, Sleeve, BLength, FHLength, TWLength, ...): MA has no continuous-blendshape equivalent, so either keep the FX layer and rewrite the broken paths, or accept discrete states. Do not delete the FX slider layers.
- Run a binding check after any reparent: enumerate clips and find bindings whose `path` no longer resolves under the avatar root.

## 5. MenuInstaller binding rules

`ModularAvatarMenuInstaller` is ignored unless it has a `menuToAppend` asset **or** a `MenuSource` component on the **same GameObject** (`VirtualMenu.cs`: `if (installer.menuToAppend == null && installer.GetComponent<MenuSource>() == null) return;`).

**Working layout**:
- Put `ModularAvatarMenuInstaller` **and** `ModularAvatarMenuGroup` on the avatar root.
- Point the MenuGroup's `targetObject` at the menu tree root (e.g. `_MA_Menu`).
- The menu tree root contains the top-level MenuItems (`Body Menu`, `Face Menu`, ...) with `MenuSource = Children`.
- Each submenu's children are its menu items.

A MenuGroup is **not** needed on the menu-tree root itself; having it there (plus one on the avatar root) causes duplicate traversal.

## 6. Keep the menu tree clean; delete stray MA components

- `_MA_Menu` must contain only the intended menu-tree children. A duplicate node (e.g. a stray `Costume Menu`) or a self-referencing child (`_MA_Menu` nested inside itself, often with a stray `ModularAvatarMenuInstallTarget`) corrupts menu installation.
- When prototyping MA components directly on a clothing node, remove them afterwards. A leftover `ObjectToggle` on the clothing node competes with the menu-tree toggle and produces undefined behavior (verify with a search for all `ModularAvatarObjectToggle` instances and their parent paths).
- After cleanup, re-read the components rather than assuming deletion worked.

## 7. Validate the build, not just the inspector

The Inspector showing a component does not prove the toggle works. After any MA change:

```csharp
var clone = nadena.dev.ndmf.AvatarProcessor.ProcessAvatarUI(root.gameObject);
```

Then verify on the clone:
- `VRCAvatarDescriptor.expressionsMenu` contains your menu items (watch for auto-added `More` overflow submenus when > 7 root controls);
- the FX controller gained `MA Responsive: <name>` layers for each ObjectToggle;
- the submenu controls have the expected type/parameter/value.

Destroy the clone afterwards (`Object.DestroyImmediate`). This is a preview; it does not publish or modify the source scene.

## 8. RadialPuppet: parameter goes in `subParameters`, not `parameter`

**Symptom**: A RadialPuppet menu item shows up but the radial dial does nothing (no parameter is driven).

**Cause**: For `RadialPuppet` controls the radial rotation parameter belongs in **`Control.subParameters[0]`**, and **`Control.parameter` must be empty**. Putting the parameter in `Control.parameter` instead binds the wrong field and the dial is inert. Verified against the original `VRCExpressionsMenu` asset (`type: 203`, `parameter.name` empty, `subParameters: - name: BreastSize`).

**Correct MA MenuItem serialized config**:
- `Control.type = 203` (RadialPuppet)
- `Control.parameter.name = ""` (empty)
- `Control.subParameters[0].name = "BreastSize"` (the radial parameter)

**Wrong**:
- `Control.parameter.name = "BreastSize"` with `subParameters` empty.

**Verify** after building: on the clone menu, RadialPuppet controls show `param=""` and `subParams=[<name>]`.

## 9. EditorOnly tag silently excludes a node from ObjectToggle animation

**Symptom**: A group ObjectToggle references N nodes, but the build generates only N-1 `MA Responsive` layers. One specific node is missing with no Console error.

**Cause**: That node's GameObject has the `EditorOnly` tag. EditorOnly objects are stripped at build/runtime, so MA skips generating animation for them. The node still exists in edit mode, so inspector checks pass and the reference resolves — only the generated animation is missing.

**Bigger variant**: a **container/directory-level `EditorOnly` tag strips the ENTIRE subtree at build**. The mango.milfy 08.05 avatar had `Clothes` tagged EditorOnly → all clothing meshes vanished from the built clone (menus/params/toggles all fine, but no cloth nodes at all), while `Hair` and other dirs were Untagged and kept their meshes. **When "a whole category of nodes disappears after build", check the tag of the container/parent GameObject first.**

**Diagnosis**: Compare the `tag` of every target node; the missing one is `EditorOnly` while the others are `Untagged`. For whole-subtree loss, check parent/container tags.

**Fix**:
```csharp
node.gameObject.tag = "Untagged";
```

**Verify** after fixing: rebuild and confirm the `MA Responsive: <name>` layer now exists for that node (layer count should equal node count).

## 10. Outfit containers: keep inactive in scene; let MA isDefault control default

**Symptom**: An outfit/container GameObject is set `activeSelf=true` in the scene, so the avatar uploads wearing it regardless of menu state, or a non-default outfit is shown on load.

**Cause**: When building outfit-suite toggles (MA ObjectToggle controlling a container like `Clothes/Default`), the container's scene active state acts as the fallback. Setting it active in edit mode hard-codes "this outfit is worn" and fights the menu toggle.

**Correct pattern**:
- Keep **every outfit container inactive** in the scene (`activeSelf=false`), including the default outfit.
- Let the **menu item's `isDefault=True`** decide which outfit is worn on load — MA builds the animation to enable it.
- Verify on the built clone: `Clothes$Default$14 activeInHierarchy=True`, `Clothes$OtherOutfit$13 activeInHierarchy=False`.

## 11. Independent-armature outfits need OutfitRoot + MergeArmature + MeshSettings

**Symptom**: A newly added outfit (own armature, bones point at its own skeleton) doesn't follow the body, or MA doesn't treat it as a switchable suite.

**Fix**: On the outfit root add:
- **ModularAvatarOutfitRoot** — marks it as an independent outfit module so MA treats it as a switchable suite.
- **ModularAvatarMergeArmature** on its armature root — `mergeTarget` = the avatar's main armature; MA matches shared bones by name (prefix/suffix auto-inferred) and keeps outfit-specific bones.
- **ModularAvatarMeshSettings** — mesh bounds/anchor configuration.

Verify `GetBonesMapping().Count > 0` after adding MergeArmature, and confirm built skeleton puts outfit bones under the avatar's main armature (`Armature/Hips/...`).

## 12. Menu item count per submenu: avoid the auto "More" overflow page

**Symptom**: MA adds a `More` overflow submenu automatically, splitting your controls.

**Cause**: A VRChat submenu supports at most 8 controls (7 + overflow trigger). When a suite menu has many part toggles, MA auto-splits.

**Prefer**: keep per-suite part toggles flat within a suite menu when under the limit; if you exceed 8 controls, deliberately organize into logical sub-groups rather than relying on the auto `More` page. Re-check the built menu for unexpected `More` entries.

## 13. `automaticValue` silently breaks suite mutual exclusion

**Symptom**: Multiple suites share a `cloth`/`hair` parameter for exclusive switching, but toggling one suite doesn't turn off the others, or the built param type is `Bool` instead of `Int`.

**Cause**: MA's `ParameterAssignerPass` overwrites values for MenuItems with `automaticValue=true` (the default):
- Default value comes only from `list.FirstOrDefault(m => m.isDefault && !m.automaticValue)?.Control?.value` — an `isDefault` item **with `automaticValue=true` does NOT set the default**;
- If an `isDefault`+`automaticValue` param has a single menu item, the implicit default is `1`; if there are **multiple** items (mutual-exclusion), the default falls back to **0** → nothing active on startup;
- non-default item → for **Bool/Float** params forced to `1` ("just set it to 1") — no exclusion possible;
- non-default item → for **Int** params auto-assigned an increasing value — exclusion works.

So with Bool/Float params (or auto values), two suite toggles both end up `value=1` and never switch; and with a default suite left `automaticValue=true`, the built param default becomes `0` (nothing shown on startup).

**Fix**: set **every** suite `all` toggle (including the default suite) to:
```csharp
so.FindProperty("automaticValue").boolValue = false;
so.FindProperty("Control.value").floatValue = <distinct per suite>; // Default=1, next=2, ...
```
Default suite: `isDefault=true` + `automaticValue=false` + `value=1` (its value becomes the param default). Non-default suites: `isDefault=false` + `automaticValue=false` + distinct value.

**Verify**: built `cloth`/`hair` param type is `Int` **and default is `1` (not 0)**; default container active, non-default containers inactive.

## 14. Commercial MA modules ship complete config — don't rebuild, integrate

**Symptom**: Imported assets (e.g. Nova) are full MA modules, not plain outfits. Treating them as a new suite breaks their built-in behavior or duplicates menus.

**Recognition**: The module root has its own `ModularAvatarMenuInstaller` + `ModularAvatarMenuItem` (type=103, subMenu → its own VRCExpressionsMenu asset), plus its own `MergeAnimator`/`MergeArmature`/`MeshSettings`/`BlendshapeSync` and parameters (e.g. `N_Wing`). The shop pack often has a per-model prefab (`@Prefab/<Model>/@Modular/`).

**Integration**:
- Place the module under the correct category folder (`Hair/`, etc.).
- **Keep all its MA components** — its menu auto-installs via its MenuItem (MenuItem acts as MenuSource, so its MenuInstaller is active even with menuToAppend=null).
- In `_MA_Menu/<menu>`, create a module submenu that wraps the module's own MenuItem as a child, and add an `all` suite toggle (shared param, manual value, see #13).
- Module containers start inactive; default via `isDefault`.
- The module's detail toggles (wing/tail etc.) are driven by its own controller — leave them as-is.

## 15. Moved commercial MA module: controller animation paths break (fix via relativePathRoot)

**Symptom**: A commercial MA module's controller animations (e.g. `Nova/~Wing`) break after moving the module into a subfolder (`Hair/Nova`). Detail toggles (wings/tail) stop working.

**Cause**: The module's controller paths start with the module name (`Nova/...`), assuming it sits at the avatar root. After moving, those paths resolve to nothing. MA's MergeAnimator doesn't rewrite them by default.

**Fix** (don't edit commercial animation assets): set the module's **MergeAnimator `relativePathRoot`** to the module's parent, with `pathMode=Relative`:
```csharp
rpr.FindPropertyRelative("referencePath").stringValue = "Hair";
rpr.FindPropertyRelative("targetObject").objectReferenceValue = hair.gameObject;
so.FindProperty("pathMode").intValue = 0; // Relative
```
MA prepends the root-relative prefix, so `Nova/~Wing` → `Hair/Nova/~Wing` and resolves.

**Verify**: built module animation bindings all resolve (e.g. 31/31 OK).

## 16. Moved SMR node: bones still reference the old prefab Armature

**Symptom**: A SkinnedMeshRenderer node (especially cloned from a prefab) no longer follows the body after being moved/reparented.

**Cause**: The SMR `bones` array still references the **source prefab's Armature** (`Milfy_Another/Armature/...`, not present in scene) instead of the avatar's main Armature. Reparenting doesn't rebind them.

**Diagnosis**: count how many `smr.bones` are actually under the avatar's main Armature. Broken case: `0/222` (all point to a missing prefab root).

**Fix**: rebind bones by name to the main Armature, and set rootBone:
```csharp
var nameMap = mainArm.GetComponentsInChildren<Transform>(true).ToDictionary(t => t.name, t => t);
foreach (ref var bone in smr.bones) if (nameMap.TryGetValue(bone.name, out var t)) bone = t;
smr.rootBone = avatarRoot.Find("AutoAnchorObject");
```

**Note**: bone **names** matching (222/222) does not mean the Transform references are valid — always check the actual referenced objects.

## 17. NDMF-plugin assets generate their own menu/params (no manual menu needed)

**Symptom**: An imported asset (e.g. `nyappu.lightcontroller`) auto-generates a "Lighting Setting" submenu and `LightController/*` parameters on build, without any manual setup.

**Recognition**: The asset ships an `Editor/NDMFPlugin.cs` (`ExportsPlugin`), and its component has an `installTargetMenu` field. The generator creates MA components (MergeAnimator with pathMode=Absolute + Parameters + MenuInstaller + MenuItem) at build time.

**Behavior**:
- `installTargetMenu=null` → auto-creates a submenu (e.g. "Lighting Setting") installed to the avatar menu root;
- auto-generates params (`LightController/LightLimit` etc.);
- auto-generates an animation controller.

**Note**: don't hand-build a menu for these; just build-verify it doesn't conflict. If you need it elsewhere, set `installTargetMenu` to redirect.

## 18. Standalone accessory (e.g. elf ears): use BoneProxy, not OutfitRoot

**Symptom**: Adding a small independent-bone accessory (elf ears prefab with its own `Armature.E.ears`) to an avatar. Putting **ModularAvatarOutfitRoot** on it makes its parent suite container (e.g. `Hair/Default`) **inactive in the built clone** (aSelf=False) even though the `all` toggle and params are correct.

**Cause**: OutfitRoot tells MA to treat the object as an independent suite, changing the Responsive-layer logic for its parent container.

**Fix**: for accessories that just need to follow a body bone (no armature merging):
- Instantiate the prefab at the avatar root (e.g. `E.ears Variant`), not inside a suite container;
- add **ModularAvatarBoneProxy**, target = a main Armature bone (e.g. `Head`);
- optionally add ModularAvatarMeshSettings.
- After build, accessory bones land under the target bone (`parent=Head`) and follow the body; the SMR stays aHier=True.

**Reserve OutfitRoot for true independent-armature outfits that need MergeArmature.**

## 19. Multiple avatars sharing a scene / shared expression assets

**Scenario**: Scene holds several avatar roots (`mango.milfy.26.08.02/05/06`) sharing the same `VRCExpressionsMenu` + `VRCExpressionParameters` asset (same instance ID).

**Gotchas**:
- **Don't hardcode the avatar root name** in automation; a version rename silently breaks `root.Find("...")`. Dynamically find the active avatar.
- **Building an inactive avatar gives unreliable output** (SMR=0 or truncated clone). Set the target avatar active before building.
- Shared expression assets are **instantiated per clone at build** — the original shared asset is never mutated, so "cloth/hair missing in the shared asset" is expected and fine; the build clone has them.

## 20. Localizing MA menu names to Chinese

**Safe to rename** (menu text only): pure MenuItems with fixed param names (`Nail`, `FHSharp`...), fixed-param sliders, `all` toggles (param=cloth/hair), fixed-param prefab toggles (`Smartphone`).

**Do NOT rename**: ObjectToggle nodes with empty param (auto `__MA/AutoParam/<name>$hash` — renaming breaks the auto param), auto-param sliders, suite brand names, commercial module internals.

When writing Chinese names via codedom, use `\uXXXX` escapes (direct chars become `?`). Verify actual names via base64 of the UTF-8 bytes.

## Diagnostics summary

When a toggle "doesn't work", check in order:

1. `Objects[].Object.referencePath` non-empty and correct (gotcha #1).
2. ObjectToggle shares a GameObject with (or is a child of) its MenuItem (gotcha #2).
3. `Active` polarity matches parameter semantics (#3).
4. No stray MA components on the real clothing nodes (#6).
5. MenuInstaller has menuToAppend or a same-GameObject MenuSource (#5).
6. FX layer has the `MA Responsive: <name>` layer after build (#7).
7. RadialPuppet dials have their parameter in `subParameters`, not `parameter` (#8).
8. Target nodes do not carry the `EditorOnly` tag (see #9) — compare tag of every target against siblings that DO get a responsive layer.
9. Outfit containers are inactive in the scene and rely on menu `isDefault` (#10).
10. Independent-armature outfits have OutfitRoot + MergeArmature (#11).
11. Per-suite menu stays under 8 controls to avoid auto `More` overflow (#12).
12. Suite `all` toggles on non-default suites set `automaticValue=false` + distinct value (#13).
13. Commercial MA modules keep their own components; only wrap + add suite toggle (#14).
14. Moved commercial modules: check controller paths; fix via MergeAnimator `relativePathRoot` (#15).
15. Moved SMR nodes: verify bones reference the main Armature, rebind by name if not (#16).
16. NDMF-plugin assets auto-generate menus/params; no manual menu needed (#17).
17. Standalone accessories: BoneProxy to follow a bone; OutfitRoot on a plain accessory breaks its container's built state (#18).
18. Scene with multiple avatars: build only active avatars, don't hardcode root names (#19).
19. Menu localization: never rename auto-param ObjectToggle nodes or auto sliders (#20).
