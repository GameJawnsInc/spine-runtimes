# Render Separation for spine-godot

A Godot equivalent of spine-unity's [`SkeletonRenderSeparator`](https://esotericsoftware.com/spine-unity-utility-components#SkeletonRenderSeparator). Lets an unrelated `SpineSprite` (or any Node2D) be drawn *between* the front and back halves of another skeleton's draw order, without duplicating the skeleton.

Originally proposed in [issue #2689](https://github.com/EsotericSoftware/spine-runtimes/issues/2689); see [PR #3091](https://github.com/EsotericSoftware/spine-runtimes/pull/3091) for the design discussion. The PR was declined upstream — Mario Zechner's position is that `SpineSlotNode` already covers this. This branch (`godot-render-separation` on `GameJawnsInc/spine-runtimes`) keeps the feature alive as a maintained fork. See [Why not just use SpineSlotNode?](#why-not-just-use-spineslotnode) for the counter-argument and the case `SpineSlotNode` can't express.

> The repo-root `CLAUDE.md` contains the *original* implementation plan; the code diverged from it as the work progressed. Treat this README and the source files as authoritative; treat `CLAUDE.md` as historical.

## What this adds

- **`SpineSlotRangeProxy`** (`Node2D`): claims a contiguous slot range from a source `SpineSprite`, hides the source's own meshes for those slots, mirrors the source's per-slot geometry/material each frame on its own canvas item, and optionally follows the source's global transform. Lives anywhere in the scene tree.
- **`SpineSpriteRenderSeparator`** (`Node`): convenience — auto-partitions a skeleton into N+1 proxies given a list of separator slot names. The orchestrator analogue of Unity's `SkeletonRenderSeparator`.
- **`SpineSprite` extensions**: a proxy registration API + a visibility skip in `update_meshes` so slots claimed by a proxy don't double-render at the source.

## File map

| File | Status | Role |
|------|--------|------|
| `spine_godot/SpineSprite.h` | modified | forward-decl `SpineSlotRangeProxy`; `friend class SpineSlotRangeProxy` inside `SpineMesh2D`; `struct SpineRenderProxyBinding`; `Vector<SpineRenderProxyBinding> render_proxies`; 5 public proxy methods (`_register_proxy`, `_unregister_proxy`, `is_slot_externally_rendered`, `collect_slot_range_meshes`, `get_draw_order_count`). |
| `spine_godot/SpineSprite.cpp` | modified | implements the proxy registration methods; the critical hook in `update_meshes`: `mesh_instance->set_visible(!is_slot_externally_rendered((int) slot->getData().getIndex()))`. `_register_proxy` warns on overlapping ranges. |
| `spine_godot/SpineSlotRangeProxy.{h,cpp}` | new | the proxy class. Mirrors `SpineSlotNode`'s NodePath-resolution + slot-name property pattern. |
| `spine_godot/SpineSpriteRenderSeparator.{h,cpp}` | new | the convenience separator. `rebuild_proxies()` does the partition + auto-create. |
| `spine_godot/register_types.cpp` | modified | `GDREGISTER_CLASS(SpineSlotRangeProxy);` and `GDREGISTER_CLASS(SpineSpriteRenderSeparator);` next to the existing `GDREGISTER_CLASS(SpineSlotNode);`. |
| `SConstruct` | modified | GDExtension uses an **explicit** source list (unlike the module's wildcard `SCsub`). Adds: `sources.append("spine_godot/SpineSlotRangeProxy.cpp")` and `sources.append("spine_godot/SpineSpriteRenderSeparator.cpp")`. Forgetting this is how the GDExtension build initially broke with 9 unresolved externals. |
| `example-v4/examples/14-render-separator/` | new | demo scene: two raptor+spineboy pairs — pair 1 bone-attached, pair 2 flat siblings. The `.gd` has an inline comment block explaining the layering. |

Nothing under `spine_godot/spine-cpp/` is touched — that's the cross-runtime core, shared with every other runtime, and must stay off-limits.

## Architecture

`SpineSprite` owns one `SpineMesh2D` child per skeleton slot. Each frame:

1. `update_meshes` rewrites geometry into each `SpineMesh2D` based on its slot's current attachment.
2. `sort_slot_nodes` walks `skeleton->getDrawOrder()` and calls `move_child(...)` so Godot child order matches Spine draw order.

The proxy bolts in at two points:

- **At registration**, the source records the proxy's `[start_idx, end_idx]` range in `render_proxies`.
- **In `update_meshes`**, the source still computes geometry for every slot (the proxy reads it), but calls `set_visible(false)` on its own mesh for any slot in any registered range. Net effect: the source's canvas item simply doesn't draw those slots.

The proxy itself:

- Resolves `source_sprite` (NodePath) and `start_slot_name` / `end_slot_name` to indices.
- Owns its own `SpineMesh2D` children — one per slot in the claimed range.
- Each frame, calls `collect_slot_range_meshes` on the source to pull the freshly-computed geometry + materials, then updates its own child meshes.
- Optionally copies the source's `global_transform` (default: on).

### Why mirror instead of share mesh RIDs?

The original plan (see `CLAUDE.md`) was to share mesh RIDs across canvas items via `RenderingServer::canvas_item_add_mesh`. The actual implementation has the proxy own its own `SpineMesh2D` children that mirror geometry/material each frame. Reason: the per-slot blend-mode material plumbing in Godot is most easily reused by going through `SpineMesh2D` rather than partitioning RIDs into contiguous blend-mode runs. The shared-RID approach is cheaper per frame but more work to keep correct across attachment changes (skin swaps, blend-mode flips, texture swaps).

### The `SpineBoneNode` draw-order gotcha

`SpineSprite::sort_slot_nodes()` only orders the skeleton's slot meshes and `SpineSlotNode` children. A `SpineBoneNode` child's draw index is **unmanaged** — bones have no draw-order position; only slots do. So when the source `SpineSprite` is nested under a `SpineBoneNode` of another skeleton (the "bone-attached" pair in the example), its draw order relative to its host's slots is unstable.

The proxy's transform / registration / geometry mirroring still work — only the layering is unstable. **The fix is `z_index`**: in the example, the bone-attached source has `z_index = -1`, so its back-half slots reliably draw behind the host skeleton, while the proxy stays at default `z_index = 0` and draws in front. Pair 2 of the example uses the same `z_index = -1` trick for symmetry even though it's a flat sibling, so both setups look identical.

If you reproduce a similar setup elsewhere and the layering looks wrong, this is almost certainly the cause; `z_index` is the lever.

## Build

All from **Git Bash** (`.sh` scripts won't run in cmd or PowerShell). Godot is cloned at `spine-godot/godot/`.

```sh
# scons needs to be on PATH. On the original dev machine it lived at:
#   /c/Users/skaki/AppData/Local/Python/pythoncore-3.14-64/Scripts
# Adjust to wherever scons.exe is installed on your machine.
export PATH="/c/Users/skaki/AppData/Local/Python/pythoncore-3.14-64/Scripts:$PATH"

cd spine-godot

# --- Module build (recommended for dev — fastest iteration) ---
./build/build-v4.sh
# Editor binary lands at: spine-godot/godot/bin/godot.windows.editor.x86_64.exe

# --- GDExtension build (3 targets: editor, template-debug, template-release) ---
./build/setup-extension.sh      # one-time setup
./build/build-extension.sh
```

### Build gotcha — shared object files clobber each other

The module build and GDExtension build both compile sources from `spine_godot/` and **write `.obj` files in place** alongside the `.cpp`. They use different compile flags (the GDExtension defines `SPINE_GODOT_EXTENSION`), so an object built for one build will mis-link in the other. **When switching build types, delete the shared object files first:**

```sh
find spine-godot/spine_godot -name '*.obj' -delete
```

Symptom of forgetting: link errors referencing `godot::`-namespaced symbols on a module build (or the inverse for GDExtension). This was a real failure during initial development; budget for it.

### GDExtension `SConstruct` reminder

`spine_godot/SCsub` (used by the module build) globs `*.cpp` — new files compile automatically.
`spine-godot/SConstruct` (used by the GDExtension build) uses an **explicit source list** — new files must be added or they'll link-fail with unresolved externals. Currently:

```python
sources.append("spine_godot/SpineSlotRangeProxy.cpp")
sources.append("spine_godot/SpineSpriteRenderSeparator.cpp")
```

If you add a new `.cpp` for this feature, add it here too.

## Test plan after rebase

Run these in order before pushing a rebased version of the branch. Scenes must run **windowed** (not `--headless` — headless gives blank screenshots).

1. **Module builds clean.**
2. **GDExtension builds clean** (all 3 targets).
3. **`07-slot-node` regression** — the existing `SpineSlotNode` example must render unchanged. This catches accidental breakage of `sort_slot_nodes` or the slot/draw-order plumbing.
4. **`14-render-separator` pair 1** (bone-attached, left) — raptor's body draws between spineboy's back arm and front body, while spineboy rides the raptor. Both walk cycles play. This is the case `SpineSlotNode` alone *cannot* express.
5. **`14-render-separator` pair 2** (flat siblings, right) — same visual effect, simpler setup.
6. **Multi-proxy via `SpineSpriteRenderSeparator`** — partition spineboy into N+1 slot ranges (e.g., split at `front-upper-arm` → 25/27 slot coverage) and verify each range renders at its proxy.

If you need to capture a screenshot from within a scene for evidence, the pattern is:

```gdscript
await RenderingServer.frame_post_draw
get_viewport().get_texture().get_image().save_png("user://check.png")
```

(Headless mode produces blanks — always run windowed.)

## Rebasing onto a new upstream

The branch is currently based on `EsotericSoftware/spine-runtimes:4.3`. To track a new version (4.4, 4.5…):

```sh
git fetch upstream
git rebase upstream/4.X
# resolve conflicts (see "Likely friction points" below)
git push --force-with-lease fork godot-render-separation
```

The closed PR #3091 will update with the new commit list automatically — useful as an evolving public "current state of the patch" view. Don't delete the branch on the fork; the PR closure doesn't lock you out of anything, but a deleted branch would.

**Frequent small rebases are vastly less painful than rare big ones.** Drift compounds; a year of upstream churn turns a 10-minute rebase into a multi-hour archaeology session.

### Likely friction points

If the rebase hits conflicts in any of these, double-check carefully:

- **`SpineSprite.h`** — the `friend class SpineSlotRangeProxy` declaration inside `SpineMesh2D`, the forward decl, the `SpineRenderProxyBinding` struct, the `render_proxies` member, and the 5 public proxy methods are all insertions. If Esoteric refactors `SpineMesh2D`'s class body or `SpineSprite`'s member layout, the insertions may need to move; the `friend` line in particular must stay inside the new `SpineMesh2D` body wherever that ends up.
- **`SpineSprite.cpp::update_meshes`** — the `set_visible(!is_slot_externally_rendered(...))` line is *the* critical hook. If the per-slot mesh update loop is rewritten upstream, this needs to land in the new equivalent loop, in the same place relative to where the slot's mesh visibility is set.
- **`SpineSprite::sort_slot_nodes`** — no direct change in this patch, but it's load-bearing for everything the proxy does. If upstream changes how draw order is propagated to Godot child order, re-verify both `07-slot-node` and the bone-attached pair end-to-end.
- **`register_types.cpp`** — the two `GDREGISTER_CLASS(...)` calls are simple but the file shifts as upstream adds classes; place them next to `GDREGISTER_CLASS(SpineSlotNode);` wherever that ends up.
- **`SConstruct`** — if Esoteric switches the GDExtension build to a wildcard source list (as the module's `SCsub` already does), the two `sources.append(...)` lines become unnecessary and can be dropped. If they keep the explicit list, any new sources in this patch must be added there.

### What's unlikely to need changes

The new files (`SpineSlotRangeProxy.{h,cpp}`, `SpineSpriteRenderSeparator.{h,cpp}`) are self-contained and only depend on `SpineSprite`'s public surface plus Godot's `Node2D` / canvas-item plumbing. They should survive most upstream churn intact unless `SpineSprite`'s `collect_slot_range_meshes` / `is_slot_externally_rendered` API surface changes — which it won't unless this patch itself changes them.

## Why not just use SpineSlotNode?

The upstream maintainer's position is that `SpineSlotNode` already covers this — parent the foreign node under a `SpineSlotNode` of the source, and it renders at that slot's draw position. This is **true for the simple case** ("draw character B between character A's halves" with B parented under `A/SlotNode/`). The `07-slot-node` example demonstrates exactly that pattern.

The case it doesn't cover is the bone-attached pair in the demo scene: spineboy is parented under a `SpineBoneNode` on raptor's `head` bone — spineboy rides the raptor — and the goal is for the raptor's body to draw between spineboy's back arm and front body. To get that render order with `SpineSlotNode` alone, the raptor would have to be a child of a `SpineSlotNode` on the spineboy — which directly contradicts "spineboy rides raptor"; the scene tree can't hold both relationships at once.

The proxy decouples rendering position from scene-tree parenting: the source can sit anywhere in the tree, and its slots can be drawn at a different node with their own transform / `z_index` / material. `SpineSlotNode` binds rendering position to tree parenting; `SpineSlotRangeProxy` doesn't.

There's also the multi-range case — `SpineSpriteRenderSeparator` splits one skeleton into N+1 separately rendered chunks at arbitrary scene-tree locations — which `SpineSlotNode` insertion doesn't generalize to.

The closed PR #3091 has the full back-and-forth if you want the source material.

## Open questions (from the original PR — moot now, but worth remembering)

These were the four maintainer-facing questions in the original PR description. If you ever re-propose this (under a different design, or to a different runtime), they're the real design surface to nail down:

1. **Naming.** Mirror Unity 1:1 (`SpineSkeletonRenderSeparator` + `SpineSkeletonPartsRenderer`) or use the pair I shipped (`SpineSlotRangeProxy` + `SpineSpriteRenderSeparator`)?
2. **Material override.** Should `SpineSlotRangeProxy` let you override the per-slot material, the way Unity's `SkeletonPartsRenderer` does?
3. **`SpineSlotNode` children inside a proxy range.** If a user places a `SpineSlotNode` bound to a slot that's also inside a proxy's range, where does it render — under the source or under the proxy? Current behavior: under the proxy (the proxy "owns" that slot's draw). Worth documenting per-direction in API docs.
4. **Overlapping proxy ranges.** Current behavior: warn via `WARN_PRINT` and every claiming proxy renders the slot (so it draws twice). Alternative would be "first-registered wins" or "last-registered wins." Pick one and document.

## References

- Upstream issue: <https://github.com/EsotericSoftware/spine-runtimes/issues/2689>
- Upstream PR (closed): <https://github.com/EsotericSoftware/spine-runtimes/pull/3091>
- Forum thread: <https://esotericsoftware.com/forum/d/27253-godot-43-slot-draw-order-manipulation-at-runtime>
- spine-unity reference: <https://esotericsoftware.com/spine-unity-utility-components#SkeletonRenderSeparator>
