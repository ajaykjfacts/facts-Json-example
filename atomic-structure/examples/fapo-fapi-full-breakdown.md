# FAPO/FAPI Purchase Order — Exhaustive Atomic Breakdown

Complete element-by-element, prop-by-prop decomposition of the six layout-store
keys pasted by the project owner:

| Key | Kind | Purpose |
|---|---|---|
| `##LISTLAYOUT1COLS##` | generic row template | 1-column static list row (engine-provided) |
| `##LISTLAYOUT2COLS##` | generic row template | 2-column caption/value static list row (engine-provided) |
| `_fapi-general-sect` | page layout | Purchase Invoice page — **complete capture**, the authoritative source below |
| `_fapo-general-sect` | page layout | Purchase Order page — identical to `_fapi` up to the truncation point |
| `_fapi-whenload-events` | action list | page `whenload` — defines formulas, fetches sub-layouts + event fragments |
| `_fapo-event-1` | action list (`doctype:"part"`) | reusable per-row tax recalculation fragment |

Everything below was read directly out of that paste. Nothing is inferred or
invented. Where a prop appears only once, it is marked **(single usage)**.

---

# PART 1 — PAGE SKELETON

## 1.1 Root node

```json
{
  "type": "sect",
  "props": {
    "container": true,
    "resolvestyles": true,
    "appendprops": {
      "style": {
        "pointerEvents": "[dsPagemode.pagemode(eq)raw.view(iftrue)raw.none(iffalse)raw.all]",
        "padding": "5px",
        "display": "block"
      }
    }
  },
  "chld": [ ... ]
}
```

| Prop | Value | Meaning |
|---|---|---|
| `container` | `true` | MUI-grid container; children marked `item` become grid cells |
| `resolvestyles` | `true` | **Required** for any bracket expression inside `style`/`appendprops.style` to be evaluated. Without it the expression renders as a literal string |
| `appendprops` | object | Props merged in *after* expression resolution — this is where dynamic values live. Static values stay in `props` |
| `appendprops.style.pointerEvents` | ternary expression | **Page-wide read-only switch.** When `dsPagemode.pagemode == "view"` the entire page becomes `pointerEvents:none` (nothing clickable/editable); otherwise `all` |

**The whole page is made read-only by one expression at the root.** Individual
controls do *not* each check page mode. Only extra restrictions (per-field
locks like `DM_FIXED_LOCATION`) are re-declared lower down. Copy this root node
verbatim for any new entry page.

## 1.2 Top-level child order

```
root sect
├── sect (item, container)                    ← PART 2: header form, 2 columns + 1 full-width row
│   ├── sect (item, container, xs:6)          ← "Document Details"
│   ├── sect (item, container, xs:6)          ← "Party Details"
│   └── sect (item, container, xs:12)         ← "Other Details"
├── sect (container, className:TableParentSect)  ← PART 3: line-items grid
├── sect (xs:12, container)                   ← PART 4: "Delivery Information"
├── sect (xs:12, container)                   ← PART 5: "Additional Amount" grid
└── sect (xs:12, container)                   ← PART 6: "Summary" totals
```

## 1.3 Section heading atom (repeated 6×)

Every section opens with the identical heading node:

```json
{
  "type": "sect",
  "props": { "xs": 12, "item": true, "resolvestyles": true, "className": "subHeadingLbl", "style": {} },
  "chld": [ { "type": "span", "props": { "style": {} }, "chld": "Document Details" } ]
}
```

Observed texts: `Document Details`, `Party Details`, `Other Details`,
`Delivery Information`, `Additional Amount`, `Summary`.

Note the heading text is a **`span` with the text in `chld`**, not a `lbl` with
`props.text`. Both atoms exist in this engine; headings consistently use `span`.
The empty `"style": {}` and `resolvestyles:true` are carried on every instance
even though nothing dynamic is in them — harmless, and kept for consistency.
See `atoms/section-heading.json`.

---

# PART 2 — HEADER FORM

## 2.1 The universal field wrapper (used ~20× on this page)

```
div.entryLabelOuterDiv
├── div.entryLabelDiv      [optional style.width]
│   └── lbl (props.text, className:"entryLabel")
└── div.entryValueDiv      [optional style.width | optional appendprops.style]
    └── <control>
```

Width overrides observed:
- `entryLabelDiv` → `{"width":"12.5vw"}` on Doc#, Doc. Date, Expected Delivery Date
- `entryValueDiv` → `{"width":"10vw"}` on Doc#, Doc. Date, Expected Delivery Date
- No width (fills available) on every other field

The wrapper `sect` around each field is consistently
`{"item":true,"style":{"padding":"5px"},"xs":12}`. In the Delivery Information
column the wrapper also carries `className:"factsFullWidth"` and
`style.paddingLeft:"20px"`.

## 2.2 Complete field inventory

### Column A — "Document Details"

| # | Label | Control | Dataset.Field | Notable props |
|---|---|---|---|---|
| 1 | `Doc#` | `entry` | `DsFrmHeaderEntry.PUR_DOCNO` | `disabled:true`, `inputProps.maxLength:20`, `validation{for:text, minLength:3}`, `variant:"outlined"` |
| 2 | `Doc. Date` | `dxdtpick` | `DsFrmHeaderEntry.PUR_DOCDATE` | `type:"date"`, `labelMode:"floating"`, `closeOnSelect:true`, `showClearButton:false`, `required:true`, `whenchange:[]` |
| 3 | *(conditional block)* | `hide` > `view` | — | Previous-document fetch bar, see 2.4 |
| 4 | `Division` | `lkup` | `DsFrmHeaderEntry.PUR_DIVN_DESC` | listtype `FAPOPDIV`, key `PUR_DIVN_DOCNO` |
| 5 | `Location` | `lkup` | `DsFrmHeaderEntry.PUR_LOC_DESC` | listtype `FAPOUSRLOC`, key `PUR_LOCATION_DOCNO`, locked by `DM_FIXED_LOCATION` |

⚠️ **Naming inconsistency confirmed in the source** (do not "fix" it when
reusing): the Location lookup's `descfield` is `PUR_LOC_DESC` and its
`codefield` is `PUR_LOCATION_DOCNO` — *not* `PUR_LOC_DOCNO`. But
`_fapi-whenload-events` reads `[DsFrmHeaderEntry.PUR_LOC_DOCNO]` when building
`dsFetchHeader.LOCATION`. Two different field names for the location code
appear in the same page.

### Column B — "Party Details"

| # | Label | Control | Dataset.Field | Notable props |
|---|---|---|---|---|
| 6 | `GL code` | `lkup` | `DsFrmHeaderEntry.PUR_AC_DESC` | listtype `FAPOGL`, key `PUR_AC_DOCNO`, locked by `DM_FIXED_GL_DOCNO`, runs `GetAccountDetails` mode `GL` |
| 7 | `Supplier` | `lkup` | `DsFrmHeaderEntry.PUR_SUBLEDGER_DESC` | listtype `FAPOAP`, key `PUR_SUBLEDGER_DOCNO`, gated by `ACMST_SUBLEDGER_EXISTS`, `filtercondition` on GL, runs `GetAccountDetails` mode `SL` |
| 8 | `Contact` | `lkup` | `DsFrmHeaderEntry.PUR_CONTACT_DESC` | listtype `FAPOACCONDET`, key `PUR_CONTACT_DOCNO`, **no `validation` block** despite `required:true` |
| 9 | `Currency` | `lkup` | `DsFrmHeaderEntry.PUR_CURR_DESC` | listtype `FAPOCUR`, key `PUR_CURR_DOCNO`, hidden by `HIDE_CURRENCY`, runs `GetCurrencyRate` |
| 10 | `Exchange Rate` | `dxnumberbox` | `DsFrmHeaderEntry.PUR_CURR_RATE` | `disabled:true`, `format:"##,###.00"`, hidden by `HIDE_EXCHANGE_RATE`, `whenchange:[]` |

### Column C — "Other Details" (full width, `xs:12`, fields at `xs:12 md:6`)

