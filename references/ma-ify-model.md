# Convert an existing model to MA-driven (模型 MA 化)

Turn a VRChat avatar that was originally driven by a monolithic FX controller / VRCExpressionsMenu assets into a Modular Avatar-driven setup, after reorganizing its hierarchy. Use this when the avatar's nodes are being regrouped (e.g. into Clothes/Hair/Accessories) and the change breaks FX animation root paths.

This workflow is the combined experience from a full MA-ification pass on a VRCSDK 3 avatar with MA 1.18 / NDMF. Read `references/gotchas.md` first — nearly every step below has a corresponding pitfall.

## When to use

- The avatar's top-level meshes are being reorganized into category groups (`Clothes/Default/...`, `Hair/Default/...`, `Accessories/Default/...`).
- Reparenting breaks FX anim clip root paths (e.g. `Cardigan.anim` binds `path: Cardigan`, which no longer resolves).
- You want menu toggles owned by MA instead of hand-written FX layers, while keeping blendShape sliders in the FX layer.

## Phase 1 — Reorganize the hierarchy

1. Inventory every top-level child of the avatar root: name, SkinnedMeshRenderer, blendshape count, materials, rootBone.
2. Classify: system nodes that must stay at root vs. clothing/hair/accessory nodes that can be grouped.
3. **Keep at avatar root** (do not move): `Armature`, the body/base meshes that carry most blendshapes (`Body`, `Body_base`), `AutoAnchorObject` (it is the rootBone of every skinned mesh), `Ground`/colliders, `VRCHeadChop`, `AutoAnchorObject`.
4. Create category group nodes under the avatar root with a `Default` sub-folder each:
   - `Clothes/Default/` — clothing meshes;
   - `Hair/Default/` — hair meshes and hair accessories;
   - `Accessories/Default/` — jewelry/headgear/other.
   New items later go as a sibling folder, e.g. `Clothes/MyNewOutfit/`.
5. Reparent the classified meshes under the `Default` nodes (preserve world transform: `SetParent(target, true)`).
6. Group nodes are empty GameObjects at identity transform; they may later host MA components.

## Phase 2 — MA foundation

1. Add `ModularAvatarParameters` on the avatar root; declare every parameter used by menus/animators (Bool/Int/Float, saved flag, default value).
2. Add `ModularAvatarMenuInstaller` on the avatar root.
3. Add `ModularAvatarMenuGroup` on the avatar root, `targetObject` = the menu-tree root GameObject (e.g. `_MA_Menu`).
   - The MenuInstaller is only active if it has `menuToAppend` **or** a `MenuSource` (MenuGroup) on the same GameObject.
4. Create the menu tree root `_MA_Menu` as a child of the avatar root (no MA component needed on it; do not add a MenuGroup there too).

## Phase 3 — Build the MA menu tree

1. Under `_MA_Menu`, create one submenu per top-level menu section (Body, Face, Hair, Costume, Facial Set, etc, KumaPhone). Each submenu node has `ModularAvatarMenuItem` with:
   - `Control.type = 103` (SubMenu);
   - `MenuSource = 1` (Children) so its children become menu items.
