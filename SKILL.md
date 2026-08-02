---
name: modular-avatar
description: Safely inspect, configure, automate, and validate Modular Avatar in Unity/VRChat projects. Use for outfits, armature merging, accessories, menus, parameters, animator merging, reactive components, blendshape synchronization, mesh clipping, materials, NDMF previews, and build troubleshooting.
license: MIT
compatibility: OpenCode v2; Unity projects with Modular Avatar installed; optimized for VRChat avatar projects and Unity MCP
metadata:
  opencode/slash: "true"
  opencode/autoinvoke: "true"
  version: "1.0.0"
  captured: "2026-08-02"
---

# Modular Avatar agent skill

Use this skill as a **workflow navigator**, not as a frozen copy of Modular Avatar documentation.

## Non-negotiable source order

1. The current Unity project's `Packages/manifest.json` and `Packages/packages-lock.json`.
2. The exact installed package's `package.json`, C# source, bundled docs, samples, and changelog.
3. Official Modular Avatar documentation and official GitHub releases.
4. Model memory only for forming search terms, never for exact fields or API calls.

Read `references/source-policy.md` before relying on documentation.

## Native tool policy

- Prefer OpenCode's native `unityMCP_*` tools for live Unity inspection and changes.
- Do not write Python, JavaScript, PowerShell, curl, or raw JSON-RPC to call an MCP endpoint.
- Shell scripts in this skill are only for **local package inspection**, not for controlling Unity or bypassing MCP permissions.
- If `unityMCP_*` tools are unavailable, stop live-editor work and report that the tools were not injected.
- Reading project files is allowed when it is safer or more exact than live UI inspection.
- Never directly edit `.unity`, `.prefab`, `.asset`, or `.meta` YAML unless the user explicitly requests it and the change is reviewed first.

## Start every task with discovery

1. Locate the Unity project root by finding `Assets/`, `Packages/`, and `ProjectSettings/`.
2. Run one of:
   - Windows: `scripts/inspect-modular-avatar.ps1`
   - Cross-platform: `scripts/inspect_modular_avatar.py`
3. Record:
   - Unity editor version;
   - Modular Avatar package ID and resolved version;
   - package source type and source directory;
   - NDMF version if available;
   - whether the package is stable, beta, alpha, Git, local, or embedded.
4. If Unity is open, use native Unity MCP tools to identify:
   - connected Unity instance;
   - active scene;
   - avatar roots containing `VRCAvatarDescriptor`;
   - prefab status of the target;
   - existing MA, NDMF, VRCFury, and relevant VRCSDK components;
   - current Console errors and warnings.
5. Report the proposed change and its scope before a write operation.

## Resolve types and fields; never guess

Before adding or modifying a Modular Avatar component:

1. Search the installed package source for the component display name and candidate type.
2. Inspect the actual component class and custom editor/inspector code.
3. Prefer Unity serialization/reflection information exposed by Unity MCP.
4. Confirm object-reference semantics: direct reference, path reference, humanoid bone reference, GUID, or generated asset.
5. Confirm version-specific options before using them.

Useful local searches:

```text
rg -n "class ModularAvatar|AddComponentMenu|CustomEditor|SerializedProperty" <package-source>
rg -n "MergeArmature|BoneProxy|MenuItem|Parameters|BlendshapeSync|MeshCutter" <package-source>
```

Use `scripts/find-ma-symbol.ps1` or `scripts/find_ma_symbol.py` when convenient.

## Select the least destructive representation

Use this preference order:

1. A separate module GameObject or prefab under the avatar.
2. Modular Avatar declarative components.
3. New copied materials, clips, menus, parameters, or controllers under a custom asset folder.
4. An Editor script with Undo support and explicit validation.
5. Direct modification of original avatar assets only when no safer representation exists and the user approves.

Do not edit files under `Library/PackageCache/` or immutable package sources. Do not modify imported BOOTH content in place.

## Component selection

Read `references/component-guide.md` for details.

- Skinned outfit armature integration: **Merge Armature**.
- Rigid accessory attached to an avatar bone: **Bone Proxy**.
- Hierarchy-authored expression control: **Menu Item**, normally bound by **Menu Installer** or parent menu structure.
- Custom animator/expression parameters: **Parameters**.
- Merge a controller into a playable layer: **Merge Animator**.
- Keep corresponding outfit/body blendshapes aligned: **Blendshape Sync**.
- Change body blendshapes when a module is active: **Shape Changer**.
- Hide/delete selected body polygons: **Mesh Cutter** plus one or more vertex filters.
- Swap or set materials reactively: **Material Swap** or **Material Setter**, after verifying installed support.
- Common mesh bounds/anchor configuration: **Mesh Settings**.

Do not use Bone Proxy as a substitute for fitting skinned clothing. Do not use Mesh Cutter to hide an entire mesh when an object toggle is more efficient.

## Standard execution loop

For every modification:

1. **Observe**: query current objects, components, paths, asset references, and Console state.
2. **Plan**: name the exact objects/assets to create or modify and state rollback strategy.
3. **Approve**: request approval for destructive or externally consequential actions.
4. **Apply minimally**: perform one coherent change through native Unity MCP or a reviewed Editor script.
5. **Re-read**: inspect the resulting component and references rather than assuming success.
6. **Preview**: use MA/NDMF preview, play mode, or build validation appropriate to the change.
7. **Check Console**: compare new errors/warnings against the pre-change baseline.
8. **Report**: list changed GameObjects, files, components, parameters, and unresolved warnings.

Read `references/workflows.md` for task-specific sequences and `references/safety-validation.md` for approval boundaries. Before adding MA components or rebuilding menus, read `references/gotchas.md` for known pitfalls (ObjectToggle reference paths, MenuItem binding, `_OFF` parameter polarity, reparenting breaking FX paths, MenuInstaller binding, stray-component cleanup, and build validation). When the task is converting an existing model into MA-driven controls (regrouping nodes, rebuilding menus, moving toggles to MA, fixing FX paths), follow `references/ma-ify-model.md`.

## VRChat-specific constraints

- Do not upgrade Unity, VRCSDK, NDMF, Modular Avatar, shaders, or other packages unless explicitly requested.
- Keep original avatar FX controllers, menus, and parameter assets untouched when MA can merge a module at build time.
- Detect duplicate parameter names, type mismatches, defaults, saved/synced flags, and parameter-budget impact.
- Validate armature mapping, humanoid bone targets, scale, root transforms, and prefab overrides.
- Check desktop and Android/per-platform overrides separately when present.
- Do not publish or upload an avatar automatically.

## Documentation use

When exact behavior is unclear:

1. Search bundled package docs/samples first.
2. Read only the relevant official web page listed in `references/official-sources.md`.
3. Compare the page's feature/version with the installed package.
4. State when documentation describes a newer version than the project.
5. Never use unofficial tutorials as the sole authority for serialized fields or build semantics.

## Output format after inspection

Report:

```text
Project:
Unity version:
Avatar target:
Modular Avatar package ID/version/source:
NDMF version:
Unity MCP status:
Relevant existing components:
Console baseline:
Proposed change:
Files/GameObjects affected:
Rollback method:
Approval required:
```

## Output format after modification

Report:

```text
Completed change:
GameObjects changed:
Assets created/changed:
MA components added/changed:
Parameters/menu items affected:
Validation performed:
New Console errors/warnings:
Remaining manual checks:
Saved scene/prefab: yes/no
Built/uploaded: no unless explicitly approved
```
