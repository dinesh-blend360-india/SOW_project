# Phase F report: intake, legacy opening, regeneration safety (CP-26 to CP-53)

Date: 2026-11-12. Status: **complete in code, with live checks still pending** (see Open items).

## 1. Sources wired through the intake pipeline

Every content source reaches the document through `shared/intake.ts` and the shared allowlist, or through a catalog path that the tests check.

| Source | Path | Intake | User-facing notice |
|---|---|---|---|
| Paste (Word, web, plain) | `editor/paste.ts` | yes (`intake`) | "Some formatting was removed..." for everyone; details for SOW admins |
| Markdown-looking paste | `paste.ts` smart paste | yes (`markdown`) | "Pasted text was formatted as Markdown..." with undo to plain text |
| LLM insert (`insertContent`) | `editorApi.ts` | yes (`intake`, format from the caller) | "Some formatting was removed..." (new, CP-53); audit row per insert (CP-49) |
| Streaming preview commit | `streaming.ts` | yes (commits through `insertContent`) | as LLM insert. No screen starts a stream yet (no streaming endpoint) |
| Add-in insert | `DraftEditorScreen` + `sowStudioAddInInsertion` | catalog text, not user input | none; a test checks every catalog add-in passes intake unchanged |
| Legacy open (CKEditor-era draft) | `legacy-baseline.ts`, `ConversionReportNotice` | conversion, read-only report | "written in the previous editor... changes some formatting" for everyone; element and class counts for admins (CP-53) |
| Draft load from the server | `applyLoadedDraft` | server-side allowlist on save; unsupported tags block editing | warning when the draft has tags the live editor cannot edit |
| Room replacement (regenerate, intake sync, continue, accept, reload) | `room-replace.ts` | the source is the stored, already-allowlisted draft | "The draft was regenerated..." notice to everyone in the room (CP-52) |

The Owner validates every incoming update against the same allowlist before applying it (`validation.ts`), so a client that skips the frontend intake still cannot write a disallowed element.

## 2. Fidelity summary

- **Automated:** the fidelity test suites (`testing/intake.test.ts`, `sow_module/testing/sanitizeHtmlParity.test.ts`, `tests/test_sow_module_html_parity.py`) compare the TypeScript and Python allowlists on the same hostile corpus. All pass.
- **Conversion of older drafts:** `legacy-baseline.ts` converts a stored draft the way the live editor does. `npm run fidelity:report` (in `blend_ode_be/sow_module/collab`) checks a folder of exported drafts for lost text, instability and conversion errors. **It has not been run on real drafts yet.** That needs exports from you (TODO_FOR_YOU, CP-51 step 6).

## 3. Fuzz coverage (CP-53)

`blend_ode/src/sow_module/testing/intake.fuzz.test.ts`, fast-check, fixed seed, 2,000 random documents per format:

- **HTML:** no element, attribute, class or style property outside the contract survives.
- **Markdown:** same invariant after markdown-it rendering.
- **Plain text:** same invariant after escaping.
- **Stability:** running intake on its own HTML output changes nothing (500 documents).
- **Control:** the checker is tested against known-bad markup, so a passing run means the checker can fail.

Result: all five pass. Two false alarms in the checker itself were fixed (inert parsing; `class` and `style` are checked by their own rules, and empty leftovers are allowed). No sanitizer bypass was found.

## 4. Regression (run 2026-11-12)

| Suite | Result |
|---|---|
| Frontend `sow_module` (jest) | 285 passed (one run had a timing flake in `SowLexicalHarness`; re-run passed) |
| Frontend `tsc`, eslint | clean; 2 pre-existing lint errors in `testing/chunks.test.ts` and `testing/intake.test.ts` |
| Collab service (vitest) | 200 passed, 7 skipped (need live Snowflake) |
| Collab `tsc`, eslint, module boundary | clean |
| Shared copies (`sync_shared.sh --check`) | in sync |
| Backend full suite (pytest) | 588 passed, 9 failed. The 9 are the known pre-existing failures (duration and template tests) |
| Native collaboration and mail tests | pass (28 with the new participants tests) |

## 5. Open items

- Two-browser checks for CP-49 to CP-52 (`TODO_FOR_YOU.txt`).
- Live Snowflake checks: audit rows, guest expiry filter, lease renewal and atomic fencing (KNOWN_ISSUES #14, #16).
- Real-draft fidelity run (needs exports; the script is ready).
- Add-in text is catalog text, not user input; it is checked by test, not by a runtime intake step.
- No streaming LLM endpoint yet, so the streaming preview has no caller.
- Pre-existing items unrelated to Phase F: `core.autocrlf` (KNOWN_ISSUES #13), the 9 backend test failures, the 2 lint errors above.
- Decision needed before Phase G: D-B1 (see `design_collab_features.md`).