| # | Label | Control | Dataset.Field | Notable props |
|---|---|---|---|---|
| 11 | `Staff` | `lkup` | `DsFrmHeaderEntry.PUR_SMAN_DESC` | listtype `FAPOSMAN`, key `PUR_SMAN_DOCNO` |
| 12 | `Payment Mode` | `lkup` | `DsFrmHeaderEntry.PUR_PAYMODE_DESC` | listtype `FAPOPAYMOD`, key `PUR_PAYMODE_DOCNO` |

## 2.3 The three field-lock mechanisms (all different, all on this page)

| Mechanism | Where it sits | Expression | Effect |
|---|---|---|---|
| **Hard-coded** | on the control | `"disabled": true` | Doc#, Exchange Rate — never editable |
| **Conditional lock** | `sect` wrapper `style.pointerEvents` + `resolveprops`+`resolvestyles` | `[dsDocumentMaster.DM_FIXED_LOCATION(eq)raw.1(iftrue)raw.none(iffalse)raw.all]` | Location, GL code — visible but frozen when the doctype master says the value is fixed |
| **Conditional lock (inner)** | `div.entryValueDiv` `appendprops.style.pointerEvents` | `[DsFrmHeaderEntry.ACMST_SUBLEDGER_EXISTS(eq)raw.1(iftrue)raw.all(iffalse)raw.none]` | Supplier — **inverted polarity**: enabled only when the chosen GL account actually has sub-ledgers |
| **Conditional hide** | `hide` wrapper | `[dsDocumentMaster.HIDE_CURRENCY(eq)raw.1]` | Currency, Exchange Rate, VAT Amount — removed entirely |

Note the lock expression lives on the **`sect`** for Location/GL but on the
**inner `entryValueDiv`** for Supplier. Both work; the inner placement leaves the
label clickable/selectable.

## 2.4 Previous-document fetch bar **(single usage)**

```json
{
  "type": "hide",
  "props": { "appendprops": { "xsUp": "[dsDocumentMaster.DM_PREVIOUS_DOCTYPE(ifnull)raw.(eq)raw.]" } },
  "chld": [
    { "type": "sect", "props": { "container": true, "xs": 10, "style": { "padding": "5px", "display": "flex" } },
      "chld": [
        { "type": "view", "props": { "includerefdata": true, "contenttype": "json", "binding": true,
                                      "content": "[_globalLayouts.fetchPreviousDoc]" } }
      ] }
  ]
}
```

Two distinct things to note:

1. **The `(ifnull)raw.(eq)raw.` chain** — coalesce `DM_PREVIOUS_DOCTYPE` to
   empty string, then compare to empty string. The `hide` shows its children
   when `xsUp` is falsy, so this renders the bar **only when a previous doctype
   IS configured**. Note `raw.` with nothing after it is a valid empty-string
   literal.
2. **`_globalLayouts` dataset** — a globally available layout store, addressed
   like any dataset. `[_globalLayouts.fetchPreviousDoc]` pulls a shared,
   app-wide sub-layout (the "copy from previous document" widget) into this
   page. This is the *global* counterpart to
   `[dsCountryMasterDeliverTo.settingsView]`, which is fetched per-page. When
   you need a standard cross-page widget, look in `_globalLayouts` first.

## 2.5 Lookup field — the complete anatomy

Every one of the 8 lookups on this page has exactly this shape:

```json
{
  "type": "lkup",
  "props": {
    "dset": "DsFrmHeaderEntry",
    "required": true,
    "bind": "PUR_DIVN_DESC",
    "variant": "outlined",
    "className": "small-ctr",
    "label": "",
    "fullWidth": true,
    "icon": "search",
    "validation": { "for": "text", "msg": "Please fill the field", "validationParams": { "minLength": 3 } },
    "icons": {
      "notempty": {
        "icon": "close",
        "whenclick": [
          { "exec": "mergedataset", "args": { "dset": "DsFrmHeaderEntry",
              "data": { "PUR_DIVN_DOCNO": "", "PUR_DIVN_DESC": "" } } }
        ]
      }
    },
    "whenclick": [ /* 2 or 3 actions — see 2.6 */ ]
  }
}
```

| Prop | Always | Notes |
|---|---|---|
| `bind` | the **`_DESC`** field | The lookup always displays and binds the description; the `_DOCNO` code field is written by the popup via `codefield` |
| `label` | `""` | Empty because the visible label is the sibling `lbl` in `entryLabelDiv` |
| `icon` | `"search"` | Trailing search affordance |
| `icons.notempty.icon` | `"close"` | Swaps to a clear button when the field has a value |
| `icons.notempty.whenclick` | `mergedataset` | Clears **both** the desc and the code field. Always clear both — clearing only the visible one leaves a stale code |
| `required` | `true` on all 8 | But only 5 carry a `validation` block; `required` alone does not enforce anything (validation is action-based) |

## 2.6 Lookup popup-open sequence (the 2- and 3-action forms)

**Simple form (2 actions)** — Division, Location, Contact, Staff, Payment Mode:

```json
[
  { "exec": "setdataset", "args": { "dset": "customevents", "data": null } },
  { "exec": "setdataset", "args": { "dset": "dslist", "data": {
      "listtype": "FAPOPDIV", "commontype": "manual", "title": "Division",
      "dataset": "DsFrmHeaderEntry", "section": "Alltype",
      "doctypefield": "PUR_DIVN_DOCTYPE", "codefield": "PUR_DIVN_DOCNO", "descfield": "PUR_DIVN_DESC" } } },
  { "exec": "filldataset", "args": { "proc": "PWA.LoadLayout", "column": "layoutinfo",
      "dset": "popupinfo", "section": "Alltype", "args": { "doctype": "popup", "docno": "lookup" } } }
]
```

**Chained form (3 actions)** — GL code, Supplier, Currency: identical, except
the first action *stages a follow-up action list* into `customevents` instead
of clearing it. See PART 7.2.

`dslist` payload fields:

| Field | Meaning |
|---|---|
| `listtype` | Server-side list identifier (see PART 9) |
| `commontype` | Always `"manual"` here |
| `title` | Popup window title |
| `dataset` | Target dataset the picked row writes back into |
| `section` | Always `"Alltype"` for lookups |
| `doctypefield` | Field receiving the picked row's doctype |
| `codefield` | Field receiving the picked row's code |
| `descfield` | Field receiving the picked row's description |
| `filtercondition` | **Supplier only** — `{"PUR_AC_DOCNO":"[DsFrmHeaderEntry.PUR_AC_DOCNO]"}` restricts suppliers to the chosen GL account |
| `rowindex` | **Grid lookups only** — `"[this.propkey]"` |
| `calledFrom` | **Grid lookups only** — `"list"` |

The third action is byte-identical across every lookup on the page: it fetches
the shared `doctype:"popup" / docno:"lookup"` layout into the `popupinfo`
dataset, which the shell renders as a modal. **The lookup popup is one shared
layout; `dslist` is how you configure it.**

---

# PART 3 — LINE-ITEMS GRID (`DsFrmDetails1`)

## 3.1 Container

```json
{ "type": "sect", "props": { "container": true, "className": "TableParentSect",
                              "style": { "padding": "7px 7px 5px" } }, "chld": [ ... ] }
```

## 3.2 Header row — 12 columns, `xs` summing to 12

```json
{ "type": "sect", "props": { "className": "TableHeaderSect", "item": true, "container": true, "xs": 12 }, "chld": [...] }
```

| # | Header text | `xs` | className |
|---|---|---|---|
| 1 | `Sr#` | 0.5 | `TableHeaderChildSect` |
| 2 | `Stock Code` | 1.5 | `TableHeaderChildSect` |
| 3 | `Stock Description` | 2 | `TableHeaderChildSect` |
| 4 | `Unit` | 0.5 | `TableHeaderChildSect` |
| 5 | `Quantity` | 1 | `TableHeaderChildSect` |
| 6 | `Rate` | 1 | `TableHeaderChildSect` |
| 7 | `Gross Amount` | 1 | `TableHeaderChildSect` |
| 8 | `Discount %` | 1 | `TableHeaderChildSect` |
| 9 | `Discount Amount` | 1 | `TableHeaderChildSect` |
| 10 | `VAT Amount` | 1 | `TableHeaderChildSect` |
| 11 | `Amount` | 1 | `TableHeaderChildSect` |
| 12 | *(empty — actions)* | 0.5 | `TableHeaderLastChildSect` |

