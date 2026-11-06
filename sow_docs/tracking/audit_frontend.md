# Frontend audit — Part 1: DraftEditor props and CKEditor parity

Audited file: `blend_ode/src/pricing-tool/components/DraftEditor.tsx` (754 lines)
Test file: `blend_ode/src/pricing-tool/components/__tests__/DraftEditor.test.tsx` (425 lines)
Date: 2026-10-30 | Checkpoint: CP-01

---

## 1  Props contract

| Prop | Type | Default | Purpose | Lines |
|---|---|---|---|---|
| `content` | `string` | (required) | Draft HTML, system of record | 213 |
| `addInNavigation` | `{ id: string; requestId: number }` | `undefined` | Scroll-to target after add-in insertion | 214 |
| `hasPendingSignOffs` | `boolean` | (required) | Show/hide DRAFT watermark | 216 |
| `isProvisionalTemplate` | `boolean` | (required) | Show/hide provisional-pricing banner | 218 |
| `comments` | `SerializedDraftComments \| null` | `null` | Opaque comment blob passed through to `onSave` | 220 |
| `onSave` | `(content: string, comments: …) => Promise<void>` | (required) | Persist content + comments | 222 |
| `onContentChange` | `(content: string) => void` | `undefined` | Fires on every author edit (after placeholder strip) | 224 |
| `onCommentsChange` | `(comments: …) => void` | `undefined` | Fires when comment state changes (currently unused) | 226 |
| `onCommentChangeHandlerReady` | `(handler) => void` | `undefined` | Registers external comment-change handler | 228 |
| `onDirtyChange` | `(isDirty: boolean) => void` | `undefined` | Standalone dirty-state callback | 230 |
| `dirtyState` | `boolean` | `undefined` | When supplied, parent owns dirty display | 236 |
| `isSaving` | `boolean` | `false` | External saving indicator (for autosave) | 239 |
| `onDownload` | `(format: 'doc' \| 'pdf') => void` | `undefined` | Download button handler | 241 |
| `onCopyLink` | `() => boolean \| Promise<boolean>` | `undefined` | Copy-link button handler | 243 |
| `isLoading` | `boolean` | `false` | Show spinner while draft loads | 245 |
| `error` | `string` | `undefined` | Error message for failed load | 247 |
| `onRetry` | `() => void` | `undefined` | Retry callback for error state | 249 |
| `readOnly` | `boolean` | `false` | Lock editor for historical versions | 253 |
| `readOnlyLabel` | `string` | `'Read-only — viewing a prior version'` | Text shown in toolbar when locked | 254 |
| `documentId` | `string` | `undefined` | SOW id for optional Cloud Services (unused in GPL build) | 256 |

`SerializedDraftComments` = `Record<string, unknown>` (sowStudioServices.tsx:99).

---

## 2  CKEditor plugin list (lines 84–103)

| # | Plugin | Toolbar item(s) | Notes |
|---|---|---|---|
| 1 | `Essentials` | undo, redo | Core editing (clipboard, enter, undo) |
| 2 | `Paragraph` | — | Block-level paragraphs |
| 3 | `Heading` | heading | h1–h6 |
| 4 | `Bold` | bold | `<strong>` |
| 5 | `Italic` | italic | `<i>` / `<em>` |
| 6 | `Underline` | underline | `<u>` |
| 7 | `Strikethrough` | strikethrough | `<s>` |
| 8 | `Highlight` | highlight | `<mark class="…">` (7 options, see §3) |
| 9 | `FontSize` | fontSize | 7 options (see §3) |
| 10 | `FontFamily` | fontFamily | 9 options (see §3) |
| 11 | `Alignment` | alignment | left / center / right / justify |
| 12 | `List` | bulletedList, numberedList | `<ul>` / `<ol>` |
| 13 | `Table` | — (via contentToolbar) | `<table>` |
| 14 | `TableToolbar` | tableColumn, tableRow, mergeTableCells | Context toolbar in table |
| 15 | `TableProperties` | tableProperties | Border, background, dimensions |
| 16 | `TableCellProperties` | tableCellProperties | Per-cell styling |
| 17 | `TableColumnResize` | — (drag handles) | Column width |
| 18 | `FindAndReplace` | findAndReplace | Ctrl+F search |

