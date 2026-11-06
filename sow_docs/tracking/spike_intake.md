# Paste and LLM-text intake spike (CP-14)

**Status: DONE, fully run.** No Snowflake access needed — pure frontend/Node logic.

## Samples

10 synthetic LLM-draft-output samples and 5 synthetic Word-paste samples were requested from the
user; none were available (same situation as CP-12's real draft samples). Synthesized instead,
consistent with that precedent:

- `shared/fixtures/intake/llm/synthetic-llm-01..10-*.html` — modeled on realistic LLM imperfections
  found across the audited agent paths: a leftover unrendered `**bold**` markdown marker inside
  HTML output, MSO-style empty spacer paragraphs, a complete `suggested_fix_agent`-shaped draft with
  a billing table, doubled `<br><br>` line breaks used instead of paragraph breaks, inconsistent
  heading levels, a leaked markdown bullet list rendered as plain dashes, an unclosed `<strong>` tag,
  a leaked code fence, an unresolved `[AMOUNT]`/`[TBD]` placeholder pattern (continued-draft style),
  and mixed-case tags (`<H2>`, `<B>`, `<DIV>`).
- `shared/fixtures/intake/word-paste/synthetic-word-01..05-*.html` — modeled on real Microsoft Word
  HTML-paste structure: `mso-*` inline styles, `MsoNormal`/`MsoListParagraph`/`MsoTableGrid` classes,
  `<o:p>` Office-namespaced tags, `<!--[if gte mso 9]>` conditional comments, deeply nested spans for
  a single bold run, an embedded `<img>` with a `file:///` source, and an embedded `<script>`.

## `shared/intake.ts`

`intake({format: 'html' | 'markdown' | 'text', content})` — one function, one allowlist. All three
formats funnel into the exact same `sanitizeHtml` (CP-13's function, with backend parity already
proven):

- **markdown** → `markdown-it` with **`html: false`** — raw HTML in the markdown source is escaped
  to inert text by the renderer itself, a layer of defense before `sanitizeHtml` ever runs, not a
  substitute for it.
- **text** → split on blank lines into paragraphs, assigned via `textContent` (the DOM does the
  escaping; no hand-rolled escape function).
- **html** → passed through directly.

After sanitizing, `collapseBlankParagraphs` removes any `<p>` with no element children and only
whitespace/`&nbsp;` text. A removal report (`{kind: 'tag'|'class'|'blank-paragraph', value, count}`)
is computed by comparing element/class counts before and after.

`fast-check@3.23.2` was used, not the current 4.x — the 4.x release ships as pure ESM (a rolldown
bundle with top-level `import`), which Jest's default CRA transform can't parse without ejecting or
adding a custom transform; 3.23.2 is the newest version with a CommonJS entry point, and is only a
test-time dependency, not shipped to users. (`fc.assert` calls use a fixed `seed` for the same
reason test-suite determinism matters generally: a fuzz test that can fail on an unlucky seed and
pass on the next run is not trustworthy — see "false positives" below for why one seed still did
reveal something real on its first run.)

## Real defects found (and fixed) by this spike

Testing against the Word-paste fixtures — not hypothetical input — surfaced two real gaps in
`sanitizeImportHtml`/`sanitizeExportedHtml` (CP-12/13's functions), both now fixed in
`shared/allowlist.ts`:

1. **An entirely unrecognized tag survived untouched.** The unwrap pass only unwrapped tags
   explicitly listed in `html-contract.json`'s `importUnwrap` (`figure`, `div`, `section`, `font`,
   `a`). Office's non-standard `<o:p>` tag matches none of those and isn't an allowed tag either, so
   it fell through both checks and was never touched. **Fixed**: the unwrap pass now unwraps *any*
   tag not on the allowed-tags list, which subsumes the explicit list (every one of those five tags
   is also "not in `contract.tags`") and additionally catches genuinely unexpected input like
   `<o:p>` that no one anticipated when the contract was written.
2. **Vendor-prefixed CSS properties survived.** `mso-bidi-font-weight: normal` inside a `<span
   style="...">` was not stripped, because the code relied on the DOM's own `CSSStyleDeclaration`
   (`el.style`) to enumerate properties, and jsdom simply doesn't recognize Microsoft's
   non-standard property name — so it never appeared in the iteration the per-property removal loop
   was built on. This is the same root-cause *pattern* as the `javascript:`-in-`url()` defect CP-13
   found (an unrecognized value makes the browser's CSS parser silently omit it from `el.style`,
   which then makes anything built on `el.style` blind to it), just triggered by a different kind of
   unrecognized value. **Fixed** by replacing the DOM-based style filtering entirely with a
   hand-written regex parser (`filterStyleAttribute`) — the same approach the Python side
   (`html_allowlist.py`) already used, which is why the Python side never had either of these two
   bugs in the first place. One fix now serves both: the `javascript:`/`expression(` safety net and
   the unrecognized-property gap are both closed by no longer trusting the DOM's CSS engine to tell
   us what's in an attribute we can just parse ourselves.

Both fixes are covered by the Word-paste fixture tests and verified not to regress CP-12's 23
fixtures or CP-13's 81 parity/hostile tests (all still pass).

## A test false-positive, caught before it hid a real signal

The first fast-check run (before pinning a seed) found an input where
`expect(result.html).not.toMatch(/=\s*["']?\s*javascript:/i)` failed — the optional quote in that
regex meant it could match plain, already-escaped text content that merely happened to contain the
literal characters `=javascript:` with no quote at all (the same class of lesson CP-13 learned with
an HTML-entity-escaped `&lt;img onerror=...&gt;` string). Tightened to require a quote
(`=\s*["']\s*javascript:`) in both `intake.test.ts` and the Python parity test
(`test_sow_module_html_parity.py`), which had the same loose pattern. After the fix, 5 consecutive
full test runs and a fixed fast-check seed all pass with no flakiness.

## Results

- **Word paste reduced to allowed formatting**: confirmed on all 5 fixtures — no `mso-*` style or
  class, no `<o:p>`, no `<script>`, no `<img>` survives; visible text is preserved
  (`synthetic-word-01`'s "Client responsibilities include timely sign-off." checked explicitly).
- **Blank paragraphs collapse**: confirmed, with an exact count in the removal report.
- **Removal report**: confirmed to list dropped tags (e.g. `div`) and classes (e.g. `MsoNormal`).
- **Hostile content neutralized**: a `<script>` and a markdown-syntax `javascript:` link embedded in
  markdown are both neutralized by `html: false` plus `sanitizeHtml`; fuzzing (400 random inputs
  total, 200 per format, fixed seed) found no surviving `<script>`, `on*=` attribute, or
  `javascript:` URL after the two fixes above.
- **LLM-output quirks**: all 10 fixtures sanitize cleanly; none of the simulated imperfections
  (leaked markdown syntax, unclosed tags, mixed-case tags, doubled breaks) caused an exception or
  let disallowed markup through. Leaked markdown syntax (`**bold**`, a leaked ` ```html ` fence, bare
  `-` bullets) is **not** converted to real formatting by the `html` intake path — that's expected:
  `format: 'html'` does not run markdown-it. Content arriving as already-rendered HTML with leaked
  markdown syntax inside it is a content-quality issue for the agent prompts to avoid (tracked as an
  open item below), not a security or allowlist issue.
- **Timing**: a synthetic 20-page paste (roughly 500 words/page, Word-paste-shaped) completed intake
  in well under 100ms in Jest/jsdom on this machine. Per the prompt's own instruction, this number is
  explicitly **not from the reference workstation** and should not be treated as a production timing
  guarantee — only as evidence that nothing in the pipeline is pathologically slow (consistent with
  the depth-bomb and giant-table results from CP-13).

## Open items (not resolved by this checkpoint)

1. If leaked markdown syntax inside LLM-generated *HTML* output turns out to be a real, recurring
   problem (not just a hypothetical fixture), the fix belongs in the agent prompts
   (`suggested_fix_agent.py`, the narrative template renderer) — "do not use markdown or
   placeholders" is already the narrative agent's instruction per the CP-04 audit; `intake.ts`
   deliberately does not try to detect and re-render markdown found inside an `html`-format input,
   since guessing at malformed input is worse than clearly flagging it upstream.
2. Real Word-paste and real LLM-output samples are still needed to validate anything beyond what was
   anticipated here — tracked in `SOW_project/testing_pending_guide.md`.
3. Timing was measured in Jest/jsdom on the current development machine, not the reference
   workstation named in the prompt. A real timing budget needs a real measurement there.
