You are planning a feature for AudioHax, a Rust project that converts images into music via MIDI and also includes an MFSK data modem. The user will describe a feature they want to build.

Read the feature description provided as input: $ARGUMENTS

Then produce a task breakdown suitable for Claude Code agent teams by doing the following:

1. Read the root CLAUDE.md and docs/ROADMAP.md to understand project scope and current phase.
2. Identify which existing modules are affected. Read those source files to understand current state.
3. Determine if new modules are needed (check module boundary conventions in CLAUDE.md).

Produce a plan with these sections:

FEATURE SUMMARY: One paragraph restating the feature in concrete terms.

AFFECTED MODULES: List each file that needs changes, with a one-line description of what changes.

NEW MODULES: Any new files needed, with their responsibility and which existing module boundary they belong to.

TASK BREAKDOWN: Numbered tasks, each with:
- Description of the work
- File ownership (which files this task creates/modifies)
- Estimated complexity (LOW/MEDIUM/HIGH)
- Dependencies (which other tasks must complete first)

AGENT TEAM RECOMMENDATION: How many agents, which tasks each agent owns, and the coordination pattern (fan-out, pipeline, or hybrid). Follow the constraint: 3-5 agents max, always include a quality gate.

RISKS: Anything that could go wrong — module boundary violations, musical quality concerns, backward compatibility issues.

If the feature involves musical decisions, use proper music theory terminology. The project owner has a music performance degree and professional experience.
