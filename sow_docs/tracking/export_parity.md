# Export and quality parity: Lexical output vs CKEditor-era output (CP-65)

Date: 2026-11-12. Method: the Lexical sample (`blend_ode_be/tests/fixtures/lexical_html/lexical_sample.html`) is produced by the
Owner's Lexical pipeline; the 23 CKEditor-era fixtures are in `blend_ode_be/tests/fixtures/sow_module_html/html`. All run
through `sow_docx_export`, `sow_pdf_export` and `draft_quality` (test_sow_module_export_parity.py: 26 passing).

## Differences found

| # | Difference | Where | Status |
|---|---|---|---|
| 1 | A nested list was written as an empty bullet followed by the sub-list. | Owner renderer (`yjs-html.ts`) | **Fixed in our serializer** (attachNestedLists). Test: yjs-html.lists.test.ts. |
| 2 | Table cell text is wrapped in `<p>` (`<td><p>Role</p></td>`). | Owner renderer | Approved: exporters read the text correctly (test: Word output contains the cell text). |
| 3 | Top title written as `<h1>`; heading ids are not written. | Owner renderer | Approved: heading ids are not in the allowlist, and navigation uses text matching (as in CKEditor). |
| 4 | Text alignment kept as an inline style. | Owner renderer | Same as CKEditor. Checked in the fixture; the exporter output was not compared pixel by pixel. |
| 5 | Placeholder marker class kept. | Owner renderer | Same as CKEditor. Test: the placeholder text is in the Word output. |
| 6 | Merged header kept (`colspan`). | Owner renderer | Same as CKEditor. Test: Word output has the gridSpan. |

## Not yet compared (open)
- Font families and highlight colours in the exported files: the sample has no font-family spans, so not checked.
- The quality, quote-grounding, open-items and source-coverage checks were run on the sample for errors and text only. They were not compared with the CKEditor outputs for the same content, because no pair of identical content exists yet.
- The live download button calls the same export handler as CKEditor (chrome.test.tsx checks the call). A real download in a browser is not checked.

## What to do next
- Owner: export a few real drafts from the live editor and the same drafts from CKEditor, and send them for the comparison (see TODO_FOR_YOU).
