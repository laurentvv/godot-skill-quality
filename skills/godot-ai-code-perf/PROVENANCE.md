# Provenance

- **Upstream source**: https://github.com/gykim80/andrew_game_perf — MIT License,
  Copyright (c) 2026 Andrew Kim.
- **Pinned commit**: `d0976ee` (cloned read-only to `C:/Git/andrew-game-perf` on
  2026-11-17; extraction done from that clone, not from memory).
- **What was taken**: the ten FPS-drop patterns — names, severity levels, failure
  mechanisms, detection heuristics and fix shapes — from
  `skills/game-perf/SKILL.md`, `skills/game-perf/references/pattern-1..10-*.md` and
  `README.md` (English). Upstream's measured numbers are quoted with their web/JS
  context explicitly attached; they are motivating evidence, not Godot measurements.
- **What was written new**: every Godot 4 mapping (GDScript shapes, node/API choices,
  detection greps, fixes). API facts were checked against the Godot 4.7 documentation
  (`get_monitor`, `PackedVector3Array.resize` semantics, MultiMesh instancing, physics
  interpolation / `Engine.get_physics_interpolation_fraction`). No in-engine
  measurements have been run for the Godot mappings yet — see
  `references/evidence-status.md`.
- **License**: this skill folder is MIT (see `LICENSE`); attribution to the upstream
  repository is required when redistributing derived versions, per upstream MIT.
- **Extraction date**: 2026-11-17.
