# Evidence status (honest ledger)

The other skills in this repository follow one rule: every technique was reproduced and
measured in a real Godot 4.7.2 scene before being written down. **This skill is not
there yet.** Its current evidence tier:

- **Verified upstream (web/JS)**: the ten patterns, their severity ordering and their
  before/after numbers come from andrew_game_perf `d0976ee` (MIT), where they were
  measured on web engines (React/Three.js/PixiJS/Phaser). Those numbers are quoted as
  motivation, not as Godot results.
- **Verified against documentation (Godot 4.7)**: the API-level claims used by the
  mappings — `Performance.get_monitor` (+ render monitors), `PackedVector3Array.resize`
  being faster than per-element appends, MultiMesh single-draw instancing, fixed-step
  `_physics_process` + physics interpolation and
  `Engine.get_physics_interpolation_fraction`. Checked against the Godot 4.7 stable
  documentation on 2026-11-17.
- **NOT yet measured in engine (Godot)**: the per-pattern cost deltas (e.g. "Dictionary
  lookup vs `.find()` at n=1000 in GDScript", "MultiMesh vs 1000 MeshInstance3D draw
  calls on this hardware", "pooled vs spawn/destroy churn"). Treat the Godot-specific
  performance claims as strong engineering guidance pending measurement.

## Measurement plan (to reach the repo's evidence bar)

1. Micro-bench pattern 1: `Array.find` vs `Dictionary[id]` vs space-state query at
   n = 100 / 1000 / 5000, in `_physics_process`, 300-frame averages (reuse the
   discipline of `godot-render-performance`: fixed conditions, averaged, repeated).
2. Pattern 8: same scene of N identical meshes as MeshInstance3D vs MultiMeshInstance3D;
   record `Performance.RENDER_TOTAL_DRAW_CALLS_IN_FRAME` and frame time.
3. Pattern 9: pool vs instantiate/queue_free at 50 and 200 spawns/s; watch
   `OBJECT_COUNT` and `TIME_PROCESS`.
4. Pattern 10: tunneling test — fast body through a thin wall in `_process` vs
   `_physics_process` at capped 30 FPS.
5. Record results here with numbers and conditions; until then this file stays the
   disclaimer.
