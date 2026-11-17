---
name: godot-ai-code-perf
description: >-
  Find the ten FPS-drop patterns that AI-generated game code falls into, translated from the
  andrew_game_perf web-game checklist to Godot 4 (GDScript): quadratic lookups in _process,
  per-frame UI mutations, per-event scene-tree churn, fresh containers in hot paths, node-per-item
  lists, per-frame allocations, leaked references to GPU resources, unbatched draw calls,
  create/destroy entity churn without pooling, and physics run in _process with a variable
  timestep. Use this whenever generated GDScript feels stuttery, when FPS drops scale with entity
  count, when the profiler shows _process cost or growing memory, or when reviewing AI-written
  gameplay code for performance before shipping.
license: MIT (see LICENSE)
---

# AI-generated code perf patterns: the Godot 4 translation

Source checklist: [gykim80/andrew_game_perf](https://github.com/gykim80/andrew_game_perf)
(MIT, pinned `d0976ee`, extracted 2026-11-17) documents 10 recurring FPS-killers in
AI-written web game code, with detection heuristics and measured before/after numbers
(upstream numbers quoted below are from that repo's web/JS context — the *mechanisms*
transfer; the Godot mappings here were written against Godot 4.7.2 documentation and are
**not yet re-measured in engine**, see `references/evidence-status.md`).

Work order (upstream's, kept): fix Pattern 4 first (it can hang the game), then 1, 2, 3, 5.
In a Godot codebase the usual suspects are 1, 6, 8, 9 in `_process`/`_physics_process`
and 10 in the wrong callback.

## Pattern 1 — O(n²) lookup in hot loops (CRITICAL, upstream ~10 ms/frame → ~0.5 ms)

**Web shape**: `.find()` inside a per-frame loop. **Godot shape**:

- `for` over entities calling `array.find()`/`.has()`/`.filter()` inside `_process` or
  `_physics_process` — quadratic at 60 Hz.
- Node lookups per frame: `get_node("...")`, `find_children(...)`, or
  `get_tree().get_nodes_in_group(...)` scanned every tick to find one node.
- Proximity or pair tests done as double loop instead of using the physics broadphase.

**Fix**:

- Build a `Dictionary` keyed by id once; O(1) `dict[id]`. Rebuild only when membership
  changes, not per frame.
- Cache node references: `@onready var mesh := $Mesh` beats a path lookup per frame.
  For dynamic nodes, resolve once in `_ready`/on spawn and store the reference.
- For "what is near me", use the space state (broadphase octree) instead of pair scans:
  `get_world_3d().direct_space_state.intersect_shape(...)` (or `intersect_point` /
  `intersect_ray`) — the engine already has the spatial index.

**Detection**: grep `_process`/`_physics_process` bodies for `.find(`, `.filter(`,
`get_nodes_in_group(`, `find_children(`. Exclude one-time `_ready` code.

## Pattern 2 — per-frame values driving UI state (CRITICAL, upstream 60 re-renders/s → 0)

**Web shape**: frame-varying value in React state. **Godot shape**:

- Setting `Label.text`, ProgressBar values or theme overrides every `_process` tick —
  each text assignment re-lays-out and re-draws a Control subtree.
- `queue_redraw()` called every frame for a `CanvasItem` whose content rarely changes.
- `set_process(true)` left on for nodes that only need to react to events.

**Fix**:

- Drive HUD updates from *change*, not from the frame: update when the watched value
  crosses a threshold or on a signal, not on every tick.
- For continuously-changing readouts (speed, coords), throttle to a visible rate
  (a 10 Hz `Timer` is plenty) and keep formatted strings cached between updates.
- Turn processing off when idle: `set_process(false)` until there is something to do.

## Pattern 3 — scene-tree mutations in input/drag handlers (WARNING, upstream 120 renders/s → 1)

**Web shape**: `setState` in `onMouseMove`. **Godot shape**:

- Mouse-motion events arrive at display rate (120 Hz+); doing structural work per event —
  `add_child`, `remove_child`, `reparent`, instantiating preview nodes — turns one drag
  into hundreds of tree mutations.
- Emitting signals with fresh payloads per motion event, or calling `queue_redraw()`
  per motion for changes that only matter on release.

**Fix**:

- During the drag, mutate only cheap properties (`position`, `modulate`); defer structural
  changes to the release (`NOTIFICATION_DRAG_END`, button-up in `_unhandled_input`).
- For Control drags use the built-in `_get_drag_data` / `_notification` flow instead of
  hand-rolled per-motion logic.
- Distinguish click from drag with a small movement threshold before doing any work.

## Pattern 4 — fresh containers in equality-checked hot paths (CRITICAL, upstream: tab crash → stable)

**Web shape**: selector returning a new object → infinite loop. Godot has no render-loop
equivalence check, but the *allocation* half is a real killer and one variant does loop:

- Building a new `Array`/`Dictionary` literal per frame to pass into a method or signal
  (`spawn({"pos": p, "vel": v})` at 60 Hz), or calling `.duplicate()` on a Resource per
  frame.
- Recursive deferred self-calls: `call_deferred("tick").call_deferred(...)` chains, or a
  signal handler that emits the same signal — the Godot analogue of the runaway loop
  (each hop defers, so the game "runs" while it melts).

**Fix**:

- Pass primitives or reuse a preallocated container for per-frame payloads.
- Never `duplicate()` resources in a loop; cache one working copy.
- A signal handler that can re-trigger itself needs a guard flag or `CONNECT_ONE_SHOT`.

## Pattern 5 — node-per-item lists without windowing (WARNING, upstream 25 fps → 60 fps scroll)

**Web shape**: 500 DOM nodes via `.map()`. **Godot shape**:

- Hundreds/thousands of `Control` nodes (one per inventory entry, log line, leaderboard
  row) inside a container — each is a laid-out, drawn `CanvasItem`.
- Long debug logs implemented as one Label per line.

**Fix**:

- Use the built-ins that draw items themselves instead of one node per item:
  `ItemList`, `Tree`, `RichTextLabel`.
- For custom huge lists, window: create Controls only for the visible slice, recycle on
  scroll (same idea as upstream's virtualization, ~10–20 live rows).
- For non-interactive 2D content (minimap blips, damage numbers), draw in a single
  `CanvasItem._draw()` (`draw_texture_rect`, `draw_string`) instead of node-per-sprite.

## Pattern 6 — allocation in the hot loop (CRITICAL, upstream GC 15 ms → 0)

**Web shape**: `new Vector3` in the render loop. GDScript's `Vector3` is stack-cheap, but
the same *garbage* arrives differently:

- Per-frame `String` building: `str()`, `"%s" %`, `+=` concatenation in `_process`.
- Per-frame container literals and `get_children()` (allocates a fresh `Array` each call)
  inside `_process`.
- Untyped `Array` of Variants where a packed, typed array would do; growing arrays with
  `append` in a hot loop instead of sizing once (`PackedVector3Array.resize()` once then
  assign beats appending one by one — documented in Godot 4.7).
- Creating `Callable` lambdas per frame (captures).

**Fix**:

- Cache formatted strings; update them at Pattern-2 rates, not per frame.
- Cache `get_children()` results (or iterate a maintained list); keep one reusable
  member `Array`/`Dictionary` for per-frame bookkeeping.
- Prefer typed/packed arrays (`Array[Node3D]`, `PackedVector3Array`) for hot data.

## Pattern 7 — GPU-sized resources created per frame, references never dropped (CRITICAL, upstream 320 MB leak → 0)

Godot Resources are reference-counted and free themselves — the leaks are **cycles and
long-lived references**, and per-frame *creation*:

- `ImageTexture.create_from_image(...)` / `ShaderMaterial.new()` / `shader_code`
  assignment per frame (dynamic minimaps, per-frame procedural textures) — each new
  texture/material holds VRAM until released.
- Holding freed-object references: signals still connected to a freed node's method, or
  containers of nodes freed with `queue_free()` but never removed from the container.
- Reference cycles (`a.b = b; b.a = a`, including Object-to-Object via Dictionary caches)
  that ref-counting cannot collect.

**Fix**:

- Create textures/materials/shaders once; update *contents* (`texture.update(image)`)
  instead of re-creating resources.
- Disconnect signals you will not use (`disconnect`, `CONNECT_ONE_SHOT`), clear containers
  on `NOTIFICATION_PREDELETE`/`_exit_tree`, `is_instance_valid()` before use.
- Watch drift with monitors: `Performance.get_monitor(Performance.OBJECT_COUNT)` /
  `MEMORY_STATIC` — a monotonic climb across minutes is the smoking gun.

## Pattern 8 — unbatched draw calls (WARNING, upstream 30 ms CPU → 0.05 ms)

**Web shape**: one draw per sprite. **Godot shape**: one `MeshInstance3D` / unique-texture
`Sprite2D` per object — thousands of items each submitting their own draw.

**Fix** (Godot's native batching tools):

- `MultiMeshInstance3D` / `MultiMeshInstance2D`: one draw call for thousands of instances
  of one mesh (`transform`/`color`/`custom` per instance) — the documented way to render
  "thousands of fish".
- Texture atlas: pack sprites into one texture so 2D batching groups them (same texture,
  adjacent items); avoid interleaving different textures in draw order.
- Share materials: per-object `ShaderMaterial` duplicates defeat batching; use shared
  material + per-instance data.
- Ephemeral swarms (bullets, sparks): `GPUParticles`/`CPUParticles` rather than
  node-per-projectile.

**Detection**: `Performance.get_monitor(Performance.RENDER_TOTAL_DRAW_CALLS_IN_FRAME)`
(the same `get_monitor` API as `TIME_FPS`) — snapshot before/after batching.

## Pattern 9 — create/destroy churn in the update loop (WARNING, upstream GC every 1–2 s → 0)

**Web shape**: `new Sprite()` per bullet. **Godot shape**: `instantiate()` + `add_child()`
per shot and `queue_free()` on expiry — allocation, tree churn and physics registration,
per entity, per frame.

**Fix**:

- Pool: `preload()` the `PackedScene`, instantiate a fixed bank up front, recycle with
  `visible = false` + `process_mode = PROCESS_MODE_DISABLED` (and pause monitoring,
  collisions) instead of create/destroy.
- Manage the active list with swap-remove (copy the last element into the hole, then
  `pop_back()`) — O(1) removal, never `remove_at(i)` in a forward loop.
- For pure visuals, prefer particles or a `MultiMesh` over pooled nodes (Pattern 8).

**Detection**: `_process` bodies containing `instantiate(`; bullet counters that track
spawn rate × lifetime — upstream's math (50 spawns/s = steady-state garbage flow) applies
directly.

## Pattern 10 — physics on a variable timestep (WARNING, upstream: tunneling → deterministic)

**Web shape**: `position += velocity * delta` in a variable-rate loop. **Godot shape**:
generated code putting movement/integration in `_process(delta)` (render-rate, variable
delta) — including direct `position +=` writes — or ticking physics through timers.

**Why it bites in Godot**: collision tunneling at low FPS, spring/damper instability on
delta spikes, desync between clients, and teleporting after an alt-tab stall.

**Fix**:

- Movement and any force integration belong in `_physics_process` — Godot already runs a
  fixed timestep (`Engine.physics_ticks_per_second`, default 60).
- The death spiral is already guarded (`Engine.max_physics_steps_per_frame` caps catch-up;
  the game slows down instead of locking up) — do not "fix" it by multiplying raw
  `_process` delta into physics.
- For smooth rendering of fixed-step motion, enable physics interpolation (Godot 4.7's
  "fixed timestep interpolation" — `physics_interpolation` on nodes / project setting) or
  interpolate manually with `Engine.get_physics_interpolation_fraction()`.
- Test: focus another window for 5 s and return — nothing may teleport.

## Review pass checklist (all ten, one sweep)

1. `_process` vs `_physics_process` usage correct? (P10)
2. Any `.find/.filter/get_nodes_in_group/find_children` inside a per-frame body? (P1)
3. UI/text updates driven by change, not by tick? (P2)
4. Input/drag handlers free of per-event tree mutations? (P3)
5. Any per-frame `Array`/`Dict` literals, `.duplicate()`, or self-retriggering deferred
   chains? (P4)
6. Node-per-item where ItemList/Tree/`_draw` would do? (P5)
7. Per-frame strings, `get_children()`, untyped arrays in hot loops? (P6)
8. Textures/materials created per frame; stale refs; cycles? (P7)
9. Draw-call count measured; MultiMesh/atlas/particles where it counts? (P8)
10. Pools + swap-remove for spawned entities? (P9)

Measure before and after each fix (see `godot-render-performance` for the GPU-side bench
discipline; `Performance` monitors + the engine profiler cover the CPU/object side).
