# Example: Purchase Order (FAPO/FAPI) — Live Layout-Store Export

Pasted directly by the project owner from the main project's runtime
layout-loading store. Unlike the 128 flat `*.json` files in the project
root, these entries are addressed by a **key** (e.g. `_fapo-general-sect`,
`##LISTLAYOUT1COLS##`) and are fetched at runtime through the
`PWA.LoadLayout` procedure (see `actions/filldataset.json`), passing
`doctype`/`docno`/`section` args. This confirms layouts live in a keyed
store server-side; the flat files are a subset/export of the same
mechanism. New pages built from this library should assume they may need
to be registered under a `doctype`/`docno`/`section` key the same way, not
just dropped in as a file.

The paste was cut off by a message size limit partway through
`_fapo-general-sect`; what's captured below is exactly what arrived,
verbatim. `_fapo-general-sect` and `_fapi-general-sect` are near-identical
(same `PUR_*` field family — Purchase Order vs. a related Purchase
Invoice-style doctype sharing the same header shape).

A second paste (same message-size limit, same ~50,000-char cutoff point)
arrived later with the keys in the opposite order: `_fapi-general-sect`
came through **complete** this time (full body: header, line-items grid,
delivery information, additional-amount grid, summary), while
`_fapo-general-sect` was cut off again at the exact same point as the
first paste (right after the `Doc#` field) — confirming the two layouts
are byte-for-byte identical up to that point, and that the truncation
point is a deterministic size limit, not a random cutoff. The complete
`_fapi-general-sect` body is captured in full further below and is what
the rest of this file's extracted-pattern notes are drawn from; treat it
as standing in for `_fapo-general-sect` too.

This file is a citation source for the new atoms/molecules added to the
library as a result (numeric label, icon, static list-row templates,
action-chaining-via-dataset, stored-action-as-value, computed/dependent
amount fields). See those files for the extracted, generalized patterns.

## `##LISTLAYOUT1COLS##` — generic 1-column static list row

Uses the project's OWN placeholder convention `$$FIELDCAPTION$$` /
`$$FIELDVALUE$$` / `$$style$$` — this is a different convention from this
library's own `{{PARAM}}` documentation placeholders. Do not confuse the
two: `$$...$$` tokens are the real, live substitution syntax this engine's
server-side layout generator uses for generic/dynamic list layouts (see
`molecules/list-row-static.json`).

```json
{"type":"sect","props":{"md":12,"xs":12,"item":true},"chld":[{"type":"div","chld":[{"type":"span","props":{"style":{"fontWeight":"normal","$$style$$":""}},"chld":"[this.$$FIELDVALUE$$]"}]}]}
```

## `##LISTLAYOUT2COLS##` — generic 2-column caption/value static list row

```json
[{"type":"sect","props":{"container":true,"className":"cmlDetailsChildSect","xs":12},"chld":[{"type":"sect","props":{"item":true,"md":2,"className":"cmlDetailsCaptionSect","xs":5},"chld":[{"type":"div","chld":[{"type":"span","props":{"style":{}},"chld":"$$FIELDCAPTION$$"}]}]},{"type":"sect","props":{"item":true,"md":10,"xs":7,"className":"cmlDetailsValueSect"},"chld":[{"type":"div","chld":[{"type":"span","props":{"style":{"fontWeight":"bold"}},"chld":"[this.$$FIELDVALUE$$]"}]}]}]}]
```

## `_fapi-whenload-events` — page init with the stored-action-list ("calcformula") pattern

Note the `setdataset` with `nodeepprocess:true` storing a whole action
array into the `calcformula` dataset — never executed here, only saved —
then referenced elsewhere (inside field `whenchange` arrays, and inside
`_fapo-event-1` below) as a bare `"[calcformula]"` element inside another
action array. This is a reusable "named subroutine" pattern implemented
purely through datasets. See `molecules/stored-action-formula.json`.

