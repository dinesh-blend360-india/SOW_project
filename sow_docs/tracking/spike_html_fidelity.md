# HTML <-> Lexical fidelity spike (CP-12) — Part 1

**Status of this part: DONE, fully run.** Unlike CP-08 to CP-11, this spike needs no Snowflake
access — it is pure frontend/Node logic, and every number below is from an actual test run, not a
projection. Part 2 (Python sanitizer/exporter parity and the HTML GO/NO-GO decision) is CP-13's job,
not this file's.

## Fixtures

20 real sanitized draft samples were requested from the user; none were available yet (confirmed
2026-11-01 — "We dont have other data currently"). Per the CP-12 prompt's own fallback, **23
synthetic fixtures** were written instead, built directly from the audited CKEditor configuration
(`audit_frontend.md`), each prefixed `synthetic-` and containing no customer data:
`blend_ode/src/sow_module/shared/fixtures/html/synthetic-01-short-paragraph.html` through
`synthetic-23-long-document.html`. Coverage: short and long documents, nested headings, lists to
depth 3 (bulleted, numbered, and mixed), six table variants (simple, colspan, rowspan, a
colspan+rowspan combination, the exact Retainer and Fixed-Fee billing-schedule header shapes from
`billing_schedule.py`), a signature block matching `_signatory_fields_are_complete`'s expected
shape, all 7 highlight classes, an unresolved placeholder, font-size and font-family spans, all 4
text-align values, a link, a CKEditor-style `<figure class="table">` wrapper, `<div>/<section>/
<font>` wrappers, and the `b/em/strike/del` alias tags.

**This set should be replaced or supplemented with real sanitized drafts as soon as they're
available** — see `SOW_project/testing_pending_guide.md`. Nothing here should be read as proof
against production HTML, only against CKEditor's documented configuration.

## Lexical packages and TypeScript compatibility

Added to `blend_ode/package.json`, all pinned to the exact same version, no existing dependency
changed: `lexical`, `@lexical/react`, `@lexical/rich-text`, `@lexical/list`, `@lexical/table`,
`@lexical/html`, `@lexical/selection`, `@lexical/utils`, `@lexical/clipboard`, `@lexical/mark`,
`@lexical/yjs`, `@lexical/headless`, plus `yjs` (pinned to the same `13.6.33` the backend collab
service resolved to).

**Version chosen: 0.44.0, not latest (0.53.0).** The latest Lexical release declares a peer
dependency of `typescript: '>=5.2'`; this project is pinned to TypeScript `^4.9.5` and the CP-12
prompt is explicit: *"if Lexical's types do not compile under 4.9.5, stop and report instead of
upgrading TypeScript."* Bisecting the version history found that the `>=5.2` peer requirement was
introduced at Lexical `0.46.0`; `0.44.0` (published 2026-10-30, one day before this spike) declares
no TypeScript peer dependency and was confirmed, empirically, to compile the whole existing frontend
project cleanly under `tsc --noEmit` with TypeScript 4.9.5 (`src/sow_module/testing/lexicalTsCheck.ts`
exercises the type surface of every added package). **TypeScript was not upgraded.**

`snowflake-sdk`-style vulnerability scanning: `npm audit` reports pre-existing moderate/high findings
typical of this CRA 5 dependency tree; nothing newly introduced here was flagged as more severe than
what the existing `react-scripts` toolchain already carries.

## The allowlist (`shared/allowlist.ts`)

Implements the module's own HTML allowlist (D-B3, decisions.md) on top of `html-contract.json`:

- `sanitizeImportHtml` — unwraps `figure`/`div`/`section`/`font`/`a` (the link-handling question was
  put to the user; answer confirmed 2026-11-01: **links are reduced to plain text**, not preserved —
  this is why `a` is in `importUnwrap`, not `tags`), drops `script`/`style`/`iframe`/etc., and renames
  alias tags (`b`→`strong`, `em`→`i`, `strike`/`del`→`s`) — all before the HTML ever reaches Lexical.
