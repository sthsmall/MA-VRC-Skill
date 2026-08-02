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

## Diagnostics summary

When a toggle "doesn't work", check in order:

1. `Objects[].Object.referencePath` non-empty and correct (gotcha #1).
2. ObjectToggle shares a GameObject with (or is a child of) its MenuItem (gotcha #2).
3. `Active` polarity matches parameter semantics (#3).
4. No stray MA components on the real clothing nodes (#6).
5. MenuInstaller has menuToAppend or a same-GameObject MenuSource (#5).
6. FX layer has the `MA Responsive: <name>` layer after build (#7).
7. RadialPuppet dials have their parameter in `subParameters`, not `parameter` (#8).