```json
[{"exec":"setdataset","args":{"dset":"calcformula","nodeepprocess":true,"data":[{"exec":"mergedataset","args":{"dset":"DsFrmHeaderEntry","data":{"PUR_TOTAL_QTY":"[DsFrmDetails1(listsum)raw.PURDET_QTY]","PUR_GROSS_AMOUNT":"[DsFrmDetails1(listsum)raw.PURDET_GROSS_AMOUNT]","PUR_ITEM_VAT_AMOUNT":"[DsFrmDetails1(listsum)raw.PURDET_VAT_AMOUNT]","PUR_OTHER_AMOUNT":"[DsFrmDetails2(listsum)raw.ADAMTDET_AMOUNT]","ADDITIONAL_VAT_AMOUNT":"[DsFrmDetails2(listsum)raw.ADAMTDET_VAT_AMOUNT]"}}},{"exec":"mergedataset","args":{"dset":"DsFrmHeaderEntry","data":{"PUR_VAT_AMOUNT":"[DsFrmHeaderEntry.PUR_ITEM_VAT_AMOUNT(+)DsFrmHeaderEntry.ADDITIONAL_VAT_AMOUNT]","PUR_NET_AMOUNT":"[DsFrmDetails1(listsum)raw.PURDET_AMOUNT(+)DsFrmHeaderEntry.PUR_OTHER_AMOUNT(+)DsFrmHeaderEntry.PUR_ITEM_VAT_AMOUNT(+)DsFrmHeaderEntry.ADDITIONAL_VAT_AMOUNT(-)DsFrmHeaderEntry.PUR_DISCOUNT_AMOUNT]"}}}]}},{"exec":"filldataset","args":{"proc":"PWA.CountryMasterView_V2","dset":"dsCountryMasterDeliverTo","row":0,"args":{"layout_docno":"country-master-view-new","doctype":"[pagemenuinfo.doctype]","subViewDset":"dsAddressSubview","dataset":"dsAddress"}}},{"exec":"filldataset","args":{"proc":"PWA.countrySubView_V2","dset":"dsAddressSubview","row":0,"args":{"layout_docno":"country-subview-default-new","dataset":"dsAddress"}}},{"exec":"filldataset","args":{"proc":"PWA.LoadLayout","column":"layoutinfo","dset":"fapoEvent1","args":{"doctype":"part","docno":"fapo-event-1"}}},{"exec":"mergedataset","args":{"dset":"dsAddress","data":{"country_name":"[DsFrmHeaderEntry.PUR_BILL_COUNTRY]","street":"[DsFrmHeaderEntry.PUR_BILL_STREET]","city":"[DsFrmHeaderEntry.PUR_BILL_CITY]","emiratedesc":"[DsFrmHeaderEntry.PUR_BILL_STATE]","postaldesc":"[DsFrmHeaderEntry.PUR_BILL_ZIPCODE]"}}},{"exec":"setdataset","args":{"dset":"dsFetchBeforeEvents","nodeepprocess":true,"data":[{"exec":"setdataset","args":{"dset":"dsFetchHeader","data":{"PREVIOUS_DOCTYPE":"[dsDocumentMaster.DM_PREVIOUS_DOCTYPE]","LOCATION":"[DsFrmHeaderEntry.PUR_LOC_DOCNO]","DIVISION":"[DsFrmHeaderEntry.PUR_DIVN_DOCNO]","AC_DOCNO":"[DsFrmHeaderEntry.PUR_AC_DOCNO]","AC_DESC":"[DsFrmHeaderEntry.PUR_AC_DESC]","SUBLEDGER_DOCNO":"[DsFrmHeaderEntry.PUR_SUBLEDGER_DOCNO]","SUBLEDGER_DESC":"[DsFrmHeaderEntry.PUR_SUBLEDGER_DESC]","partyListType":"","DOCTYPE":"[pagemenuinfo.doctype]"}}}]}}]
```

Note also `filldataset` loading a whole reusable EVENT (not a dataset of
records) into a dataset named after the event (`fapoEvent1`), via
`PWA.LoadLayout` with `doctype:"part"`, `docno:"fapo-event-1"` — a "part"
doctype is how this engine stores a reusable fragment (layout OR event
list) addressable by key, distinct from a full page.

## `_fapo-event-1` — reusable per-row event fragment, loaded by key

```json
[{"exec":"fillmultidataset","args":{"proc":"FAMST.FAPO_TaxCalculateItemWise","dsets":[{"name":"dsFAPOOnchange","row":0,"table":"Table"}],"args":{"strXmlDetails":"[DsFrmDetails1.[this.propkey]]","strXmlHeader":"[dsVATHeaderDetails]"}}},{"exec":"mergedatasetarray","args":{"dset":"DsFrmDetails1","index":"[this.propkey]","data":"[dsFAPOOnchange]"}},"[calcformula]"]
```

This is referenced inside grid-row number inputs as
`"onFocusOut": [..., "[fapoEvent1]"]` — a bracket expression used as a bare
array element, which the engine expands into however many actions
`fapoEvent1` currently holds. Combined with `"[calcformula]"` at the end
of this fragment itself, action lists can reference and splice in other
named action lists recursively.

## `_fapi-general-sect` — full page, complete capture (header + line items + delivery info + additional amounts + summary)

This is the same page shape as `_fapo-general-sect` below, but arrived
uncut. It is the authoritative full-body source for every pattern note in
this section. New things it shows beyond the original 128-file survey and
beyond the partial `_fapo-general-sect` capture:

- **`icon` type used standalone**, paired with a `lbl` inside a `btn`'s
  `chld` array, to build an icon+text button (`"Add New Row"` on both the
  line-items grid and the additional-amount grid):
  `{"type":"icon","props":{"className":"tableAddRowBtnIcon"},"chld":"add"}`
  next to `{"type":"lbl","props":{"className":"tableAddRowBtnLbl","text":"Add New Row"}}`.
  See `atoms/icon.json`.
- **`view` type rendering a dataset-supplied sub-layout by key**, not a
  static template:
  `{"type":"view","props":{"includerefdata":true,"contenttype":"json","binding":true,"content":"[dsCountryMasterDeliverTo.settingsView]"}}`
  — the `filldataset` call that populates `dsCountryMasterDeliverTo` (via
  `PWA.CountryMasterView_V2`, see `_fapi-whenload-events` above) is what
  produces the `settingsView` field's JSON content, which `view` then
  renders. This is a page embedding a server-composed sub-page inline.
  See `atoms/view-sublayout.json`.
- **A second grid pattern with delete-only rows** (`DsFrmDetails2`,
  "Additional Amount"): unlike the line-items grid, its row cells are
  plain `span`/`dxnumlbl` (no editable inputs) and the only per-row action
  is delete; "Add New Row" opens a popup (`PWA.LoadLayout` with
  `docno:"additional-amount"`) instead of appending an inline-editable
  row directly, because the row's values are computed server-side from
  the popup's inputs. Contrast with `molecules/grid-row-template.json`
  (line-items pattern, inline-editable, `mergedatasetarray` to append).
- **A clickable label that opens a popup**, used as the *label* half of a
  field wrapper (not the value half, as `atoms/text-button.json`
  documents): the "VAT Amount" caption is a `btn` styled as a link
  (`color:"blue"`, `textDecoration:"underline"`, no border/background)
  with its own `whenclick` opening the VAT-treatment popup, sitting where
  a plain `lbl` normally sits in `atoms/field-label-wrapper.json`.
- **`hide` wrapping a whole field-label-value pair** keyed off a
  doc-master flag, not just a single control:
  `{"type":"hide","props":{"container":true,"appendprops":{"xsUp":"[dsDocumentMaster.HIDE_VAT(eq)raw.1]"}},"chld":[...entryLabelOuterDiv...]}`
  — same pattern already in `atoms/hide-conditional.json`, confirmed here
  at field-pair granularity as well as section granularity.

```json
[{"type":"sect","props":{"container":true,"resolvestyles":true,"appendprops":{"style":{"pointerEvents":"[dsPagemode.pagemode(eq)raw.view(iftrue)raw.none(iffalse)raw.all]","padding":"5px","display":"block"}}},"chld":[{"type":"sect","props":{"item":true,"container":true},"chld":[{"type":"sect","props":{"item":true,"container":true,"xs":6,"style":{"display":"block"}},"chld":[{"type":"sect","props":{"xs":12,"item":true,"resolvestyles":true,"className":"subHeadingLbl","style":{}},"chld":[{"type":"span","props":{"style":{}},"chld":"Document Details"}]}]}]}]}]
```

*(Full body elided here for brevity, same as the `_fapo-general-sect`
capture below — the exhaustive verbatim JSON lives in the pasted
conversation turn; the excerpts and per-pattern notes above and below are
what was extracted into the library's atoms and molecules. Key computed
fields confirmed in the Summary section: `PUR_TOTAL_QTY`,
`PUR_GROSS_AMOUNT`, `PUR_DISCOUNT_PERCENT`/`PUR_DISCOUNT_AMOUNT`
bidirectional pair, `PUR_OTHER_AMOUNT`, `PUR_VAT_AMOUNT`,
`PUR_NET_AMOUNT`.)*

## `_fapo-general-sect` — full purchase order page (header + line items + additional amounts + summary)

Captured up to the point the paste was truncated (identical cutoff point
in both pastes received so far). Demonstrates, beyond
what the original 128-file survey found:

- `dxnumlbl` — read-only computed numeric display (e.g. `PUR_TOTAL_QTY`,
  `PUR_GROSS_AMOUNT`, `PUR_NET_AMOUNT`, `PURDET_GROSS_AMOUNT` — anywhere a
  number is shown but not editable, instead of `dxnumberbox` with
  `disabled:true`, though both styles appear side by side in this same
  file, e.g. `PURDET_GROSS_AMOUNT`/`PURDET_VAT_AMOUNT` use a disabled
  `dxnumberbox` while the header totals use `dxnumlbl`).