- `sanitizeExportedHtml` — strips any class, style property, or attribute the contract doesn't list
  for that tag, run on Lexical's output before it's trusted anywhere else.
- `buildHtmlConfig` — registers two custom Lexical import/export overrides, both discovered to be
  necessary by running the real test, not anticipated in advance (see "Defects found and fixed"
  below).

**The contract itself (`html-contract.json`) needed no changes.** Every tag, class, style property,
and attribute the fixtures required was already present from CP-05's design (derived from the
CKEditor audit). This is a byproduct of the fixtures being synthesized *from* that same audit rather
than independently sourced; real draft samples may still surface gaps the audit missed — the file's
`unverified` list (table inline styles, `colgroup`/`col` details, cell `text-align`) stands as-is.

## Defects found and fixed during this spike

Two real defects were found by running the test, not predicted:

1. **`@lexical/mark`'s `MarkNode` cannot round-trip `<mark>` HTML at all.** `MarkNode.importDOM()`
   returns `null` (it only supports being created programmatically, for comment-range marking), and
   — more subtly — its `excludeFromCopy()` method is hardcoded to `destination !== 'clone'`, meaning
   it **always** excludes itself from HTML export, by design, for its intended use case (the
   wrapper shouldn't appear in copied text, only the text itself). Confirmed by an isolated probe
   test before any fix: a `<mark class="marker-yellow">` round-tripped as plain unwrapped text, with
   the highlight silently gone. **Fix:** `shared/SowHighlightNode.ts`, a `MarkNode` subclass that
   overrides only `excludeFromCopy()` to return `false`, reusing every other MarkNode mechanism
   (the `ids` field used to carry the class list, `createDOM`/`updateDOM`, JSON shape). Registered via
   a custom `html.import`/`html.export` pair in `buildHtmlConfig` that encodes/decodes the class list
   through `ids`.
2. **Lexical's default `TextNode.exportDOM()` double-wraps every formatted run in legacy tags for
   copy-paste compatibility** — its own source comment says so explicitly: bold exports as
   `<b><strong style="white-space:pre-wrap">text</strong></b>`, italic as `<i><em>text</em></i>`,
   strikethrough as `<s><span>text</span></s>`. This is the right default for a general-purpose
   editor and the wrong shape for a fixed HTML contract (canonical: `strong`/`i`/`u`/`s`, no
   double-wrapping, no `<em>`/`<b>`). **Fix:** a custom `TextNode` exporter registered in
   `buildHtmlConfig` that emits exactly the contract's tags with no outer legacy wrapper.
3. **A third, smaller issue**: Lexical's table/block exporter always writes an explicit
   `style="text-align: start"` even when no alignment was ever set, using `"start"` as its own
   internal convention for "default." Re-importing that explicit value then set an actual format
   where there had been none, breaking byte-for-byte JSON stability with zero visible or semantic
   effect. **Fix:** `sanitizeExportedHtml` now treats `text-align: start` as equivalent to absent.

All three fixes are in `shared/allowlist.ts` / `shared/SowHighlightNode.ts`, covered by the test.

## A fourth finding: `@lexical/headless`'s default DOM is not safe for tables

`@lexical/headless` ships `withDOM()`, a convenience helper that provides a `happy-dom`
window/document for running Lexical outside a browser. Testing the identical fidelity logic through
it (as a first attempt at the "and in Node" requirement) found that **every fixture containing a
`<table>` silently lost the table entirely** on import under happy-dom — no error thrown, the table
simply never appeared in the re-exported HTML — while every non-table fixture round-tripped
correctly. Swapping to a directly-constructed `jsdom.JSDOM` instance in the same Node script fixed
every table fixture with no other code change, confirming the gap is in `@lexical/headless`'s bundled
`happy-dom`, not in this module's own code. **`scripts/check-html-fidelity-node.ts`** uses `jsdom`
directly for this reason, and both the browser's real DOM and `jsdom`-in-Node pass all 23 fixtures;
`happy-dom` via `withDOM()` does not.

**Consequence for CP-37** ("Owner checkpoint to the existing draft system," which the build plan
describes as "Export the live document to HTML with headless Lexical and pass it through the
allowlist"): if that export runs inside the Node collaboration service, it must not rely on
`@lexical/headless`'s default `withDOM()` for any document containing a table — it needs `jsdom`
(or an equivalent) explicitly. Recorded in `testing_pending_guide.md` and flagged here so CP-37
doesn't rediscover it from a production failure.

## Per-fixture results

All 23 fixtures: imported, exported, and semantically compared (normalized text-content match) with
**zero defects**, and JSON-stable on a second pass, in both test environments that actually matter —
Jest/jsdom (`npm test` — the same environment this entire project's existing suite already uses) and
plain Node with `jsdom` (`npm run check:html-fidelity:node`). 30/30 Jest assertions pass; 23/23 pass
in the standalone Node script.

| # | Fixture | Jest (jsdom) | Node (jsdom) | Node (happy-dom, informational only) | Notes |
|---|---|---|---|---|---|
| 1 | synthetic-01-short-paragraph | PASS | PASS | PASS | |
| 2 | synthetic-02-headings-nested | PASS | PASS | PASS | |
| 3 | synthetic-03-list-bulleted-depth3 | PASS | PASS | PASS | |
| 4 | synthetic-04-list-numbered-depth3 | PASS | PASS | PASS | |
| 5 | synthetic-05-list-mixed-nested | PASS | PASS | PASS | |
| 6 | synthetic-06-table-simple | PASS | PASS | **FAIL (table lost)** | happy-dom finding, see above |
| 7 | synthetic-07-table-colspan | PASS | PASS | **FAIL (table lost)** | ″ |
| 8 | synthetic-08-table-rowspan | PASS | PASS | **FAIL (table lost)** | merged-cell survival confirmed (jsdom) |
| 9 | synthetic-09-table-merged-cells | PASS | PASS | **FAIL (table lost)** | colspan+rowspan combo, confirmed (jsdom) |
| 10 | synthetic-10-table-billing-schedule-retainer | PASS | PASS | **FAIL (table lost)** | exact `billing_schedule.py` header text preserved |
| 11 | synthetic-11-table-billing-schedule-fixed-fee | PASS | PASS | **FAIL (table lost)** | ″ |
| 12 | synthetic-12-signature-block | PASS | PASS | **FAIL (table lost)** | matches `_signatory_fields_are_complete` shape |
| 13 | synthetic-13-highlights-all-types | PASS | PASS | PASS | all 7 highlight classes confirmed |
| 14 | synthetic-14-placeholder-unresolved | PASS | PASS | PASS | bracket text and marker class both confirmed |
| 15 | synthetic-15-font-size-spans | PASS | PASS | PASS | |
| 16 | synthetic-16-font-family-spans | PASS | PASS | PASS | |
| 17 | synthetic-17-text-align-variations | PASS | PASS | PASS | |
| 18 | synthetic-18-link-reference | PASS | PASS | PASS | link reduced to text, confirmed |
| 19 | synthetic-19-figure-table-wrapper | PASS | PASS | **FAIL (table lost)** | `<figure>` unwrapped |
| 20 | synthetic-20-alias-tags-mixed | PASS | PASS | PASS | b/em/strike/del normalized |
| 21 | synthetic-21-inline-formatting-combo | PASS | PASS | PASS | nested bold+italic+underline |
| 22 | synthetic-22-div-section-font-wrappers | PASS | PASS | PASS | div/section/font unwrapped |
| 23 | synthetic-23-long-document | PASS | PASS | **FAIL (table lost)** | full combined draft, two tables |

## Approved-loss (structural differences with no content or semantic impact)

- **Every text run exports as its own `<span>`**, even unformatted ones (Lexical's default). This is
  cosmetic; the module's own existing tests confirm text content and all semantic markers (highlight
  classes, placeholder text) survive regardless of this per-run wrapping, and the backend's
  regex/HTMLParser-based extractors (`_HtmlToBlocksParser`, `_signatory_fields_are_complete`,
  `completed_billing_schedule`) all strip inline tags before reading text, so this is invisible to
  them.
- **Table cell content is always wrapped in an inner `<p>`** on export (`<td><p><span>...`), even when
  the source had bare text directly inside `<td>`. Semantically identical; confirmed not to break the
  backend's cell-text extraction, which also strips inner tags.
- **`<colgroup><col>...</colgroup>` appears on every exported table**, even when the source table had
  none. Already an allowed tag in the contract (anticipated from the `TableColumnResize` plugin in
  the CKEditor audit), so this is not a new allowance, just newly confirmed to actually occur.

## Open items (not resolved by this checkpoint)

1. **Table inline styles (border, background, width, height, padding, vertical-align) remain
   unverified** — no fixture exercised them, consistent with `html-contract.json`'s own
   `unverified` list. Needs either real draft samples or a deliberately styled synthetic fixture.
2. **`data-add-in-id` was not exercised by any CP-12 fixture.** CP-02's audit flagged this as a risk
   (CKEditor is known to strip `data-*` attributes on serialization; the frontend already has a
   text-matching fallback). Whether Lexical's import/export preserves it needs its own test, planned
   for the add-in checkpoints (CP-36).
3. **The `@lexical/headless` / happy-dom table gap (above) is unresolved upstream** — worked around
   for this spike's own Node script, but needs a deliberate decision in CP-37 about which DOM
   implementation the Owner's server-side export uses.
4. Real sanitized draft samples are still needed to validate anything beyond what the CKEditor audit
   already predicted.

---

# Part 2 (CP-13): Python parity, hostile inputs, and the GO/NO-GO

**Status: DONE, fully run.** No Snowflake access needed here either.

## The backend's own allowlist (`src/sow_module/html_allowlist.py`)

New code, `sanitize_draft_html(html)`, driven by the same vendored `html-contract.json` the
TypeScript side uses (kept in sync by `sync_shared.sh`, which now also copies the contract into
`src/sow_module/` for runtime use, not just into test fixtures). **`application/rich_text.py` was
not edited, and `sanitize_rich_text` is never called from this module's code** — confirmed by the
test file only calling it for a side-by-side report, never asserting equality against it.

Structurally this mirrors the existing `_RichTextSanitizer`'s streaming approach (a tag outside the
allowed set is unwrapped by default — content keeps flowing, only the tag is dropped; a small set
of tags drops its content entirely), generalized to read tags/classes/styles/attributes from the
JSON contract instead of a hardcoded Python set, plus the contract's alias renaming
(`b`→`strong`, `em`→`i`, `strike`/`del`→`s`).

## Why parity is semantic, not byte-for-byte

CP-13 asks for `sanitize_draft_html`'s output to "equal" the TypeScript allowlist's expected
output. Taken literally (byte-for-byte), this isn't achievable without a browser-grade DOM engine
in Python, which isn't warranted here: jsdom (a full HTML5 parser, used by the TypeScript side) and
Python's stdlib `html.parser.HTMLParser` (a streaming parser, the same one already used throughout
this codebase) are different engines that will legitimately disagree on edge cases no real draft
HTML ever produces — e.g. jsdom's HTML5 table content-model silently drops a `<td>` found outside a
`<table>`, while a streaming parser (lacking that content model) keeps it. Neither behavior is a
security or content defect; they are different, equally safe answers to a question real HTML never
asks (CKEditor and Lexical both always emit properly nested tables).

**Parity is therefore defined and tested as:** the normalized *visible text* must match exactly
(confirmed after accounting for one cosmetic difference below), and the *set of surviving CSS
classes* must match exactly, for every one of the 23 real fixtures. One cosmetic difference was
found and is not a defect: Python's `html.escape()` quotes a literal `"` in text content as
`&quot;`, which is optional (not required) in text content per the HTML spec; jsdom's serializer
leaves it as a literal `"`. Both render identically to a viewer. The parity test
(`test_sow_module_html_parity.py`) decodes entities before comparing text for exactly this reason.

**Result: 23/23 real fixtures pass semantic parity** (`test_sanitize_draft_html_matches_typescript_allowlist_semantically`).

## Export acceptance

**23/23 real fixtures pass** `build_sow_docx_bytes` and `build_sow_pdf_bytes` with no error
(`test_lexical_html_exports_to_docx_and_pdf_without_error`) — confirming the HTML this editor
produces is accepted by the existing, unmodified export pipeline.

## What `sanitize_rich_text` removes (report only, no equality asserted)

Run against all 23 fixtures (`test_sanitize_rich_text_report_only`), confirming the known,
pre-existing limitation documented in `decisions.md` ADR-009 and `audit_backend.md` Q2: it strips
every `<mark>` tag and its classes entirely (so highlights and placeholders disappear from the
public approval preview — already known, not caused by this module), strips the CKEditor
`<figure class="table">` wrapper, and strips `<del>`/`<strike>`/`<font>`/`<section>` (not on its
allowlist). It does **not** strip `<div>` (unlike `sanitize_draft_html`, which unwraps `<div>` per
the contract) — a deliberate difference between the two allowlists' designs, not a bug in either.

## Hostile inputs

Six hostile fixtures (`shared/fixtures/hostile/`): a `<script>` tag plus `onclick`/`onmouseover`
attributes, a `javascript:` link, a malformed `url(javascript:...)` inline style, a 500-level deep
nesting bomb, a 200-row table, and literal markdown-looking text with an embedded `<script>` and a
markdown-syntax `javascript:` link. **All six are neutralized by both allowlists** — no fixture
survives with a live `<script>`, a live `on*=` attribute, or a live `javascript:` URL
(`test_hostile_inputs_neutralized_by_python_allowlist` and the TypeScript-side
`writes sanitized HTML for every hostile fixture` Jest test).

**One real defect was found and fixed during this work**, on the TypeScript side: the malformed
`url(javascript:alert('xss'))` style broke jsdom's CSS parser badly enough that
`element.style.length` reported zero properties, meaning the per-property removal loop never ran at
all and the raw, unparsed, dangerous style text would otherwise have survived untouched in the
output `style` attribute. Fixed in `sanitizeExportedHtml` with a raw-text safety-net check
(`javascript:` or `expression(` anywhere in the attribute's raw text forces the whole `style`
attribute to be dropped, independent of whether the browser's CSS parser could make sense of it).
The equivalent Python code does not have this gap in the same form, because it parses style
declarations itself with a simple regex rather than delegating to a DOM's CSS object model — but it
carries the identical raw-text safety-net check anyway, for defense in depth.

The depth bomb (500 nested `<div>`s) and the giant table (200 rows) were both processed in well
under a second on both sides, with no crash, no hang, and no stack-depth error.

## GO / NO-GO verdict: **GO — HTML stays the system of record.**

Reasons:

1. Every one of the 23 synthesized-from-the-real-CKEditor-config fixtures round-trips through
   Lexical with zero defects (Part 1) and passes the backend's own allowlist with matching visible
   text and matching surviving classes (Part 2).
2. The existing, unmodified DOCX and PDF exporters accept every fixture's HTML without error.
3. Both allowlists (TypeScript and Python) independently neutralize the same set of hostile inputs,
   including one real defect found and fixed during this work — found *because* real tests were run
   against real hostile input, not assumed safe.
4. The backend's existing `sanitize_rich_text` (used only for the separate public approval page)
   continues to work unmodified and untouched, confirming this module has not disturbed it.
5. The one open risk — real draft HTML samples still don't exist, so everything here is validated
   against the CKEditor *configuration*, not CKEditor's actual historical output — is already
   tracked in `SOW_project/testing_pending_guide.md` and does not change the verdict: nothing found
   so far suggests the configuration-derived contract is wrong, only that it remains to be checked
   against real data when available.

This GO does not cover performance at scale (CP-15), the live collaboration data layer (CP-16
onward), or anything Snowflake-dependent (CP-08 to CP-11, still pending live credentials) — it
covers specifically whether HTML, as a format, survives the Lexical and Python round trip intact
and safely. On that narrow but foundational question: yes.
