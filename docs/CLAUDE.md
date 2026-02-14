# docs/ — AudioHax Documentation

## Document Inventory

ROADMAP.md: Phased development plan. 5 phases from music quality through installation deployment. Each phase has tasks with complexity, ownership, and dependencies. Update when phases complete or priorities shift.

interactive-architecture.md: Architectural plan for the interactive experience layers (mouse painting, camera tracking, game integration, art installation). Covers the reactive pipeline refactor, new module designs, and crate recommendations.

phase1-agent-team.md: Agent team configuration for Phase 1 (music quality). Contains spawn prompts, task definitions, and execution sequence.

modem-agent-team.md: Agent team configuration for modem hardening. Contains spawn prompts, task definitions, and execution sequence.

## Conventions

Write in clear prose, not bullet-point lists. Use proper music theory terminology when discussing musical features — the project owner has a performance degree. Technical terms should be precise (name specific Rust types and function signatures when referencing code). Keep documents actionable — every section should help someone do work, not just describe the system.

## When to Update

Update ROADMAP.md when completing a phase or reprioritizing. Add a new agent-team doc for each new swarm configuration. Architecture docs should be updated when new modules are added or interfaces change. If a doc references specific line numbers in source files, verify them after refactors.

## Relationship to Project Knowledge

The Claude Project knowledge files (audiohax-architecture.md, development-conventions.md, music-theory-context.md, agent-teams-reference.md) are the canonical reference. docs/ files extend them with plans, configurations, and reviews. If there's a conflict, project knowledge files are authoritative for current state; docs/ files are authoritative for planned future work.