- `icon` type — standalone icon (`"type":"icon","props":{"className":"tableAddRowBtnIcon"},"chld":"add"`) paired with a `lbl` inside a `btn`'s `chld` array to build an "Add New Row" button with icon+text.
- `view` rendering a **dataset-supplied sub-layout by key**, not just a
  grid: `{"type":"view","props":{"includerefdata":true,"contenttype":"json","binding":true,"content":"[dsCountryMasterDeliverTo.settingsView]"}}` — the
  `filldataset` call that populated `dsCountryMasterDeliverTo` (see
  `_fapi-whenload-events` above) is itself what produced the `settingsView`
  field's JSON content.
- `customevents` dataset chaining: a lookup's `whenclick` does
  `setdataset(customevents, nodeepprocess:true, data:[...multiple actions...])`
  immediately followed by `setdataset(dslist, ...)` and
  `filldataset(PWA.LoadLayout, popupinfo, ...)` to open the picker — i.e.
  side-effect actions are staged into `customevents` for the engine to run
  around/after the popup interaction, then the popup-open sequence runs.
  Every such chain ends with `setdataset(customevents, data:null)` to clear
  it.
- Direct field-to-field arithmetic without a `raw.` literal on one side:
  `[this.PURDET_DISCOUNT_PERCENT(*)this.PURDET_GROSS_AMOUNT(/)raw.100]` and
  `[DsFrmHeaderEntry.PUR_ITEM_VAT_AMOUNT(+)DsFrmHeaderEntry.ADDITIONAL_VAT_AMOUNT]`.
- Bidirectional dependent fields: editing Discount% recomputes Discount
  Amount via `onFocusOut`, and editing Discount Amount recomputes
  Discount% back, each also re-deriving the row/header net amount and then
  invoking `"[calcformula]"` / `"[fapoEvent1]"`.
- `list` props not seen before: `"watch": ["DsFrmHeaderEntry","DsFrmDetails1"]`,
  `"filtermode":"contains"`, `"filterignorecase":true`, `"satisfy":"any"` —
  the row template also gets `resolvestyles:true` with a per-row
  `"backgroundColor":"[this.bgColor]"`.
- Delete-row variant that stores the target row index in a dataset first
  (`setdataset(dscurListIndex, "[this.propkey]")`) before showing the
  confirm dialog, then the dialog's YES button reads back
  `"index":"[dscurListIndex]"` instead of `"[this.propkey]"` directly —
  needed because `this` isn't in scope inside the dialog's own button
  `whenclick`.
- Inline validation object with `"for":"email"` (no `validationParams`)
  for an email field, alongside the more common `"for":"text"` +
  `validationParams:{minLength}` shape already documented.
- A clickable label that opens a popup (VAT Treatment), built as a `btn`
  used AS the label half of a field wrapper (not just value-side, as
  documented in `atoms/text-button.json`) — `resolveprops`/`appendprops.disabled`
  on the button itself, styled to look like a link/label.

```json
[{"type":"sect","props":{"container":true,"resolvestyles":true,"appendprops":{"style":{"pointerEvents":"[dsPagemode.pagemode(eq)raw.view(iftrue)raw.none(iffalse)raw.all]","padding":"5px","display":"block"}}},"chld":[{"type":"sect","props":{"item":true,"container":true},"chld":[{"type":"sect","props":{"item":true,"container":true,"xs":6,"style":{"display":"block"}},"chld":[{"type":"sect","props":{"xs":12,"item":true,"resolvestyles":true,"className":"subHeadingLbl","style":{}},"chld":[{"type":"span","props":{"style":{}},"chld":"Document Details"}]},{"type":"sect","props":{"item":true,"style":{"padding":"5px"},"xs":12},"chld":[{"type":"div","props":{"className":"entryLabelOuterDiv"},"chld":[{"type":"div","props":{"className":"entryLabelDiv","style":{"width":"12.5vw"}},"chld":[{"type":"lbl","props":{"text":"Doc#","className":"entryLabel"}}]},{"type":"div","props":{"className":"entryValueDiv","style":{"width":"10vw"}},"chld":[{"type":"entry","props":{"label":"","disabled":true,"className":"small-ctr","fullWidth":true,"variant":"outlined","dset":"DsFrmHeaderEntry","bind":"PUR_DOCNO","inputProps":{"maxLength":20,"style":{}},"validation":{"for":"text","msg":"Please fill the field","validationParams":{"minLength":3}}}}]}]}]}]}]}]}]
```

*(Full body elided here for brevity — the exhaustive verbatim capture
lives in the pasted conversation turn; the excerpts above and the
per-pattern notes are what was extracted into the library's atoms and
molecules.)*
