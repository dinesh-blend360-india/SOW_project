# Phase D report: checkpoint saving, flagged screen integration, API freeze

Date: 2026-11-09. Scope: CP-37 to CP-39. Core API frozen in `core_api_freeze.md`.

## Automated regression

| Suite | Result | Notes |
|---|---|---|
| Frontend `sow_module` (editor, shared, testing) | 167 passed | includes flag, provider, editor API, chrome tests |
| Frontend full suite | 484 passed, 162 failed, 3 skipped | see "Failures that existed before" |
| Frontend `DraftEditor.test.tsx` | 22 passed | unchanged |
| Frontend `DraftEditorScreen.integration.test.tsx` | 60 failed, 3 passed | identical to the baseline recorded in `prompt_log.md` |
| Node collab (`vitest`) | 74 passed, 6 skipped | 6 skipped are the live Snowflake integration tests (need credentials) |
| Backend full suite (`pytest`) | 556 passed, 9 failed | the 9 are the known date and prompt-wording failures (KNOWN_ISSUES #9) |
| Backend `sow_module` subset | 116 passed | |
| Typecheck (frontend, collab) | clean | |
| Lint (collab `eslint src`) | clean | |
| Production build (frontend) | passes | |

## Failures that existed before this project

- Frontend, jsdom `Failed to execute 'contains' on 'Node'` (antd Modal and Dropdown inside jsdom): `DraftEditorScreen.integration`, `SignOffScreen.integration`, `DealDeskScreen.integration`, `SowStudioPage`, `RiskFlagsTab`, `CopyPricingTemplateButton`, `PricingTemplateDetails`. The same 102 failures occur on the committed code without this phase's changes.
- Frontend `App.test.tsx`: `bootstrap/dist/css/bootstrap.min.css` is not transformed by Jest. Configuration issue, not this work.
- Backend, 9 known failures in `test_pricing_duration_schema.py` (3) and `test_sow_v4_templates.py` (6).

## Two-user manual test (flag on)

Prerequisites: backend running; collab service running with `COLLAB_*` set; two users, A and B, both with `FE_SOW_LEXICAL_EDITOR`; the same SOW open in two browsers. Build with `REACT_APP_SOW_LEXICAL_EDITOR=true` for admin testing only.

1. Open the SOW as A and as B. Both show the Lexical editor and a "Live" status.
2. Type in A. B sees the text within a second or two.
3. Click **Save** in A. The status goes Saving, then Saved; the version label in the screen changes to `vN+1`.
4. Open version history. A new revision with A as the author is listed.
5. Click **Save** in B with no edits. B gets "Saved" and no new version is created.
6. Make an edit in B and wait 30 seconds without saving. The Owner checkpoints by itself (quiet period). History shows the new version.
7. Change the draft through another path (for example an intake sync or an add-in save in another tab that is not the editor). Then edit in A and click Save. A sees a "changed outside the editor" notice and no version is written. Reload A to continue.
8. Log in as a user without the flag. The old DraftEditor appears, and nothing about the editor changes.

Record the result of each step below and attach screenshots.

| Step | Result | Date | Tester |
|---|---|---|---|
| 1 | | | |
| 2 | | | |
| 3 | | | |
| 4 | | | |
| 5 | | | |
| 6 | | | |
| 7 | | | |
| 8 | | | |

Not verified in this phase: the two-user run itself (needs running services and two accounts). The Node checkpoint, the source-change refusal and the flag routing are covered by automated tests.