2. Under each submenu, create a node per control with `ModularAvatarMenuItem`:
   - Toggle → `Control.type = 102`, `Control.parameter.name = "<param>"`, `Control.value = 1`.
   - Slider/RadialPuppet → `Control.type = 203`, `Control.parameter.name = ""` (empty), `Control.subParameters[0].name = "<param>"` (see gotcha #8).
   - Int-select (Nail/FacialType) → Toggle items each carrying a distinct `Control.value`.
3. Clear the avatar descriptor's `expressionsMenu` so MA builds the menu from scratch (otherwise the old VRCExpressionsMenu contents are appended and duplicate MA items).
4. **Icon reuse (Milfy pattern)**: reuse the commercial menu's icon textures directly — set `PortableControl.Icon` to the original `Texture2D` loaded from the commercial menu's icon folder (e.g. `Assets/.../VRC/EXMenu/Icons/*.png`). The original icons remain untouched assets; only the MA MenuItem references them. Assign icons to top-level submenus, `all` masters, part toggles, and sliders for a native look.
5. **System/legacy menu dedup**: if you keep the original full menu as a `MenuSource = MenuAsset` submenu (e.g. `System Menu`), it duplicates every toggle/slider you also rebuilt in MA. Fix: convert that submenu to `MenuSource = Children` and rebuild it as an MA tree, dropping items you now own (cloth/hair toggles) and keeping the rest (body sliders, Face, Facial Set, etc, KumaPhone, Nail). Verify in the build that each control exists exactly once across the whole tree.

## Phase 4 — Toggle controls via MA ObjectToggle

Use this for single-object on/off switches that were previously FX layers.

1. On the menu-item node add `ModularAvatarMenuItem` **and** `ModularAvatarObjectToggle` on the same GameObject (gotcha #2).
2. Set `Objects[0].Object.referencePath` to the full avatar-relative path of the real mesh node, e.g. `Clothes/Default/Cardigan`, and set `targetObject` (gotcha #1).
3. For `_OFF`-style parameters set `Active = false` so the param=1 state hides the object (gotcha #3).
4. Delete the corresponding FX controller layer (e.g. `Cardigan_ON/OFF`) to avoid double control.

## Phase 5 — Sliders / blendshapes stay in FX

MA has no continuous-blendshape equivalent; keep sliders in the FX layer.

1. Rewrite every FX anim clip binding whose root path no longer resolves: old `path: Cardigan` → `path: Clothes/Default/Cardigan`, etc. **Scan the entire `Animation/` folder, not just clips reachable from the FX controller** — AFK/pose/hand clips can also bind clothing root paths (e.g. `Milfy_AFK_*` binds `Cardigan`). Implement as: `AnimationUtility.GetCurveBindings(clip)`, read `GetEditorCurve`, delete old binding, re-add with rewritten `binding.path`, `EditorUtility.SetDirty` + `AssetDatabase.SaveAssets`.
2. **Shared-clip warning**: those rewritten clips are commercial assets shared by sibling avatar variants (e.g. `Milfy_kisekae` prefabs use the same clips). Rewriting the path fixes the current avatar but breaks variants that still use the old top-level layout. Decide up front: rewrite in place (accepting variant breakage) or copy clips/controller first. Verify the target variant after the change.
3. Keep the FX slider layers (BreastSize, Sleeve, BLength, FHLength, TWLength, TWVolume, ...).
4. Multi-object + blendshape combos (e.g. HairNoSide toggling several objects plus blendshapes) are also just path fixes — no MA rewrite needed.

## Phase 6 — Validate the build

```csharp
var clone = nadena.dev.ndmf.AvatarProcessor.ProcessAvatarUI(root.gameObject);
```

Verify on the clone:
- `expressionsMenu` contains all menu items, no duplicates, no stray old-menu overlap;
- RadialPuppet controls show `param=""` and `subParams=[<name>]`;
- the FX controller gained `MA Responsive: <name>` layers for each ObjectToggle;
- all FX anim bindings resolve under the avatar root (root-path binding scan returns 0 broken);
- **the default outfit container is active in the built clone, others inactive** (menu `isDefault` drives default wear, not scene state);
- Console has no new errors/warnings vs. baseline.

Destroy the clone afterwards (`Object.DestroyImmediate`). This preview does not publish.

## Rollback

- Keep the original FX controller and expression menu assets untouched until the MA build is verified.
- If MA build is wrong, re-point the descriptor's `expressionsMenu` back to the original asset and restore any deleted FX layers.
- The hierarchy reorder itself can be undone with Unity Undo or by restoring from version control.

## Checklist recap

- [ ] System nodes stay at root; meshes grouped into Clothes/Hair/Accessories/Default.
- [ ] MA Parameters, MenuInstaller + MenuGroup on avatar root.
- [ ] `_MA_Menu` tree with submenus (`MenuSource=Children`) and menu items.
- [ ] `expressionsMenu` cleared; MA builds from tree.
- [ ] ObjectToggle nodes carry MenuItem + ObjectToggle with correct `referencePath`, `Active` polarity.
- [ ] Target nodes carry `Untagged`, not `EditorOnly` (EditorOnly nodes are stripped and get no responsive layer).
- [ ] Old FX toggle layers deleted; slider layers kept and paths rewritten.
- [ ] RadialPuppet parameters in `subParameters`.
- [ ] **All outfit containers inactive in scene; default suite enabled via menu `isDefault`** (do not hard-code activeSelf in edit mode).
- [ ] **New outfits with own armature: add OutfitRoot + MergeArmature + MeshSettings**.
- [ ] **Per-suite menu stays under 8 controls** to avoid auto `More` overflow; part toggles flat.
- [ ] `ProcessAvatarUI` build validated; no broken bindings; Console clean.