Total = 12.0 exactly. **Fractional `xs` values (0.5, 1.5) are legal here** —
this is MUI Grid v5+ behaviour. Header cells wrap their text in a bare `span`
with no props: `{"type":"span","props":{},"chld":"Sr#"}`.

The last column uses `TableHeaderLastChildSect` (different className, and it is
left empty even though the body's matching cell holds the delete button).

## 3.3 Scroll container

```json
{ "type": "sect", "props": { "item": true, "xs": 12,
    "style": { "minHeight": "6vh", "maxHeight": "50vh", "overflow": "scroll" } }, "chld": [ /* list */ ] }
```

The grid body — not the page — owns the scrollbar. `minHeight:6vh` keeps the
empty state from collapsing; `maxHeight:50vh` caps it at half the viewport.

## 3.4 The `list` repeater — full prop set

```json
{
  "type": "list",
  "props": {
    "datasets": ["DsFrmDetails1"],
    "watch": ["DsFrmHeaderEntry", "DsFrmDetails1"],
    "filtermode": "contains",
    "filterignorecase": true,
    "satisfy": "any",
    "className": "TableListSect facts-input-list",
    "content": {
      "template": "custom",
      "props": { "resolvestyles": true, "style": { "backgroundColor": "[this.bgColor]" } },
      "chld": [ /* one sect container = one row */ ]
    }
  }
}
```

| Prop | Value | Meaning |
|---|---|---|
| `datasets` | `["DsFrmDetails1"]` | Array, though only ever one entry. The rows iterated |
| `watch` | `["DsFrmHeaderEntry","DsFrmDetails1"]` | **Re-render triggers.** Includes the *header* dataset, so header changes (e.g. VAT treatment) repaint rows. Without this, rows would not refresh when a header field changes |
| `filtermode` | `"contains"` | Substring matching for the list's filter |
| `filterignorecase` | `true` | Case-insensitive filter |
| `satisfy` | `"any"` | OR semantics across filter criteria (vs `"all"`) |
| `className` | `"TableListSect facts-input-list"` | Two classes, space-separated. `facts-input-list` marks a grid whose rows contain inputs |
| `content.template` | `"custom"` | Use `content.chld` as the row template (the only value observed anywhere) |
| `content.props.resolvestyles` | `true` | Required for the row background expression below |
| `content.props.style.backgroundColor` | `"[this.bgColor]"` | **Per-row background driven by a data field.** The row's own `bgColor` value (set by server or by an action) colours it. Compare with the `[raw.facts-row-[this.row_error]]` className approach in `molecules/grid-row-template.json` — this page uses the inline-style variant instead |

## 3.5 Row cells — complete

Row root: `{"type":"sect","props":{"container":true,"style":{}},"chld":[...]}`
Each cell: `{"type":"sect","props":{"item":true,"className":"TableContentChildSect","xs":N},"chld":[control]}`

| # | Field | `xs` | Control | Editable | Behaviour |
|---|---|---|---|---|---|
| 1 | `PURDET_DOCSRNO` | 0.5 | `span` `[this.PURDET_DOCSRNO]` | ✗ | Serial number, display only |
| 2 | `PURDET_STOCK_DOCNO` | 1.5 | `lkup` | ✓ | Stock picker, listtype `FAPOSMGT` → triggers `FAPO_Onchange` |
| 3 | `PURDET_STOCK_DESC` | 2 | `span` | ✗ | Filled by the picker |
| 4 | `PURDET_UNIT_DOCNO` | 0.5 | `span` | ✗ | Filled by `FAPO_Onchange` |
| 5 | `PURDET_QTY` | 1 | `dxnumberbox` | ✓ | `onFocusOut` → `FAPO_Onchange` + `[fapoEvent1]` |
| 6 | `PURDET_RATE` | 1 | `dxnumberbox` | ✓ | `onFocusOut` → `FAPO_Onchange` + `[fapoEvent1]` |
| 7 | `PURDET_GROSS_AMOUNT` | 1 | `dxnumberbox` `disabled:true` | ✗ | Computed server-side |
| 8 | `PURDET_DISCOUNT_PERCENT` | 1 | `dxnumberbox` | ✓ | `onFocusOut` → computes amount, then `[fapoEvent1]` |
| 9 | `PURDET_DISCOUNT_AMOUNT` | 1 | `dxnumberbox` | ✓ | `onFocusOut` → computes percent, then `[fapoEvent1]` |
| 10 | `PURDET_VAT_AMOUNT` | 1 | `dxnumberbox` `disabled:true` | ✗ | Computed by `FAPO_TaxCalculateItemWise` |
| 11 | `PURDET_AMOUNT` | 1 | `dxnumberbox` `disabled:true` | ✗ | Gross − Discount |
| 12 | *(actions)* | 0.5 | `hide` > `icbtn` | — | Delete, `TableContentLastChildSect` |

Every editable numeric cell carries the identical prop set:

```json
{ "dset": "DsFrmDetails1", "className": "small-ctr", "fullWidth": true,
  "format": "##,###.00", "variant": "outlined", "includerefdata": true,
  "rowindex": "[this.propkey]", "bind": "<FIELD>" }
```

⚠️ **Read-only numbers in this grid are `dxnumberbox` + `disabled:true`, while
read-only numbers in the Summary section are `dxnumlbl`.** Both appear on the
same page. The distinction (per `atoms/numeric-label.json`): `dxnumlbl` for
values that are *always* computed; disabled `dxnumberbox` for values sitting in
an editable row where the disabled state is contextual.

## 3.6 Stock lookup — the grid-row variant of the popup sequence

```json
"whenclick": [
  { "exec": "setdataset", "args": { "nodeepprocess": true, "dset": "customevents", "data": [
      { "exec": "fillmultidataset", "args": { "proc": "FAMST.FAPO_Onchange",
          "dsets": [ { "name": "dsFAPOOnchange", "row": 0, "table": "Table" } ],
          "args": { "strXmlDetails": "[DsFrmDetails1.[dslist.rowindex]]", "mode": "details" } } },
      { "exec": "mergedatasetarray", "args": { "dset": "DsFrmDetails1", "index": "[dslist.rowindex]", "data": "[dsFAPOOnchange]" } },
      { "exec": "setdataset", "args": { "dset": "customevents", "data": null } }
  ] } },
  { "exec": "setdataset", "args": { "dset": "dslist", "data": {
      "listtype": "FAPOSMGT", "commontype": "manual", "title": "Stock Master",
      "rowindex": "[this.propkey]", "calledFrom": "list",
      "dataset": "DsFrmDetails1", "section": "Alltype",
      "doctypefield": "PURDET_STOCK_DOCTYPE", "codefield": "PURDET_STOCK_DOCNO", "descfield": "PURDET_STOCK_DESC" } } },
  { "exec": "filldataset", "args": { "proc": "PWA.LoadLayout", "column": "layoutinfo",
      "dset": "popupinfo", "section": "Alltype", "args": { "doctype": "popup", "docno": "lookup" } } }
]
```

🔑 **The single most important detail on this page:** inside the staged
`customevents` actions the row index is `[dslist.rowindex]`, **not**
`[this.propkey]`. Reason: `customevents` runs *after the popup closes*, by which
time the `this` row context is gone. The row index is therefore parked in
`dslist.rowindex` (line 2 of the same action array) and read back from there.

Note the nested expression `"[DsFrmDetails1.[dslist.rowindex]]"` — a bracket
expression **inside** another bracket expression, resolving to "row N of
DsFrmDetails1".

## 3.7 Quantity / Rate — `onFocusOut` server round-trip

```json
"onFocusOut": [
  { "exec": "fillmultidataset", "args": { "proc": "FAMST.FAPO_Onchange",
      "dsets": [ { "name": "dsFAPOOnchange", "row": 0, "table": "Table" } ],
      "args": { "strXmlDetails": "[this]", "mode": "details" } } },
  { "exec": "mergedatasetarray", "args": { "dset": "DsFrmDetails1", "index": "[this.propkey]", "data": "[dsFAPOOnchange]" } },
  "[fapoEvent1]"
]
```

Three-step shape, reused everywhere in this engine:
1. **Send** the current row to a stored proc — `"[this]"` serialises the whole
   row context as the `strXmlDetails` argument.
2. **Merge back** the returned single row into the same index.
3. **Splice in** the shared fragment `[fapoEvent1]`, which recalculates tax and
   then calls `[calcformula]` for header totals.

`onFocusOut` (not `whenchange`) is used for every editable numeric in the grid —
avoids a server call per keystroke. `whenchange` is used for the *header*
Discount% field in the Summary section, where the calculation is purely local.

## 3.8 Discount% ⇄ Discount Amount — bidirectional local calculation

Neither direction calls the server. Both end by delegating to `[fapoEvent1]`.

**Percent edited** → derive amount:
```json
"onFocusOut": [
  { "exec": "mergedatasetarray", "args": { "dset": "DsFrmDetails1", "index": "[this.propkey]",
      "data": { "PURDET_DISCOUNT_AMOUNT": "[this.PURDET_DISCOUNT_PERCENT(*)this.PURDET_GROSS_AMOUNT(/)raw.100]" } } },
  { "exec": "mergedatasetarray", "args": { "dset": "DsFrmDetails1", "index": "[this.propkey]",
      "data": { "PURDET_AMOUNT": "[this.PURDET_GROSS_AMOUNT(-)this.PURDET_DISCOUNT_AMOUNT]" } } },
  "[fapoEvent1]"
]
```

**Amount edited** → derive percent (operators inverted):
```json
"data": { "PURDET_DISCOUNT_PERCENT": "[this.PURDET_DISCOUNT_AMOUNT(/)this.PURDET_GROSS_AMOUNT(*)raw.100]" }
```
followed by the identical `PURDET_AMOUNT` recalculation and `[fapoEvent1]`.

⚠️ **The two `mergedatasetarray` calls must be separate actions.** The second
reads `PURDET_DISCOUNT_AMOUNT`, which the first one writes — they are sequenced,
not merged into a single `data` object, because expressions within one `data`
object evaluate against the *pre-merge* state.

⚠️ **No divide-by-zero guard exists** on `(/)this.PURDET_GROSS_AMOUNT`. This is
the source's actual behaviour; if you need a guard, add it consciously.

## 3.9 Row delete — the indexed-dialog variant

```json
"whenclick": [
  { "exec": "showloader" },
  { "exec": "setdataset", "args": { "dset": "dscurListIndex", "data": "[this.propkey]" } },
  { "exec": "setdataset", "args": { "dset": "dialoginfo", "nodeepprocess": true, "data": {
      "open": true, "title": "Delete", "description": "Delete this Item?",
      "btn1": { "chld": "YES", "props": { "whenclick": [
          { "exec": "mergedatasetarray", "args": { "dset": "DsFrmDetails1", "index": "[dscurListIndex]", "data": null } },
          "[calcformula]"
      ] } },
      "btn2": { "chld": "NO" }
  } } }
]
```

Differences from `molecules/grid-row-action-delete.json` (the previously
documented version), all significant:

| | Previously documented | This page |
|---|---|---|
| Row index | `"[this.propkey]"` read inside the dialog | Parked in `dscurListIndex` first, read back as `"[dscurListIndex]"` |
| `nodeepprocess` | absent | `true` on the `dialoginfo` setdataset |
| After delete | nothing | `"[calcformula]"` recalculates totals |
| `btn2` | `{"chld":"NO","props":{"whenclick":[]}}` | `{"chld":"NO"}` — props omitted entirely |
| Description | "Are you sure you want to delete this Item?" | "Delete this Item?" |

🔑 **`nodeepprocess:true` is what makes this work.** Without it the engine would
resolve `[dscurListIndex]` and `[calcformula]` *while building the dialog*
rather than when YES is clicked. With it, the nested action list is stored raw
and only resolved at click time — which is also exactly why `[this.propkey]`
cannot be used there (no row context at click time) and the index must be
parked in a dataset.

## 3.10 Add-row button — validate then append

```json
{
  "type": "btn",
  "props": {
    "variant": "inherit", "color": "primary", "size": "small",
    "className": "tableAddRowBtn", "style": { "float": "right" },
    "whenclick": [
      { "exec": "validatedataset", "verify": {
          "satisfy": "any", "halton": "match",
          "ref": { "[DsFrmHeaderEntry.PUR_AC_DOCNO(ifnull)raw.]": "",
                   "[DsFrmHeaderEntry.PUR_SUBLEDGER_DOCNO(ifnull)raw.]": "" },
          "whenmatch": [ { "exec": "setdataset", "args": { "dset": "snackinfo",
              "data": { "open": true, "type": "error", "message": "Please select the GL code and SL code." } } } ] } },
      { "exec": "mergedatasetarray", "args": { "dset": "DsFrmDetails1",
          "data": { "PURDET_DOCSRNO": "[DsFrmDetails1(listcount)(+)raw.1(numformat)raw.0]" } } }
    ]
  },
  "chld": [ { "type": "div", "props": { "style": { "height": "100%", "display": "flex", "alignItems": "center" } },
      "chld": [ { "type": "icon", "props": { "className": "tableAddRowBtnIcon" }, "chld": "add" },
                { "type": "lbl", "props": { "className": "tableAddRowBtnLbl", "text": "Add New Row" } } ] } ]
}
```

Four separate patterns in one node:

1. **Guard-before-append.** `validatedataset` with `halton:"match"` aborts the
   whole action array when the condition matches, so the append never runs.
   `satisfy:"any"` = fire if *either* GL or Supplier is blank.
2. **`(ifnull)raw.` idiom for "is empty".** `"[X(ifnull)raw.]": ""` reads as
   "coalesce X to empty, compare against empty".
3. **Append with computed serial.** `mergedatasetarray` with **no `index`**
   appends a new row. `[DsFrmDetails1(listcount)(+)raw.1(numformat)raw.0]` =
   current row count + 1, formatted with zero decimals.
4. **Icon+text button.** `btn.chld` is an array holding a flex `div` wrapping an
   `icon` and a `lbl`.

The whole button sits inside `{"type":"hide","props":{"appendprops":{"xsUp":"[dsPagemode.pagemode(eq)raw.view]"}}}`
so it disappears in view mode — belt-and-braces alongside the root
`pointerEvents` lock.

---

# PART 4 — DELIVERY INFORMATION

Left half (`xs:6`) is an entirely server-composed sub-layout; right half
(`xs:6`) is four ordinary fields.

## 4.1 Server-composed address block **(the `view` atom)**

```json
{ "type": "sect", "props": { "item": true, "xs": 6, "style": { "padding": "5px", "paddingLeft": "20px" } },
  "chld": [ { "type": "view", "props": { "includerefdata": true, "contenttype": "json", "binding": true,
                                          "content": "[dsCountryMasterDeliverTo.settingsView]", "style": {} } } ] }
```

The entire country/state/city/postcode address form — its fields, its labels,
its cascading lookups — is **not in this layout at all**. It arrives as JSON in
`dsCountryMasterDeliverTo.settingsView`, fetched in `whenload` by
`PWA.CountryMasterView_V2`. Country-specific address shapes (emirate vs state
vs province) are therefore handled centrally, not per page.

The three-part wiring:

| Step | Where | What |
|---|---|---|
| 1 | `whenload` | `filldataset PWA.CountryMasterView_V2` → `dsCountryMasterDeliverTo`, args `{layout_docno:"country-master-view-new", doctype:"[pagemenuinfo.doctype]", subViewDset:"dsAddressSubview", dataset:"dsAddress"}` |
| 2 | `whenload` | `filldataset PWA.countrySubView_V2` → `dsAddressSubview`, args `{layout_docno:"country-subview-default-new", dataset:"dsAddress"}` |
| 3 | `whenload` | `mergedataset` seeds `dsAddress` from the header's `PUR_BILL_*` fields |
| 4 | layout | the `view` atom renders `dsCountryMasterDeliverTo.settingsView` against `dsAddress` |

**Field-name translation at step 3** — the header's `PUR_BILL_*` names are
mapped onto the address component's generic names:

| `dsAddress` field | ← `DsFrmHeaderEntry` field |
|---|---|
| `country_name` | `PUR_BILL_COUNTRY` |
| `street` | `PUR_BILL_STREET` |
| `city` | `PUR_BILL_CITY` |
| `emiratedesc` | `PUR_BILL_STATE` |
| `postaldesc` | `PUR_BILL_ZIPCODE` |

## 4.2 Right-hand fields

All four wrappers carry `className:"factsFullWidth"` and
`style:{"padding":"5px","paddingLeft":"20px"}`.

| Label | Control | Field | Notes |
|---|---|---|---|
| `Expected Delivery Date` | `dxdtpick` | `PUR_DELIVERY_DATE` | Same prop set as Doc. Date; label/value widths 12.5vw/10vw |
| `Telephone-1 *` | `entry` | `PUR_BILL_TEL1` | `maxLength:60`. ⚠️ The asterisk is **typed into the label text**, not driven by `required` |
| `Mobile` | `entry` | `PUR_BILL_MOBILE` | `maxLength:60` |
| `Email-1` | `entry` | `PUR_BILL_EMAIL` | `maxLength:100`, `inputProps.style:{"padding":"5px"}`, `validation:{"for":"email","msg":"Incorrect email format"}` — **no `validationParams`** |

The email field confirms a second `validation.for` value: `"email"` alongside
the common `"text"`. `"text"` requires `validationParams.minLength`; `"email"`
takes none.

Note these `entry` controls use **`inputVariant:"outlined"`** whereas the header
fields use **`variant:"outlined"`**. Both spellings appear on this page.

---

# PART 5 — ADDITIONAL AMOUNT GRID (`DsFrmDetails2`)

A second grid with deliberately different mechanics from the line-items grid.

## 5.1 Structure

```json
{ "type": "sect", "props": { "container": true, "className": "TableParentSect", "style": { "minWidth": "900px" } } }
```

`minWidth:900px` instead of the line-item grid's padding — this grid forces
horizontal scroll rather than compressing.

| # | Header | `xs` | Row control | Bind |
|---|---|---|---|---|
| 1 | `Code` | 2 | `span` | `[this.ADAMTDET_MST_DOCNO]` |
| 2 | `Description` | 5 | `span` | `[this.ADAMTDET_MST_DESC]` |
| 3 | `Amount` | 2 | `dxnumlbl` | `ADAMTDET_AMOUNT` |
| 4 | `VAT Amount` | 2 | `dxnumlbl` | `ADAMTDET_VAT_AMOUNT` |
| 5 | `Options` | 1 | `icbtn` delete | — |

Sums to 12. Unlike the line-items header, the last cell here **has text**
(`Options`).

## 5.2 How it differs from the line-items grid — and why

| | Line items (`DsFrmDetails1`) | Additional amounts (`DsFrmDetails2`) |
|---|---|---|
| Row cells | Editable `lkup` + `dxnumberbox` | Read-only `span` + `dxnumlbl` |
| `list` props | `watch`, `filtermode`, `filterignorecase`, `satisfy`, row `bgColor` | **`datasets` + `className` only** |
| Add row | `mergedatasetarray` appends an inline-editable row | Opens a **popup form** |
| Delete | Parks index in `dscurListIndex`, recalcs via `[calcformula]` | Uses `[this.propkey]` directly, **no recalc** |

**Add-row opens a form instead of appending:**
```json
"whenclick": [
  { "exec": "setdataset", "args": { "dset": "DsAdditionalAmount", "data": { "GrossAmount": "[DsFrmHeaderEntry.PUR_GROSS_AMOUNT]" } } },
  { "exec": "filldataset", "args": { "proc": "PWA.LoadLayout", "column": "layoutinfo", "dset": "popupinfo",
      "section": "additionalAmountSect", "args": { "doctype": "popup", "docno": "additional-amount" } } }
]
```
Seed the popup's input dataset first (here: the gross amount, so the popup can
compute percentage-based charges), then load the popup layout. Same
`filldataset`+`popupinfo` mechanism as the lookup popup, but with
`docno:"additional-amount"` and `section:"additionalAmountSect"` instead of the
shared `docno:"lookup"`/`section:"Alltype"`.

