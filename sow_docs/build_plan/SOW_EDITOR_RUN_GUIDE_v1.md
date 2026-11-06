# SOW Editor — Run Guide (how to work through the prompts)

**Date:** 2026-10-29
**Use with:** `SOW_EDITOR_BUILD_PROMPTS_v4.md` (75 prompts, CP-00 to CP-74, matched to your real code). The v1, v2 and v3 prompt files are superseded (v4 fixes mismatches found by checking the prompts against your real repositories).
**Audience:** you, running Claude Code in VS Code.

This is the short, practical loop. The big prompt file has the detail; this is the page you keep open while you work.

---

## 1. The core loop (do this for every prompt)

```
  ┌─────────────────────────────────────────────────────────────┐
  │  1. /clear   (fresh Claude Code session)                    │
  │  2. Paste one prompt (BEGIN PROMPT … END PROMPT)            │
  │  3. Let Claude Code do the work                            │
  │  4. Read its closing line:                                  │
  │        READY FOR NEXT PROMPT  → go to next prompt (step 1)  │
  │        STOP AND TEST          → go to section 2 below       │
  └─────────────────────────────────────────────────────────────┘
```

That is the whole thing. Most prompts end in READY FOR NEXT PROMPT and you just continue. Roughly every 3rd or 4th prompt ends in STOP AND TEST, and that is where you test and commit.

---

## 2. What to do at a STOP AND TEST checkpoint

Every STOP AND TEST prompt prints: what to run, what you should see, and a suggested commit message. Follow these five steps.

1. **Run the checks it names.** Usually one or more of:
   - `blend_ode_be/sow_module/run_checks.sh` (runs the frontend module tests, the Node service tests and the backend tests)
   - the Snowflake scratch-schema integration tests (from CP-18 onward)
   - the existing `blend_ode` Jest suite and `blend_ode_be` pytest suite, to prove nothing outside the module broke
   - a two-browser manual check where the prompt says so
2. **Look at the acceptance checks** in that prompt. Each check is a yes/no. If any fails, do **not** commit — go to section 4 (fixing).
3. **Record it in the test log.** Open `sow_docs/tracking/test_log.md` and add one row:
   `| 2026-xx-xx | CP-21 | typecheck + unit + boundary checks | pass | notes | <hash after commit> |`
4. **Commit.** Use the message the prompt suggested, for example:
   `git add -A && git commit -m "feat(sow_module): fastapi bridge, collab service skeleton, safety checks"` (this one is CP-21)
   `blend_ode` and `blend_ode_be` are **two separate git repositories** (`SOW_project` is not a repo). Commit in **every repo that changed**, with the same message, and record both hashes in the test-log row (it has a column for each).
   Then paste the commit hash back into the test-log row from step 3.
5. **Continue** to the next prompt with a fresh `/clear`.

> Claude Code never runs `git commit` for you. You run it. That keeps you in control of what enters history.

---

## 3. Before you start at all (one-time)

- Decide your model: use the strongest your plan offers for the spikes (CP-08 to CP-15), since those spikes decide the architecture. Opus-class for Phases B, D, E; a lighter model is fine for Phases G, H, J.
- Have your git working tree **clean** before CP-00, so the first commit is just the scaffold.
- Open VS Code at `sow_project/` (the folder that contains both `blend_ode` and `blend_ode_be`). Tracking files and the Node service live in the backend repo under `blend_ode_be/sow_module/`.
- From CP-08, have a Snowflake user and role for the new Node service and a **scratch schema** ready (you have only one database, so the prompts never touch your real tables). The prompt will ask. **No SQL migration is ever applied without your approval (CP-17).**
- Keep `SOW_EDITOR_BUILD_PROMPTS_v4.md` open in one pane and this guide in another.

---

## 4. When a check fails

1. Do not commit a failing checkpoint.
2. In the **same** Claude Code session (do not `/clear` yet), tell it plainly what failed, e.g. "TST-P2-006 fails: table column widths reset after reload. Fix it." Give it the error output.
3. Let it fix and re-run.
4. When green, record and commit as in section 2.
5. If it is stuck after a couple of tries, that is a signal to pause and ask me (in this chat) or adjust the prompt.

---

## 5. Phase-end checkpoints are bigger

Some STOP AND TEST lines say "This closes Phase N." At those, also:
- Confirm the phase report in `sow_docs/tracking/` exists (for example `phaseB_report.md`, written at CP-15) and reads sensibly.
- Make the commit message the one given (it ends with "phase N complete").
- This is a natural point to tag the commit if you like: `git tag sow-phase-1`.

---

## 6. Gate and decision checkpoints (don't skip the thinking)

A few checkpoints ask you to **review with the user** before continuing, not just run tests:
- **CP-09, CP-11 and CP-15** — the Snowflake latency, lease and crash-durability results and the GO/NO-GO. If these look bad, stop and bring them back to me before Phase C.
- **CP-16 and CP-17** — confirm the migration number (the repo stops at 010 but the code mentions 013), then approve (or reject) the SQL migration before anything is applied to your Snowflake database.
- **CP-47 and CP-68** — measured scale gates. Approve the real numbers or send them back; never let a gate be relaxed silently.
- **CP-54** — decide what happens to the existing native editor (decision D-B1) before comments and guests are built.

For these, add a line to `test_log.md` recording the decision, not just a pass/fail.

---

## 7. If you stop for the day

- Finish the prompt you are on to a READY or STOP line; don't leave it half-done.
- Make sure the work is committed (or at least the tree is in a known state).
- The next day, open a fresh `/clear` and the next prompt. Because every prompt reads `prompt_log.md`, Claude Code picks up the context itself.

---

## 8. Quick reference: the 28 test/commit points

Checkpoints fall at: CP-00, CP-04, CP-07, CP-09, CP-11, CP-15, CP-18, CP-21, CP-25, CP-28, CP-30, CP-33, CP-35, CP-39, CP-42, CP-45, CP-47, CP-50, CP-53, CP-56, CP-58, CP-61, CP-64, CP-66, CP-68, CP-71, CP-73, CP-74.

Between any two of these, you are just running prompts back to back.

---

## 9. One-line summary

**Clear → paste prompt → work → if it says STOP AND TEST, run the checks, log them, commit with the given message → next.**