Toolbar order (line 105–123): `bold | italic | underline | strikethrough | highlight | findAndReplace | — | heading | fontSize | fontFamily | alignment | — | bulletedList | numberedList | — | undo | redo`

`shouldNotGroupWhenFull: true` — toolbar never collapses into a hamburger.

**Not present** (confirmed by test "does not configure trial-only CKEditor features", test line 157–167): comment, commentsArchive, tableOfContents, balloonToolbar, cloudServices, collaboration.

---

## 3  Format options

### Font sizes (lines 126–129)
`[10, 12, 14, 'default', 18, 24, 32]` — `supportAllValues: false`.
CKEditor HTML: `<span style="font-size:10px">` etc.; `'default'` produces no inline style.

### Font families (lines 130–142)
| # | Option value |
|---|---|
| 1 | `default` |
| 2 | `Arial, Helvetica, sans-serif` |
| 3 | `Courier New, Courier, monospace` |
| 4 | `Georgia, serif` |
| 5 | `Lucida Sans Unicode, Lucida Grande, sans-serif` |
| 6 | `Tahoma, Geneva, sans-serif` |
| 7 | `Times New Roman, Times, serif` |
| 8 | `Trebuchet MS, Helvetica, sans-serif` |
| 9 | `Verdana, Geneva, sans-serif` |

`supportAllValues: true` — preserves arbitrary fonts pasted from external sources.
CKEditor HTML: `<span style="font-family:'Georgia', serif">`.

### Highlight options (lines 153–208)
| # | Model name | CSS class | Type | Title |
|---|---|---|---|---|
| 1 | `yellowMarker` | `marker-yellow` | marker | Yellow marker |
| 2 | `greenMarker` | `marker-green` | marker | Green marker |
| 3 | `pinkMarker` | `marker-pink` | marker | Pink marker |
| 4 | `blueMarker` | `marker-blue` | marker | Blue marker |
| 5 | `sowPlaceholder` | `sow-studio-placeholder-marker` | marker | Pending SOW field |
| 6 | `redPen` | `pen-red` | pen | Red pen |
| 7 | `greenPen` | `pen-green` | pen | Green pen |

All produce `<mark class="…">`. Pens change text color; markers change background.

### Table content toolbar (lines 144–152)
`tableColumn | tableRow | mergeTableCells | tableProperties | tableCellProperties`

---

## 4  SOW-specific behaviors

### 4.1  Placeholder auto-unwrap (lines 42–80)
Two functions work together:
- **`clearCompletedPlaceholderMarkers(content)`** (HTML level, line 42): parses HTML with DOMParser, finds every `<mark class="sow-studio-placeholder-marker">`, keeps it while text still contains `[` or `]`, otherwise replaces the `<mark>` with its children. Returns changed HTML or the original.
- **`clearCompletedPlaceholderHighlights(editor)`** (CKEditor model level, line 62): walks the model tree, finds `$textProxy` nodes with `highlight === 'sowPlaceholder'`, removes the attribute if no brackets remain. Preserves cursor position.

Called from `handleEditorChange` (line 373): the HTML-level function runs first; if it detects changes, the model-level function runs to avoid a full `data.set` reset.

**Test coverage** (test lines 87–122): two tests — one verifies bracket removal triggers unwrap; one verifies a single remaining bracket keeps the marker.

