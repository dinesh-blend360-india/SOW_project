# Architectural Decisions

This is the project-wide decision log. It does not duplicate the SOW editor module's own,
more detailed decision log — that log (`sow_docs/tracking/decisions.md`) is the authoritative,
actively-maintained ADR set for that module and is updated by that module's own workflow
(`blend_ode_be/sow_module/CLAUDE.md`). This file:

1. Records decisions about the **overall project structure** (the three-repo arrangement, this
   handoff system itself).
2. Indexes the SOW module's ADRs with a one-line summary and status, so a reader doesn't have to
   go hunting, without creating a second copy that can drift out of sync.

Do not silently change an **Accepted** decision below. To revisit one, add a new ADR that
supersedes it and says why. Use `templates/ADR_TEMPLATE.md` for new entries.

---

## Project-structure decisions

### ADR-001: Three independent Git repositories, no submodules
- **Date**: 2026-11-06
- **Status**: Accepted
- **Context**: The project consists of a frontend (`blend_ode`), a backend (`blend_ode_be`), and
  now a root-level documentation/handoff layer. The two application repos already exist
  independently on GitHub (`dinesh-blend360-india/blend_ode` and `.../blend_ode_be`, both with an
  `upstream` remote at `BLEND360/...`) and must keep their own history, branches, and remotes.
- **Problem**: How should the root-level documentation and agent-handoff system relate to the two
  application repos on disk and in version control?
- **Options considered**:
  1. Git submodules for `blend_ode`/`blend_ode_be` inside the root repo — rejected: submodules
     pin a single commit per app repo inside the root repo's tree, which fights with two
     independently-developed repos and adds submodule-update ceremony for no real benefit here.
  2. Merge all three histories into one repo — rejected: destroys the existing GitHub remotes and
     independent PR/review history of both app repos.
  3. Three fully independent repos on disk, sibling directories, root repo's `.gitignore`
     excludes the application directories entirely — **chosen**.
- **Decision**: `blend_ode/` and `blend_ode_be/` remain independent Git repositories, cloned as
  sibling directories under the root. The root repository's `.gitignore` excludes both (plus a few
  other local-only paths — see `.gitignore` in the root for the exact, user-maintained list).
- **Reasoning**: Preserves each app repo's own GitHub remote, history, and branch, while still
  allowing one place (the root) to hold documentation that spans both.
- **Consequences**: A developer must clone three repositories to get a working tree (see
  `LOCAL_DEVELOPMENT_SETUP.md`). Git operations (status, commit, push) must always be run inside
  the specific repo being changed — never assume `git status` at the root reflects the app repos.

### ADR-002: `agent-handoff/` as the cross-repo development memory
- **Date**: 2026-11-06
- **Status**: Accepted
- **Context**: Development on this project spans multiple Claude Code sessions, potentially by
  different people, over time. Without a durable, structured record, each new session re-derives
  context by re-reading source code and re-asking the same questions.
- **Problem**: Where does "what is this project, what's been decided, what's in progress, what
  should I do next" live, in a form that survives a session ending?
- **Options considered**:
  1. Rely on commit messages and code comments alone — rejected: doesn't capture architecture,
     rationale, or in-progress/blocked state.
  2. A single large root `README.md` — rejected: becomes unmaintainable at this scope and mixes
     onboarding, architecture, process, and status into one file nobody keeps current.
  3. A structured `agent-handoff/` directory, one file per concern, with templates for the
     append-heavy ones (session log, task board, changelog) — **chosen**.
- **Decision**: See `agent-handoff/README.md` for the full file inventory and reading order.
- **Reasoning**: Separating "stable reference" docs (architecture, contracts) from "living" docs
  (task board, session log, changelog) lets each be maintained at the cadence it actually needs,
  and keeps any single file small enough to stay readable.
- **Consequences**: This system only stays useful if every session actually updates it (see
  `DEVELOPMENT_WORKFLOW.md`). An unmaintained handoff system is worse than none, because it then
  actively misleads the next reader.

---

## SOW editor module ADRs (index only — full detail in `sow_docs/tracking/decisions.md`)

The SOW editor module (Lexical + Yjs live-collaboration editor, replacing CKEditor in SOW Studio)
has its own decision log, maintained since the module's first checkpoint (CP-05, 2026-10-30). All
of these currently carry status **Proposed** in that file except where noted; "Proposed" there
does not mean undecided in practice — several have already been implemented and are in active use
(flagged below). Read the source file for full context, consequences, and open questions before
changing any of them.

| ID | One-line decision | Status (per decisions.md) |
|---|---|---|
| ADR-001 | HTML (`SOW_DRAFT_VERSIONS.CONTENT`) is the system of record; Yjs state is disposable working state | Proposed, implemented |
| ADR-002 | Checkpoints only ever write through the existing `save_sow_draft` | Proposed, implemented |
| ADR-003 | The Node collab service writes only `SOW_COLLAB_*` tables, never existing app tables | Proposed, implemented |
| ADR-004 (D-B1) | Relationship to the native-editor pilot's guest/mention/audit tables — reuse via FastAPI only | Proposed, **not yet implemented** (guest sessions deferred to CP-60) |
| ADR-005 (D-B2) | Node + Express + `ws` + Yjs as a separate deployable (a Python-only relay was rejected — no Lexical validation in Python) | Proposed, implemented |
| ADR-006 | Memory-first, batched durable log to `SOW_COLLAB_UPDATE` | Proposed, implemented |
| ADR-007 | Single-writer lease with a fencing token, taken via `MERGE` (Snowflake has no enforced uniqueness) | Proposed, implemented at the table/data-layer level; not yet wired into the live WebSocket path |
| ADR-008 (D-B4) | `blend_ode/src/sow_module/shared/` is the source of truth for shared TypeScript; vendored into the backend with a SHA-256 drift check | Proposed, implemented |
| ADR-009 (D-B3) | The module owns its own HTML allowlist (`html_allowlist.py` / `html-contract.json`); never uses the existing `sanitize_rich_text` | Proposed, implemented |
| ADR-010 | Tracking files and the Node service live in the backend repo | Proposed, implemented (tracking docs later relocated to `sow_docs/`, see note below) |
| ADR-011 | Permission changes: 60-second re-check poll plus an immediate `access-changed` hook | Proposed, partially implemented (the hook exists for guest revoke; the 60-second poll is not yet built) |
| ADR-012 | All `SOW_COLLAB_*` tables live in `BLEND_INTERNAL_IT_APPS.NEXDRAFT`, fully qualified | Proposed, implemented; grant confirmed live 2026-11-06 |
| (unnumbered) | Standard Snowflake tables, not hybrid tables, for `SOW_COLLAB_*` | **User decision, 2026-11-06** — made with partial information (the hybrid-table grant was denied; CP-09's latency gate was never run). See `sow_docs/tracking/phaseB_report.md` and `migrations/011_sow_collab_live_workspace.sql`'s header comment. |

**Open question still awaiting the user**: decisions.md's Q2 — who exactly counts as `sow_admin`
(finalize-capable)? The code currently treats `FE_ADMIN` as the only elevated flag; this is used as
a documented stand-in in CP-19/20, not a resolution. See `sow_docs/tracking/decisions.md` for the
full question list and the one correction already made in a later session: tracking docs that
ADR-010 describes as living in `blend_ode_be/sow_module/_tracking/` were moved to the root-level
`sow_docs/tracking/` on 2026-11-06 to make them readable without entering either app repo. See
`sow_docs/tracking/prompt_log.md`'s entry around that date (if logged) or the session log in this
handoff system for the move itself.