⚠️ **The delete here does not call `[calcformula]`**, unlike the line-items
delete — yet `PUR_OTHER_AMOUNT` is a `listsum` over this very dataset. Deleting
an additional-amount row leaves the Summary stale until something else triggers
a recalculation. Recorded as-is; treat as a bug to be aware of, and include
`"[calcformula]"` when copying this pattern.

---

# PART 6 — SUMMARY

Layout: an empty `sect xs:8` spacer, then `sect xs:4` holding six right-aligned
rows. Every row uses the standard field wrapper with
`className:"factsFullWidth"` and `paddingLeft:"20px"`.

| # | Label | Control | Field | Source |
|---|---|---|---|---|
| 1 | `Total Quantity` | `dxnumlbl` | `PUR_TOTAL_QTY` | `listsum` of `PURDET_QTY` |
| 2 | `Gross Amount` | `dxnumlbl` | `PUR_GROSS_AMOUNT` | `listsum` of `PURDET_GROSS_AMOUNT` |
| 3 | `Discount(%)` | `dxnumberbox` **+** `dxnumlbl` | `PUR_DISCOUNT_PERCENT`, `PUR_DISCOUNT_AMOUNT` | user-entered % → computed amount |
| 4 | `Additional Amount` | `dxnumlbl` | `PUR_OTHER_AMOUNT` | `listsum` of `ADAMTDET_AMOUNT` |
| 5 | `VAT Amount` | `dxnumlbl` | `PUR_VAT_AMOUNT` | item VAT + additional VAT |
| 6 | `Net Amount` | `dxnumlbl` | `PUR_NET_AMOUNT` | full formula, see 7.1 |

