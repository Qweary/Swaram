Run a comprehensive health check on the AudioHax project and produce a structured report.

Execute these commands in sequence, capturing output from each:

1. `cargo check --release 2>&1` — compilation check
2. `cargo test 2>&1` — full test suite
3. `cargo clippy -- -W clippy::all 2>&1` — lint analysis
4. `cargo fmt -- --check 2>&1` — format check

Then produce a health report with these sections:

BUILD STATUS: Pass/fail, any compilation errors or warnings.
TEST RESULTS: Total tests, passed, failed, ignored. List any failures with test name and error.
LINT ISSUES: Count of clippy warnings by category. Flag any correctness or suspicious warnings.
FORMAT: Whether all files pass rustfmt check.
CODE QUALITY ASSESSMENT: Brief assessment of overall project health based on the above. Note any patterns (growing warning count, declining test coverage, etc.).

If any section has issues, prioritize them by severity: compilation errors first, test failures second, correctness lints third, style lints last.

Keep the report concise — focus on actionable findings, not passing results.
