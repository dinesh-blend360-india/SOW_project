# Test log

| Date | CP-ID | What was tested | Result | Notes | Commit hash (frontend) | Commit hash (backend) |
|---|---|---|---|---|---|---|
| 2026-10-30 | CP-00/06/07 (tooling) | run_checks.sh: shared sync, Jest, shared tsc, collab tsc + Vitest, backend pytest | pass (6/6) | Node 22.23.4; role table/contracts not built yet | (uncommitted) | (uncommitted) |
| 2026-10-30 | CP-05/06/07 | run_checks.sh: shared sync check, Jest (roles 9 + drift), shared tsc strict, collab tsc, collab ESLint, Vitest (3), backend pytest (2) | pass (7/7) | Full frontend suite also run: 34 suites pass, 2 fail (DraftEditorScreen.integration, SowStudioPage; timeouts, no importer of src/sow_module, baseline not compared) | (uncommitted) | (uncommitted) |