## 6.1 Two controls in one value slot **(single usage)**

```json
{ "type": "div", "props": { "className": "entryValueDiv", "style": { "display": "flex" } },
  "chld": [
    { "type": "dxnumberbox", "props": { "dset": "DsFrmHeaderEntry", "className": "small-ctr", "fullWidth": true,
        "format": "##,###.00", "variant": "outlined", "includerefdata": true, "bind": "PUR_DISCOUNT_PERCENT",
        "label": "%",
        "whenchange": [
          { "exec": "mergedataset", "args": { "dset": "DsFrmHeaderEntry", "data": {
              "PUR_DISCOUNT_AMOUNT": "[DsFrmHeaderEntry.PUR_DISCOUNT_PERCENT(*)DsFrmHeaderEntry.PUR_GROSS_AMOUNT(/)raw.100]" } } },
          "[calcformula]"
        ] } },
    { "type": "dxnumlbl", "props": { "dset": "DsFrmHeaderEntry", "fullWidth": true, "includerefdata": true,
        "bind": "PUR_DISCOUNT_AMOUNT" } }
  ] }
```

Put `display:flex` on `entryValueDiv` and it holds two controls side by side:
editable percent, read-only resulting amount. Note this one uses a non-empty
`label:"%"` — the only control on the page that does.

Also note: **`whenchange` here, `onFocusOut` in the grid.** Local-only maths can
afford to run per keystroke; server round-trips cannot. And unlike the row-level
discount, this header discount is **one-directional** — editing the amount is
impossible because it is a `dxnumlbl`.

## 6.2 A button used as a field label **(single usage)**

```json
{ "type": "div", "props": { "className": "entryLabelDiv" }, "chld": [
  { "type": "btn", "resolveprops": true,
    "props": {
      "title": "Click to update VAT Treatment of Party",
      "includerefdata": true,
      "appendprops": { "disabled": "" },
      "color": "inherit",
      "whenclick": [ { "exec": "filldataset", "args": { "proc": "PWA.LoadLayout", "column": "layoutinfo",
          "dset": "popupinfo", "section": "taxTreatmentSect", "args": { "doctype": "popup", "docno": "vat-treatment" } } } ],
      "resolvestyles": true,
      "style": { "color": "blue", "textDecoration": "underline", "textTransform": "capitalize", "padding": "0px" }
    },
    "chld": "VAT Amount" } ] }
```

The "VAT Amount" caption is a link-styled `btn` occupying the `entryLabelDiv`
slot where a `lbl` normally goes. `atoms/text-button.json` documented buttons on
the value side only; this confirms the label side too.

Quirks worth copying carefully:
- `"resolveprops": true` sits at **node level**, outside `props` — everywhere
  else on this page it is inside `props`. Both forms occur in this file.
