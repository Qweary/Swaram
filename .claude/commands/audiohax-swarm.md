You are generating a complete Claude Code agent team configuration for AudioHax. The user will provide a phase name from docs/ROADMAP.md or a feature description.

Input: $ARGUMENTS

Step 1: Read the root CLAUDE.md, docs/ROADMAP.md, and the agent teams reference in the project knowledge to understand the work scope and team design constraints.

Step 2: If the input is a phase name (e.g., "Phase 1", "modem hardening"), find the corresponding section in ROADMAP.md and use its task breakdown. If it's a feature description, run the /audiohax-plan logic first to produce tasks.

Step 3: Read the source files that will be affected to understand their current state, public APIs, and existing test coverage.

Step 4: Design the agent team following these constraints:
- 3-5 agents maximum
- Always include a quality gate agent
- Each agent owns specific files with no overlap
- No agent modifies files outside its ownership scope
- Music pipeline agents must not touch modem files and vice versa

Step 5: Produce the complete configuration:

TEAM OVERVIEW: Team name, agent count, coordination pattern (fan-out, pipeline, hybrid), ASCII diagram showing agent relationships and file ownership.

AGENT DEFINITIONS: For each agent — name, role, files owned, files excluded, communication protocol.

SPAWN PROMPTS: Complete, self-contained spawn prompt for each agent. Each prompt must include:
- The agent's role and specific deliverables
- Which files it owns and must NOT modify
- Build/test/lint commands (cargo build --release, cargo test, cargo fmt, cargo clippy -- -W clippy::all)
- All project context needed (the agent has zero prior conversation history)
- Musical context if relevant (use proper theory terminology)
- Testing requirements specific to the agent's deliverables
- How to signal completion (mark task done, message lead)

TASK LIST: Tasks with IDs, titles, assignees, dependencies, and descriptions formatted for the Claude Code task system at ~/.claude/tasks/{team-name}/.

LEAD COMMAND SEQUENCE: The exact steps the user runs as team lead, including prerequisite checks, spawn order, monitoring, quality gate, integration pass, and final commit.

Make spawn prompts thorough — they're the agent's only context. Better to over-specify than under-specify.