### 4.2  Watermark (lines 589–639)
A 3×3 grid of "DRAFT" labels, red (#cf1322), 7% opacity, rotated -33°. Rendered only when `hasPendingSignOffs === true`. CSS: absolute positioned, `pointer-events: none`, `user-select: none`. Clipped to the editable surface bounds via a ResizeObserver + scroll listener (lines 430–484).

**Test coverage** (test lines 170–189): watermark appears/disappears with the prop.

### 4.3  Blend logo (lines 655–657)
`<img src={blendLogo} alt="Blend" />` in `div.sow-studio-draft-document-logo`. Positioned top-right of the editable area (line 447). Image: `src/utils/Blue.png`.

**Test coverage** (test line 84): checks `img[name="Blend"]` is present.

### 4.4  Provisional-template banner (lines 559–575)
Yellow-bordered box (`#f5c242` border, `#fff7db` background, `#8a4b08` text) with copy: "This pricing isn't locked as an ODE in the Pricing App, so it may not be the final version. Amounts, resources, and pricing dates can differ from the dates entered in Intake." Rendered only when `isProvisionalTemplate === true`.

**Test coverage** (test lines 192–211): banner appears/disappears with the prop.

### 4.5  Read-only mode (lines 417–425, 646–654)
Uses CKEditor's `enableReadOnlyMode(READ_ONLY_LOCK_ID)` where `READ_ONLY_LOCK_ID = 'sow-studio-historical-version'` (line 36). A label is portaled into the toolbar: default text `"Read-only — viewing a prior version"`, overridable via `readOnlyLabel`. Shell gets `sow-studio-ckeditor-shell--read-only` class.

### 4.6  Save button (lines 487–505, 705–711)
Antd `Button` with `SaveOutlined` icon. Disabled when: `readOnly || !displayedDirtyState || isSaveInProgress || editorError`. Calls `onSave(editorContentRef.current, comments)`. On success: clears dirty state, shows `message.success`. On failure: shows `message.error` with detail.

**Test coverage** (test lines 214–360): disabled initially, enabled after edit, calls onSave with content+comments, disabled again after successful save, disabled when parent is autosaving.

### 4.7  Download button (lines 507–516, 713–729)
Antd `Dropdown` with two items: `Download DOCX` (key `'doc'`) and `Download PDF` (key `'pdf'`). Calls `onDownload(key)`.

**Test coverage** (test lines 363–372): DOCX option fires `onDownload('doc')`.

### 4.8  Copy link button (lines 519–528, 730–739)
Antd text `Button` with `LinkOutlined`. Calls `onCopyLink()`, shows success/failure message.

**Test coverage** (test lines 375–382): click fires `onCopyLink`.

### 4.9  Unsaved-changes indicator (lines 743–747)
`⚠ Unsaved changes` span, `color: #faad14`, 12px, font-weight 500. Visible when `displayedDirtyState` is truthy.

**Test coverage** (test lines 416–424): appears after edit.

### 4.10  Add-in navigation (lines 357–370)
Uses `scrollToInsertedAddIn(editable, content, addInNavigation.id)` from `addInNavigation.ts`. Finds blocks by matching `data-add-in-id` in the source HTML to rendered blocks by text content. Applies `sow-studio-add-in-inserted` class with a 3.5s timeout, then scrolls smoothly. Consumed once per `addInNavigation` reference.

Source: `blend_ode/src/pricing-tool/sow-studio/steps/draft/addInNavigation.ts` (45 lines).

### 4.11  Programmatic content application (lines 305–345)
`applyProgrammaticContent` calls `editor.data.set()` with a guard flag (`isApplyingProgrammaticContentRef`) so programmatic updates don't trigger dirty state. Optionally preserves selection path.

### 4.12  Sticky toolbar override (lines 663–669)
Disables CKEditor's built-in `stickyPanel` so the workspace's own scroll governs toolbar positioning.

---

## 5  CKEditor feature parity table

| # | CKEditor feature | HTML it produces | Lexical plan | Decision |
|---|---|---|---|---|
| 1 | Bold | `<strong>` | `FormatBoldNode` / built-in | built |
| 2 | Italic | `<em>` or `<i>` | `FormatItalicNode` / built-in | built |
| 3 | Underline | `<u>` | `FormatUnderlineNode` / built-in | built |
| 4 | Strikethrough | `<s>` | `FormatStrikethroughNode` / built-in | built |
| 5 | Heading (h1–h6) | `<h1>` … `<h6>` | `HeadingNode` / `@lexical/rich-text` | built |
| 6 | Paragraph | `<p>` | built-in | built |
| 7 | Alignment (L/C/R/J) | `style="text-align:…"` on block | custom plugin or `@lexical/rich-text` | built |
| 8 | Bulleted list | `<ul><li>` | `@lexical/list` | built |
| 9 | Numbered list | `<ol><li>` | `@lexical/list` | built |
| 10 | Font size (10/12/14/default/18/24/32) | `<span style="font-size:Xpx">` | Custom `FontSizeNode` or inline style | built |
| 11 | Font family (9 options) | `<span style="font-family:…">` | Custom `FontFamilyNode` or inline style | built |
| 12 | Highlight — yellowMarker | `<mark class="marker-yellow">` | Custom `HighlightNode` with class map | built |
| 13 | Highlight — greenMarker | `<mark class="marker-green">` | ″ | built |
| 14 | Highlight — pinkMarker | `<mark class="marker-pink">` | ″ | built |
| 15 | Highlight — blueMarker | `<mark class="marker-blue">` | ″ | built |
| 16 | Highlight — sowPlaceholder | `<mark class="sow-studio-placeholder-marker">` | ″ + auto-unwrap logic | built |
| 17 | Highlight — redPen | `<mark class="pen-red">` | ″ | built |
| 18 | Highlight — greenPen | `<mark class="pen-green">` | ″ | built |
| 19 | Table | `<table><tbody><tr><td>` | `@lexical/table` | built |
| 20 | Table column/row ops | toolbar buttons | `@lexical/table` context menu | built |
| 21 | Merge cells | toolbar button | `@lexical/table` merge/split | built |
| 22 | Table properties | border, bg, dimensions dialog | TBD — Lexical has no built-in equivalent | TBD |
| 23 | Table cell properties | per-cell border, bg, padding dialog | TBD — Lexical has no built-in equivalent | TBD |
| 24 | Table column resize | drag handles | `@lexical/table` (experimental columnResize) | TBD |
| 25 | Find and Replace | Ctrl+F dialog | Custom plugin | built |
| 26 | Undo / Redo | `@lexical/history` | built |
| 27 | Placeholder auto-unwrap | strips `<mark class="sow-studio-placeholder-marker">` when brackets gone | Port logic into Lexical transform or `onUpdate` | built |
| 28 | DRAFT watermark | pure React overlay | port as-is (no editor dependency) | built |
| 29 | Blend logo | pure React overlay | port as-is | built |
| 30 | Provisional banner | pure React | port as-is | built |
| 31 | Read-only mode | CKEditor lock API | Lexical `editable={false}` | built |
| 32 | Read-only label | portal into toolbar | Lexical toolbar conditional render | built |
| 33 | Save button + dirty state | React state | port as-is (no editor dependency) | built |
| 34 | Download (doc/pdf) | callback to parent | port as-is | built |
| 35 | Copy link | callback to parent | port as-is | built |
| 36 | Unsaved-changes indicator | React state | port as-is | built |
| 37 | Add-in navigation / scroll-to | DOM matching by text + `data-add-in-id` | Port: Lexical exposes editable DOM | built |
| 38 | Sticky toolbar override | disables CKEditor stickyPanel | Not needed — Lexical toolbar is custom React | built |
| 39 | `supportAllValues` for fonts | preserves pasted font-family values | TBD — decide if Lexical should preserve arbitrary pasted fonts or normalize | TBD |
| 40 | `shouldNotGroupWhenFull` | toolbar never collapses | Lexical toolbar is custom; always shown | built |
| 41 | Comments (legacy passthrough) | `comments` prop passed through to `onSave` | Carry forward as opaque blob (CP-55 replaces) | built |

---

## 6  Test file summary

File: `blend_ode/src/pricing-tool/components/__tests__/DraftEditor.test.tsx`

### Mock strategy
- **CKEditor mock** (lines 7–23): replaces `<CKEditor>` with a `<textarea>` that serializes config into `data-*` attributes (`data-toolbar-items`, `data-table-toolbar-items`, `data-cloud-services`, `data-collaboration`, `data-comment-mention-feeds`, `data-balloon-toolbar`). The `onChange` handler calls `getData()` on a minimal stub.
- **ckeditor5 mock** (lines 25–45): every imported plugin is an empty object.

### Behaviors tested (19 active tests, 2 commented out)
| Test | What it protects |
|---|---|
| Render with content | Editor loads, value matches, logo present, no read-only label |
| Placeholder unwrap (completed) | `sow-studio-placeholder-marker` removed when brackets gone |
| Placeholder unwrap (partial) | Marker kept while a bracket remains |
| Toolbar config | All formatting items present |
| Table toolbar | `tableColumn`, `tableRow`, `mergeTableCells` present |
| No trial features | No comment/cloud/collaboration/balloonToolbar |
| Watermark shown | `hasPendingSignOffs=true` → DRAFT watermark |
| Watermark hidden | `hasPendingSignOffs=false` → no watermark |
| Banner shown | `isProvisionalTemplate=true` → banner text |
| Banner hidden | `isProvisionalTemplate=false` → no banner |
| Save disabled initially | No edits → Save button disabled |
| Save enabled after edit | Content change → Save enabled |
| Save stays enabled on re-render | Parent re-render with same content keeps Save active |
| Clean on comment sync | External comment update doesn't dirty |
| Save calls onSave | Content + comments forwarded correctly |
| Save with comments | Explicit comments object passed |
| Download DOCX | Dropdown fires `onDownload('doc')` |
| Copy link | Button fires `onCopyLink` |
| Loading spinner | `isLoading=true, content=""` → spinner |
| Autosave indicator | `isSaving=true` → button disabled + loading class |
| Error state | Error prop → alert + retry button |
| Unsaved changes | Edit → "⚠ Unsaved changes" visible |

Two commented-out tests (lines 270–322): local comment edit → dirty + onCommentsChange, and save with local comment edit. These tested a comments-plugin integration that is currently disabled.

---

## 7  TBD items requiring product decision

1. **Table properties dialog** (row 22): CKEditor provides a modal for table-wide border, background, dimensions. Lexical has no built-in equivalent. Options: build a custom dialog, accept the loss, or use a simpler context menu. **Decision needed.**
2. **Table cell properties dialog** (row 23): same situation as table properties but per-cell. **Decision needed.**
3. **Table column resize** (row 24): `@lexical/table` has experimental resize support. Need to confirm it is stable enough for production, or build custom drag handles. **Decision needed.**
4. **Arbitrary pasted fonts** (row 39): `supportAllValues: true` in CKEditor means any pasted `font-family` is preserved. Should Lexical normalize to the 9 allowed families, or preserve everything? **Decision needed.**

---

## 8  Items not found in code

- **CSS definitions for highlight classes** (`marker-yellow`, `marker-green`, etc.): CKEditor ships these in `ckeditor5/ckeditor5.css` (loaded at line 33). Lexical will need its own stylesheet defining these classes. The exact RGB values are CKEditor CSS variables — not found in this component's source.
- **`sow-studio-add-in-inserted` CSS class**: defined externally (likely in a stylesheet), applied by `addInNavigation.ts:34`. Not found inline.
- **`sow-studio-document-editor`, `sow-studio-ckeditor-shell`, `sow-studio-draft-document-logo`**: CSS class names used on wrapper divs; styles are in an external stylesheet. Not found in this file.

---
---

# Frontend audit — Part 2: DraftEditorScreen, save flow, swap point, services

Audited file: `blend_ode/src/pricing-tool/components/DraftEditorScreen.tsx` (1284 lines)
Integration test: `blend_ode/src/pricing-tool/components/__tests__/DraftEditorScreen.integration.test.tsx`
Date: 2026-10-30 | Checkpoint: CP-02

---

## 9  DraftEditorScreen props contract

| Prop | Type | Default | Purpose | Lines |
|---|---|---|---|---|
| `sowId` | `string` | (required) | SOW identifier (`sow-<deal_id>`) | 24 |
| `versionId` | `string` | `undefined` | When set, loads a prior version → read-only | 26 |
| `preview` | `HubspotDealPreview \| null` | `undefined` | Source-of-truth fields for risk flag evaluation | 28 |
| `documentOrigin` | `string \| null` | `undefined` | Passed to risk flag evaluation | 30 |
| `documentType` | `string \| null` | `undefined` | Passed to risk flag evaluation | 31 |
| `onBackToIntake` | `() => void` | (required) | Navigate back to Intake step | 34 |
| `onProceedToReview` | `() => void` | (required) | Navigate to Deal Desk review | 35 |
| `onNavigateToSection` | `(sectionId) => void` | `undefined` | Jump to an intake section | 36 |
| `onDirtyStateChange` | `(isDirty) => void` | `undefined` | Notify parent of unsaved state | 37 |
| `onVersionSaved` | `(version?) => void` | `undefined` | Notify parent after save succeeds | 38 |
| `onVersionContinued` | `(version) => void` | `undefined` | Notify parent after "Continue from version" | 39 |
| `onReturnToLatestVersion` | `() => void` | `undefined` | Clears versionId to go back to latest | 41 |

---

## 10  Save and autosave flow

```
                      ┌──────────────────────────────────┐
                      │   DraftEditorScreen state         │
                      │                                   │
                      │  draftContent         (string)    │
                      │  draftComments        (blob)      │
                      │  draftRevisionRef     (string)    │  ← set from DraftResponse.revision on load
                      │  editGenerationRef    (number)    │  ← incremented on every author edit
                      │  hasUnsavedChanges    (boolean)   │
                      │  saveInFlightRef      (boolean)   │  ← guards against double saves
                      │  latestDraftContentRef (string)   │  ← always current, used by autosave
                      │  latestDraftCommentsRef (blob)    │
                      └────────────┬─────────────────────┘
                                   │
         ┌─────────────────────────┼─────────────────────────┐
         │ Author edits            │ Autosave timer           │ Save button click
         ▼                         ▼                          ▼
  handleEditorContentChange   setInterval(15_000)         DraftEditor.onSave
  (line 802)                  (line 907)                  → handleSave (line 830)
  ↓                           ↓                           ↓
  editGenerationRef++         checks saveInFlightRef      same handleSave
  latestDraftContentRef ←     calls handleSave(           ↓
  hasUnsavedChanges = true      latestDraftContentRef,    normalizeAddInSignaturePlacement
  reconcile open items          latestDraftCommentsRef)   ↓
                                                          saveDraft(sowId, content, comments,
                                                            parsedVersion, sowName,
                                                            addedAddIns, openItems,
                                                            draftRevisionRef.current)
                                                          ↓
                                                    POST /pricing_tool/drafts/save?deal_id=<id>
                                                    body: { content, comments,
                                                            expected_version,
                                                            expected_revision,   ← optimistic lock
                                                            sow_name, added_add_ins, open_items }
                                                          ↓
                                                    SaveDraftResponse { version, revision,
                                                      generatedDate, content, comments,
                                                      addedAddIns, openItems }
                                                          ↓
                                                    draftRevisionRef ← result.revision
                                                    if (savedGeneration === editGenerationRef):
                                                      hasUnsavedChanges = false
                                                      draftContent ← result content
                                                    else:
                                                      "Earlier edits saved. New changes still unsaved."
```

### Key details

1. **Optimistic revision lock** (lines 838–853): `handleSave` captures `editGenerationRef` before the async call. The `saveDraft` API sends `expected_revision` (the revision received from the last fetch or save). The backend can reject if the revision has changed (concurrent writer). On success, `draftRevisionRef` is updated to `result.revision`.

2. **Autosave interval** (lines 907–924): 15-second `setInterval`, fires only when `hasUnsavedChanges && !isDraftReadOnly && !saveLoading && draft`. Skips if `saveInFlightRef.current` is already true. Reads content/comments from refs, not state, so typing doesn't reset the timer.

3. **Generation tracking** (lines 838, 862, 875): `editGenerationRef` increments on every author edit. The save captures it before the request. If it still matches after the response, the save was current and dirty state clears. Otherwise, the save was stale and the user is told new changes remain unsaved.

4. **Save-and-navigate** (lines 954–976): "Proceed to review" with unsaved changes calls `handleSave` first, then `navigateToTarget('review')`. "Back to intake" with unsaved changes shows a modal with Keep editing / Discard / Save and continue.

5. **Continue from version** (lines 478–497): `continueDraft(sowId, sourceVersion, sourceRevision, expectedVersion, expectedRevision)` → `POST /pricing_tool/drafts/versions/<source>/continue`. Creates a new version from a historical one, with revision checks on both source and current.

---

## 11  Historical version → read-only

- `isViewingHistoricalVersion = Boolean(versionId)` (line 145)
- `isFinalizedDraft = draft?.status === 'Finalized'` (line 200)
- `isDraftReadOnly = isViewingHistoricalVersion || isFinalizedDraft` (line 201)
- When `isDraftReadOnly`:
  - `handleSave` returns false immediately (line 831)
  - `handleEditorContentChange` returns immediately (line 803)
  - Autosave interval doesn't start (line 908)
  - `DraftEditor` receives `readOnly={isDraftReadOnly}` (line 1103)
  - "Proceed to review" button is disabled (line 1244)
  - Add-in insert is suppressed (line 728, 1211)
  - A warning alert shows with "Continue from this version" and "Back to current draft" buttons (lines 1069–1094)
- Read-only label for finalized: `'Read-only — finalized SOW'` (line 1104)

---

## 12  Download and Copy link

- **Download** (lines 791–800): `handleDownload(format)` calls `downloadDraft(draft.downloadUrl, { dealId, content, comments, fileName, format, hasPendingSignOffs })`. The content is passed to the backend for server-side rendering. Formats: `'doc'` and `'pdf'`.
- **Copy link** (line 1122): `copyShareLink(window.location.href || draft.shareUrl)` — copies the current browser URL.

---

## 13  The swap point

### Where DraftEditorScreen renders DraftEditor

File: `blend_ode/src/pricing-tool/components/DraftEditorScreen.tsx`
Lines: **1097–1159**

```tsx
<DraftEditor
  documentId={`sow:${sowId}:draft`}
  addInNavigation={addInNavigation}
  content={draftContent || draft?.content || '<h2>Statement of Work</h2>...'}
  hasPendingSignOffs={draft?.hasPendingSignOffs !== undefined ? draft.hasPendingSignOffs : true}
  isProvisionalTemplate={false}
  readOnly={isDraftReadOnly}
  readOnlyLabel={isFinalizedDraft ? 'Read-only — finalized SOW' : undefined}
  comments={draftComments}
  onContentChange={handleEditorContentChange}
  onCommentsChange={...}
  dirtyState={hasUnsavedChanges}
  isSaving={isSavingDraft}
  onSave={async (content, comments) => { await handleSave(content, comments); }}
  onDownload={handleDownload}
  onCopyLink={...}
  isLoading={draftLoading}
  error={draftError || undefined}
  onRetry={...}
/>
```

This is the **single point** where the editor component is instantiated. The Lexical editor (`LexicalDraftEditor`) will replace `<DraftEditor>` here, matching the same props interface. The surrounding `DraftEditorScreen` code (save, autosave, open items, add-in insertion, navigation) stays unchanged.

### Where SowStudioPage renders DraftEditorScreen

File: `blend_ode/src/pricing-tool/sow-studio/pages/SowStudioPage.tsx`
Lines: **2687–2710**

```tsx
return sowId ? (
  <DraftEditorScreen
    sowId={sowId}
    versionId={viewingVersionId || undefined}
    preview={preview}
    ...
  />
) : (...);
```

`DraftEditorScreen` is rendered **unconditionally** — no feature flag gates it. `NativeDraftEditorScreen` is exported from `components/index.ts` (line 8) but **not imported or used** in `SowStudioPage.tsx`.

### Feature flag system

- Flags come from `localStorage.getItem("user_details").features` (array of strings)
- Read in `AppLayout.tsx:69`: `JSON.parse(localStorage.getItem("user_details") || "{}").features || []`
- `FE_SOW_NATIVE_EDITOR` appears only in `SowStudioPage.test.tsx:715` — a test, not application code
- `FE_SOW_LEXICAL_EDITOR` does not exist in the frontend yet (planned for CP-19)

---

## 14  Services that read or write draft HTML

| Service file | Reads HTML | Writes/modifies HTML | Must work with Lexical |
|---|---|---|---|
| `sowStudioAddInInsertion.ts` | Yes (DOMParser, heading parsing) | Yes (inserts add-in sections, renumbers subsections, updates cross-references) | **Yes** — insertion operates on HTML string, not CKEditor model. Works with any editor that produces valid HTML. |
| `sowStudioAddInPresence.ts` | Yes (DOMParser, scans for `data-add-in-id`, heading text, document text) | No | **Yes** — read-only check on HTML string. |
| `sowStudioIntakeDraftSync.ts` | Yes (DOMParser, finds signature tables, Name/Title fields) | Yes (rewrites Name/Title values in signature table) | **Yes** — operates on HTML string. |
| `sowStudioOpenItems.ts` | Yes (DOMParser, scans for signature fields, team table, fee paragraph, termination clauses) | No | **Yes** — read-only reconciliation on HTML string. |
| `addInNavigation.ts` | Yes (DOMParser for source matching; live DOM for scroll) | Yes (adds/removes CSS class `sow-studio-add-in-inserted`) | **Partial** — DOM selectors like `[data-add-in-id]` and `h1,h2,…,table,figure` work on any editable. But CSS class names (`sow-studio-add-in-inserted`) need to be styled in the Lexical theme. |
| `openItemNavigation.ts` (imported in DraftEditorScreen) | Yes (live DOM, `.ck-editor__editable`) | Yes (adds CSS classes for highlight) | **Must update** — uses `.ck-editor__editable` selector. Needs a parallel selector for Lexical's editable root, or a shared class. |
| `focusPendingPlaceholder` (DraftEditorScreen:602–682) | Yes (live DOM, `.ck-editor__editable, .ck-editor__editable_inline`) | Yes (CSS classes, selection) | **Must update** — CKEditor-specific selectors. |

---

## 15  Risks

1. **Two writers saving the same revision** — The `expected_revision` optimistic lock protects against concurrent saves from different tabs or users. But the frontend has no retry or merge strategy: if the backend rejects a stale revision, the save fails with an error message. The user must reload and re-apply their changes. The Lexical collab service (Yjs) will solve this for concurrent editing, but during the transition period (feature-flagged swap), the same revision-based optimistic lock applies.

2. **CKEditor-specific DOM selectors** — `focusPendingPlaceholder` (line 602) and `openItemNavigation.ts` query `.ck-editor__editable` and `.ck-editor__editable_inline`. These will return nothing when Lexical is the active editor. Fix: add a shared CSS class (e.g., `sow-draft-editable`) to both CKEditor's and Lexical's editable root, or branch on the active editor type.

3. **`data-add-in-id` stripping** — CKEditor strips `data-*` attributes during serialization (confirmed in the integration test mock, line 41). `sowStudioAddInPresence.ts` already handles this with a text-matching fallback. Lexical's HTML serializer must either preserve `data-add-in-id` or the same fallback continues to work.

4. **`normalizeAddInSignaturePlacement`** — Called on every load and save (lines 421, 842). It rewrites HTML with DOMParser. The rewritten HTML must round-trip through Lexical's parser without losing structure. If Lexical's HTML import doesn't handle certain table structures or `data-*` attributes, add-in placement could drift.

5. **Autosave reads from refs, not state** — The 15-second autosave reads `latestDraftContentRef` and `latestDraftCommentsRef`. If the Lexical editor's `onContentChange` callback is slow or batched differently than CKEditor's, there's a window where the ref holds stale content. The callback must update the ref synchronously.

6. **`ensureSectionAnchors` creates `<mark>` elements** — Line 345–346: it wraps bracketed tokens in `<mark class="sow-studio-placeholder-marker sow-studio-manual-placeholder">`. Lexical's HTML import must map this to the correct Highlight node, or placeholders won't be visible.

7. **`renumberDraftSubsections` modifies HTML headings** — Called from `handleEditorContentChange` (line 804). It parses and modifies the HTML string returned by the editor. If Lexical returns HTML that differs structurally from CKEditor (e.g., different whitespace, different tag nesting), the heading-number regex could fail to match.

8. **Download sends raw HTML to backend** — `handleDownload` passes `draftContent` to the backend for DOCX/PDF rendering. The backend renderer must handle Lexical's HTML output (which may differ from CKEditor's in class names, inline styles, or table structure).