- `"appendprops": { "disabled": "" }` — an empty string, presumably a
  placeholder where a page-mode expression was intended. Harmless (falsy), but
  it is dead weight; do not cargo-cult it without an actual expression.
- `title` renders the native browser tooltip — the only tooltip on the page.

This whole row is wrapped in
`{"type":"hide","props":{"container":true,"appendprops":{"xsUp":"[dsDocumentMaster.HIDE_VAT(eq)raw.1]"}}}`,
confirming `hide` works at single-field-pair granularity, not just whole
sections.

---

# PART 7 — EVENT & CALCULATION ARCHITECTURE

Three tiers, each with a different lifetime and scope.

```
whenload (_fapi-whenload-events)
│
├── defines  calcformula   ── page-local named formula, stored in a dataset
├── fetches  fapoEvent1    ── server-stored shared fragment (doctype:"part")
├── fetches  dsCountryMasterDeliverTo.settingsView ── server-composed sub-layout
├── seeds    dsAddress     ── PUR_BILL_* → generic address field names
└── defines  dsFetchBeforeEvents ── staged actions for the "fetch previous doc" flow

control events
├── onFocusOut (grid numerics)  → server proc → merge row → "[fapoEvent1]"
├── whenchange (header percent) → local maths        → "[calcformula]"
└── whenclick  (lookups)        → stage customevents → configure dslist → open popup

fapoEvent1 (fragment)
└── FAPO_TaxCalculateItemWise → merge row → "[calcformula]"

calcformula (formula)
└── 2 × mergedataset: listsum aggregates, then derived totals
```

## 7.1 `calcformula` — the header totals formula

```json
{ "exec": "setdataset", "args": { "dset": "calcformula", "nodeepprocess": true, "data": [
  { "exec": "mergedataset", "args": { "dset": "DsFrmHeaderEntry", "data": {
      "PUR_TOTAL_QTY":         "[DsFrmDetails1(listsum)raw.PURDET_QTY]",
      "PUR_GROSS_AMOUNT":      "[DsFrmDetails1(listsum)raw.PURDET_GROSS_AMOUNT]",
      "PUR_ITEM_VAT_AMOUNT":   "[DsFrmDetails1(listsum)raw.PURDET_VAT_AMOUNT]",
      "PUR_OTHER_AMOUNT":      "[DsFrmDetails2(listsum)raw.ADAMTDET_AMOUNT]",
      "ADDITIONAL_VAT_AMOUNT": "[DsFrmDetails2(listsum)raw.ADAMTDET_VAT_AMOUNT]" } } },
  { "exec": "mergedataset", "args": { "dset": "DsFrmHeaderEntry", "data": {
      "PUR_VAT_AMOUNT": "[DsFrmHeaderEntry.PUR_ITEM_VAT_AMOUNT(+)DsFrmHeaderEntry.ADDITIONAL_VAT_AMOUNT]",
      "PUR_NET_AMOUNT": "[DsFrmDetails1(listsum)raw.PURDET_AMOUNT(+)DsFrmHeaderEntry.PUR_OTHER_AMOUNT(+)DsFrmHeaderEntry.PUR_ITEM_VAT_AMOUNT(+)DsFrmHeaderEntry.ADDITIONAL_VAT_AMOUNT(-)DsFrmHeaderEntry.PUR_DISCOUNT_AMOUNT]" } } }
] } }
```

**Two passes, and the split is mandatory.** Pass 1 computes the five raw
aggregates; pass 2 consumes pass 1's outputs. A single `mergedataset` would
evaluate every expression against pre-merge state, so `PUR_VAT_AMOUNT` would
read a stale `PUR_ITEM_VAT_AMOUNT`. Same sequencing rule as 3.8.

Net amount, expanded:
```
NET = Σ PURDET_AMOUNT + PUR_OTHER_AMOUNT + PUR_ITEM_VAT_AMOUNT + ADDITIONAL_VAT_AMOUNT − PUR_DISCOUNT_AMOUNT
```
Note `PUR_NET_AMOUNT` re-sums `PURDET_AMOUNT` from the detail rows rather than
reusing `PUR_GROSS_AMOUNT` (gross is pre-row-discount; `PURDET_AMOUNT` is
post-row-discount). The header-level `PUR_DISCOUNT_AMOUNT` is subtracted on top
of the already-applied per-row discounts.

`nodeepprocess:true` is what stores the action array instead of executing it.

## 7.2 `customevents` — deferred action staging

Used by GL code, Supplier, Currency, and the grid's Stock lookup. Shape:

```json
{ "exec": "setdataset", "args": { "nodeepprocess": true, "dset": "customevents", "data": [ /* run after popup */ ] } }
```

The engine runs whatever sits in `customevents` after the popup interaction
completes. Every staged list **ends by clearing itself**:
`{"exec":"setdataset","args":{"dset":"customevents","data":null}}`.
Correspondingly, every lookup that does *not* need follow-up actions **begins**
by clearing `customevents` — otherwise a previous lookup's staged actions would
fire against the wrong field.

🔑 Rule: **every lookup's `whenclick` must either stage a new `customevents`
list or clear it. Never leave it untouched.**

### GL code chain (mode `GL`)
```
setdataset dsVATHeaderDetails ← {DOCTYPE, DOCNO, DOCDATE, AC_DOCNO}
fillmultidataset PWA.GetAccountDetails (mode:"GL", strXmlHeader:[dsVATHeaderDetails]) → dsGetAccountDetails
mergedataset DsFrmHeaderEntry ← [dsGetAccountDetails]          // whole dataset merged, not named fields
setdataset dsVATHeaderDetails ← {VAT_RATE_TYPE, VAT_TREATMENT_DOCNO}   // note: setdataset = replace
setdataset customevents ← null
```

### Supplier chain (mode `SL`)
Identical, except: `dsVATHeaderDetails` also carries `SUBLEDGER_DOCNO`;
`mode:"SL"`; and the VAT write-back uses **`mergedataset`** (preserving
`DOCTYPE`/`DOCNO`/`DOCDATE`/`AC_DOCNO`) where the GL chain uses **`setdataset`**
(replacing the whole dataset with just the two VAT fields).

⚠️ This `setdataset` vs `mergedataset` asymmetry is real in the source. The GL
chain therefore wipes `dsVATHeaderDetails` down to two fields; the Supplier
chain then rebuilds it. Because GL is always picked before Supplier, the flow
works — but it is fragile. Prefer `mergedataset` in both positions for new code.

