# sow_module

Live-collaboration editor module for SOW Studio (Lexical + Yjs, HTML stays the system of record).

- Plan: `SOW_EDITOR_BUILD_PROMPTS_v1.md` and `SOW_EDITOR_RUN_GUIDE_v1.md` in `sow_project/`.
- Rules and layout: [CLAUDE.md](../../blend_ode_be/sow_module/CLAUDE.md). Progress: [prompt_log.md](tracking/prompt_log.md), [test_log.md](tracking/test_log.md).
- Run everything: `bash sow_module/run_checks.sh` (needs Node 20+, `npm install` in `collab/`, and `npm install` in `blend_ode`).
- After editing `blend_ode/src/sow_module/shared/`: `bash sow_module/sync_shared.sh`.