### Currency chain
```
fillmultidataset PWA.GetCurrencyRate (strXmlHeader:[DsFrmHeaderEntry]) → dsGetCurrencyDetails
mergedataset DsFrmHeaderEntry ← {PUR_CURR_RATE: "[dsGetCurrencyDetails.currrate]"}
setdataset customevents ← null
```
Note this one passes the **whole header dataset** as `strXmlHeader`, not the
purpose-built `dsVATHeaderDetails`. And the returned field is lowercase
`currrate` (three r's) while every other field on the page is UPPER_SNAKE.

## 7.3 `fapoEvent1` — the shared fragment

Fetched: `filldataset PWA.LoadLayout {doctype:"part", docno:"fapo-event-1"}` → `fapoEvent1`.

```json
[
  { "exec": "fillmultidataset", "args": { "proc": "FAMST.FAPO_TaxCalculateItemWise",
      "dsets": [ { "name": "dsFAPOOnchange", "row": 0, "table": "Table" } ],
      "args": { "strXmlDetails": "[DsFrmDetails1.[this.propkey]]", "strXmlHeader": "[dsVATHeaderDetails]" } } },
  { "exec": "mergedatasetarray", "args": { "dset": "DsFrmDetails1", "index": "[this.propkey]", "data": "[dsFAPOOnchange]" } },
  "[calcformula]"
]
```

🔑 **The fragment references `[this.propkey]` even though it was fetched from
the server with no row context of its own.** It works because the fragment is
spliced into a row control's `onFocusOut` array and evaluated *there* — `this`
binds at the splice site, not the definition site. This is what makes one
server-stored fragment reusable across every row and every page that has a
`DsFrmDetails1`.

The fragment also depends on `dsVATHeaderDetails` being populated (by the GL or
Supplier chain) and on `calcformula` being defined. **Three separate hidden
contracts** — document them whenever you write a `part` fragment.

## 7.4 `dsFetchBeforeEvents` — staged, never invoked in this capture

```json
{ "exec": "setdataset", "args": { "dset": "dsFetchBeforeEvents", "nodeepprocess": true, "data": [
  { "exec": "setdataset", "args": { "dset": "dsFetchHeader", "data": {
      "PREVIOUS_DOCTYPE": "[dsDocumentMaster.DM_PREVIOUS_DOCTYPE]",
      "LOCATION":  "[DsFrmHeaderEntry.PUR_LOC_DOCNO]",
      "DIVISION":  "[DsFrmHeaderEntry.PUR_DIVN_DOCNO]",
      "AC_DOCNO":  "[DsFrmHeaderEntry.PUR_AC_DOCNO]",
      "AC_DESC":   "[DsFrmHeaderEntry.PUR_AC_DESC]",
      "SUBLEDGER_DOCNO": "[DsFrmHeaderEntry.PUR_SUBLEDGER_DOCNO]",
      "SUBLEDGER_DESC":  "[DsFrmHeaderEntry.PUR_SUBLEDGER_DESC]",
      "partyListType": "",
      "DOCTYPE": "[pagemenuinfo.doctype]" } } }
] } }
```

Another stored formula, consumed by the `_globalLayouts.fetchPreviousDoc` widget
(2.4) rather than by anything in this layout — the widget calls
`[dsFetchBeforeEvents]` to populate `dsFetchHeader` before querying. **This is
the contract between a page and a global widget**: the page stages a named
formula the widget knows to invoke. `partyListType:""` is an intentional empty
placeholder the widget fills in.

---

# PART 8 — COMPLETE PROP REFERENCE BY COMPONENT TYPE

Every `type` value and every prop observed on it in this paste.

## `sect` — grid container/cell
`container`, `item`, `xs`, `md`, `className`, `style`, `resolvestyles`,
`resolveprops`, `appendprops`. Fractional `xs` (0.5/1.5) confirmed.

## `div` — plain block
`className` (`entryLabelOuterDiv`, `entryLabelDiv`, `entryValueDiv`), `style`,
`appendprops`.

## `span` — inline text
`props.style` (often `{}`), text in `chld`. Holds either a literal
(`"Sr#"`, `"Document Details"`) or a row expression (`"[this.PURDET_STOCK_DESC]"`).

## `lbl` — label
`text`, `className` (`entryLabel`, `tableAddRowBtnLbl`). Text lives in
`props.text`, **unlike `span`**.

## `icon` — glyph
`className` (`tableAddRowBtnIcon`), icon name in `chld` (`"add"`).

## `entry` — text input
`dset`, `bind`, `label`, `disabled`, `className:"small-ctr"`, `fullWidth`,
`variant:"outlined"` *or* `inputVariant:"outlined"`, `inputProps{maxLength, style}`,
`validation{for, msg, validationParams{minLength}}`.

## `dxnumberbox` — numeric input
`dset`, `bind`, `rowindex`, `className:"small-ctr"`, `fullWidth`,
`format:"##,###.00"` (every instance), `variant:"outlined"`, `includerefdata`,
`disabled`, `label`, `whenchange`, `onFocusOut`.

## `dxnumlbl` — read-only numeric
`dset`, `bind`, `rowindex`, `fullWidth`, `includerefdata`. **No `format` prop
on any instance** — formatting is implicit.

## `dxdtpick` — date picker
`dset`, `bind`, `type:"date"`, `labelMode:"floating"`, `includerefdata`,
`closeOnSelect:true`, `showClearButton:false`, `label:""`, `required:true`,
`inputProps:{}`, `whenchange:[]`.

## `lkup` — lookup/picker
`dset`, `bind`, `required`, `variant:"outlined"`, `className:"small-ctr"`,
`label:""`, `fullWidth`, `icon:"search"`, `includerefdata`, `rowindex`,
`resolveprops`, `validation`, `icons.notempty{icon, whenclick}`, `whenclick`.

## `btn` — text/compound button
`variant` (`"inherit"`), `color` (`"primary"`, `"inherit"`), `size` (`"small"`),
`className` (`tableAddRowBtn`), `style`, `title`, `includerefdata`,
`resolveprops` (node-level **and** prop-level forms both seen), `resolvestyles`,
`appendprops`, `whenclick`. `chld` = string **or** array of nodes.

## `icbtn` — icon button
`color:"primary"`, `className:"facts-icon-color--delete"`,
`style:{"fontSize":"1.3rem"}`, `includerefdata`, `whenclick`; icon name in `chld`.

## `hide` — conditional wrapper
`appendprops.xsUp` (the condition), `container`. Children hidden when the
expression is truthy.

## `list` — repeater
`datasets[]`, `watch[]`, `filtermode`, `filterignorecase`, `satisfy`,
`className`, `content{template:"custom", props, chld}`.

## `view` — sub-layout renderer
`includerefdata`, `contenttype:"json"`, `binding:true`, `content` (expression),
`style`.

---

# PART 9 — REFERENCE TABLES

## 9.1 Datasets

| Dataset | Kind | Role |
|---|---|---|
| `DsFrmHeaderEntry` | record | Document header — all `PUR_*` fields |
| `DsFrmDetails1` | array | Line items — all `PURDET_*` fields |
| `DsFrmDetails2` | array | Additional amounts — all `ADAMTDET_*` fields |
| `dsDocumentMaster` | record | Doctype configuration: `DM_PREVIOUS_DOCTYPE`, `DM_FIXED_LOCATION`, `DM_FIXED_GL_DOCNO`, `HIDE_CURRENCY`, `HIDE_EXCHANGE_RATE`, `HIDE_VAT` |
| `dsPagemode` | record | `pagemode` — `"view"` vs anything else |
| `pagemenuinfo` | record | `doctype` of the current page |
| `dsVATHeaderDetails` | record | Purpose-built payload for tax procs |
| `dsGetAccountDetails` | record | `GetAccountDetails` return |
| `dsGetCurrencyDetails` | record | `GetCurrencyRate` return (`currrate`) |
| `dsFAPOOnchange` | record | `FAPO_Onchange` / `FAPO_TaxCalculateItemWise` return |
| `dsAddress` | record | Generic address fields for the address sub-layout |
| `dsAddressSubview` | layout | Address sub-view layout JSON |
| `dsCountryMasterDeliverTo` | layout | Address layout JSON (`settingsView`) |
| `DsAdditionalAmount` | record | Seed payload for the additional-amount popup |
| `dslist` | control | Lookup-popup configuration |
| `popupinfo` | layout | Currently open popup's layout |
| `dialoginfo` | control | Confirm-dialog state |
| `snackinfo` | control | Toast: `{open, type, message}` |
| `customevents` | actions | Deferred action staging |
| `calcformula` | actions | Stored header-totals formula |
| `fapoEvent1` | actions | Fetched `part` fragment |
| `dsFetchBeforeEvents` | actions | Stored formula for the previous-doc widget |
| `dsFetchHeader` | record | Query payload the above builds |
| `dscurListIndex` | scalar | Parked row index for the delete dialog |
| `_globalLayouts` | layout | App-wide shared layouts (`fetchPreviousDoc`) |

**Naming convention:** `Ds*` (capital D) for form data, `ds*` (lowercase) for
everything else, bare lowercase (`dslist`, `popupinfo`, `dialoginfo`,
`snackinfo`, `customevents`, `pagemenuinfo`) for engine-owned control datasets,
`_`-prefixed for global stores.

## 9.2 Stored procedures

| Proc | Args | Returns → | Used by |
|---|---|---|---|
| `PWA.LoadLayout` | `{doctype, docno}` + `column:"layoutinfo"`, `section` | `popupinfo` / fragment dset | Lookups, popups, `part` fragments |
| `PWA.GetAccountDetails` | `{strXmlHeader, mode:"GL"\|"SL"}` | `dsGetAccountDetails` | GL code, Supplier |
| `PWA.GetCurrencyRate` | `{strXmlHeader}` | `dsGetCurrencyDetails` | Currency |
| `PWA.CountryMasterView_V2` | `{layout_docno, doctype, subViewDset, dataset}` | `dsCountryMasterDeliverTo` | whenload |
| `PWA.countrySubView_V2` | `{layout_docno, dataset}` | `dsAddressSubview` | whenload |
| `FAMST.FAPO_Onchange` | `{strXmlDetails, mode:"details"}` | `dsFAPOOnchange` | Stock/Qty/Rate |
| `FAMST.FAPO_TaxCalculateItemWise` | `{strXmlDetails, strXmlHeader}` | `dsFAPOOnchange` | `fapoEvent1` |

Two namespaces: **`PWA.*`** = generic engine/platform services;
**`FAMST.*`** = module-specific business logic. Put new generic services under
`PWA.`, new purchase-module logic under `FAMST.`.

`fillmultidataset` always declares its targets as
`"dsets": [{"name": "<dset>", "row": 0, "table": "Table"}]` — `row:0` takes the
first row, `table:"Table"` is the default ADO.NET result-table name.

## 9.3 Lookup listtypes

| `listtype` | Title | Writes into |
|---|---|---|
| `FAPOPDIV` | Division | `PUR_DIVN_*` |
| `FAPOUSRLOC` | Location | `PUR_LOC_DOCTYPE` / `PUR_LOCATION_DOCNO` / `PUR_LOC_DESC` |
| `FAPOGL` | GL Code | `PUR_AC_*` |
| `FAPOAP` | Supplier | `PUR_SUBLEDGER_*` |
| `FAPOACCONDET` | Contact | `PUR_CONTACT_*` |
| `FAPOCUR` | Currency | `PUR_CURR_*` |
| `FAPOSMAN` | Staff | `PUR_SMAN_*` |
| `FAPOPAYMOD` | Payment Mode | `PUR_PAYMODE_*` |
| `FAPOSMGT` | Stock Master | `PURDET_STOCK_*` |

All nine share the prefix `FAPO` + an entity mnemonic.

## 9.4 CSS class names

| Class | Applied to |
|---|---|
| `subHeadingLbl` | Section heading `sect` |
| `entryLabelOuterDiv` / `entryLabelDiv` / `entryValueDiv` | Field wrapper trio |
| `entryLabel` | The `lbl` inside `entryLabelDiv` |
| `small-ctr` | Every compact input |
| `factsFullWidth` | Field wrapper `sect` in Delivery Info / Summary |
| `TableParentSect` | Grid outer container |
| `TableHeaderSect` / `TableHeaderChildSect` / `TableHeaderLastChildSect` | Grid header row / cells / last cell |
| `TableListSect` | The `list` node |
| `facts-input-list` | Marks a grid whose rows contain inputs |
| `TableContentChildSect` / `TableContentLastChildSect` | Body cells / last body cell |
| `tableAddRowBtn` / `tableAddRowBtnIcon` / `tableAddRowBtnLbl` | Add-row button parts |
| `facts-icon-color--delete` | Delete `icbtn` |
| `cmlDetailsChildSect` / `cmlDetailsCaptionSect` / `cmlDetailsValueSect` | `##LISTLAYOUT2COLS##` only |

## 9.5 Expression operators confirmed here

Beyond those already in `expressions/binding-expression-dsl.md`:

| Operator | Example from this paste |
|---|---|
| `(*)` | `[this.PURDET_DISCOUNT_PERCENT(*)this.PURDET_GROSS_AMOUNT(/)raw.100]` |
| `(/)` | `[this.PURDET_DISCOUNT_AMOUNT(/)this.PURDET_GROSS_AMOUNT(*)raw.100]` |
| `(+)` | `[DsFrmHeaderEntry.PUR_ITEM_VAT_AMOUNT(+)DsFrmHeaderEntry.ADDITIONAL_VAT_AMOUNT]` |
| `(-)` | `[this.PURDET_GROSS_AMOUNT(-)this.PURDET_DISCOUNT_AMOUNT]` |
| `(listsum)raw.FIELD` | `[DsFrmDetails1(listsum)raw.PURDET_QTY]` |
| `(listcount)(+)raw.1(numformat)raw.0` | `[DsFrmDetails1(listcount)(+)raw.1(numformat)raw.0]` |
| `(ifnull)raw.` (empty literal) | `[DsFrmHeaderEntry.PUR_AC_DOCNO(ifnull)raw.]` |
| chained `(ifnull)…(eq)…` | `[dsDocumentMaster.DM_PREVIOUS_DOCTYPE(ifnull)raw.(eq)raw.]` |
| nested brackets | `[DsFrmDetails1.[dslist.rowindex]]` |
| whole-dataset reference | `"[dsGetAccountDetails]"`, `"[this]"`, `"[dsVATHeaderDetails]"` |
| bare action-list reference | `"[calcformula]"`, `"[fapoEvent1]"` |

Key generalisations:
- **Operators chain left-to-right with no precedence.** `A(*)B(/)raw.100` is
  `(A*B)/100`. Parenthesise by splitting into sequenced actions if you need
  different grouping.
- **A bracket expression naming a dataset with no field yields the whole
  dataset** — used as proc arguments and as `mergedataset` payloads.
- **A bare `"[name]"` string as an *array element* (not a value) is an action
  splice**, not a value substitution.

---

# PART 10 — RULES FOR NEW PAGES

Distilled from everything above. Follow in order.

1. **Root**: copy the `pointerEvents` page-mode `sect` from 1.1. Do not
   re-implement view-mode locking per control.
2. **Sections**: open each with the `subHeadingLbl` heading from 1.3.
3. **Fields**: always the `entryLabelOuterDiv`/`entryLabelDiv`/`entryValueDiv`
   trio. Never place a control directly in a `sect`.
4. **Lookups**: bind `_DESC`, clear both `_DESC` and `_DOCNO` in
   `icons.notempty`, and always either stage or clear `customevents`.
5. **Grids**: `list` + row template. No data-grid widget exists. Keep the `xs`
   values summing to exactly 12 including the actions column.
6. **Per-row events**: `onFocusOut`, not `whenchange`, whenever a server call is
   involved.
7. **Dependent values**: one `mergedataset`/`mergedatasetarray` per dependency
   level — never compute B from A inside the same `data` object.
8. **Deferred contexts** (dialogs, `customevents`): park `[this.propkey]` in a
   dataset first and set `nodeepprocess:true`.
9. **Totals**: define one `calcformula` in `whenload`; invoke `"[calcformula]"`
   from every mutation path — *including deletes* (see the PART 5 bug).
10. **Shared logic**: page-local → `molecule.stored-action-formula`;
    cross-page → `molecule.reusable-event-fragment` (`doctype:"part"`).
11. **Popups**: seed the input dataset, then `filldataset PWA.LoadLayout` into
    `popupinfo` with the right `docno`/`section`.
12. **Address/country UI**: never hand-build it — render
    `[dsCountryMasterDeliverTo.settingsView]` via the `view` atom and seed
    `dsAddress`.

---

# PART 11 — DEFECTS & INCONSISTENCIES RECORDED

Present in the source; listed so they are not mistaken for intentional patterns.

| # | Issue | Location | Impact |
|---|---|---|---|
| 1 | Additional-amount delete omits `"[calcformula]"` | PART 5.2 | Summary totals go stale after deleting an additional-amount row |
| 2 | No divide-by-zero guard on discount maths | 3.8, 6.1 | `Infinity`/`NaN` when gross amount is 0 |
| 3 | `setdataset` (not `mergedataset`) on `dsVATHeaderDetails` in the GL chain | 7.2 | Wipes the dataset to two fields; works only because Supplier runs afterwards |
| 4 | `PUR_LOC_DOCNO` vs `PUR_LOCATION_DOCNO` | 2.2 | Two names for the location code in one page |
| 5 | `appendprops:{"disabled":""}` with no expression | 6.2 | Dead prop |
| 6 | `required:true` on Contact with no `validation` block | 2.2 | Not enforced — `required` is cosmetic |
| 7 | `Telephone-1 *` asterisk typed into the label | 4.2 | Implies required; nothing enforces it |
| 8 | `variant` vs `inputVariant` on `entry` | 2.2 / 4.2 | Inconsistent across the same page |
| 9 | `resolveprops` at node level vs inside `props` | 6.2 | Both forms in one file |
| 10 | `currrate` lowercase amid UPPER_SNAKE fields | 7.2 | Easy to mistype |
