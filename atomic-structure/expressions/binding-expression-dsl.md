# Binding Expression DSL

A single bracketed mini-language is used throughout the codebase for dynamic
values, conditional visibility, dynamic styles, and validation checks. It
appears inside: `props.text` (labels), `appendprops.xsUp` (hide/show),
`appendprops.style.*` (dynamic styles), `verify.ref` (validation), `bind`
context in list row templates (`this.*`), and directly inside action `args`.

Do not invent new operators. Only the ones observed below are confirmed to
exist in this codebase.

## Syntax

```
[<source>.<field>(<operator>)<value>]
```

- `<source>` — a dataset name (e.g. `dsForm`, `DsFrmHeaderEntry`, `dsPagemode`),
  `func` (built-in function call), or `this` (current row context inside a
  `list` row template, or the current data context inside `resolveprops`).
- `<field>` — the field/property name on that source.
- `(<operator>)` — one of the operators below.
- `<value>` — a literal prefixed `raw.` (e.g. `raw.approved`, `raw.0`), another
  bracketed expression, or a nested `func.[...]` call.

## Confirmed operators (from real usage)

| Operator | Meaning | Example |
|---|---|---|
| `eq` | equals | `[dsPagemode.pagemode(eq)raw.view]` |
| `neq` | not equals | `[dsDocumentMaster.showDocno(neq)raw.1]` |
| `<` `>` | numeric comparison | `[DsFrmDetails1(listcount)(eq)raw.0]` combined patterns |
| `and` | logical AND, combining two bracketed expressions | `[func.[...](and)func.[...]]` |
| `ifnull` | fallback when null | `[DsFrmHeaderEntry.PRQ_DIVN_DOCNO(ifnull)raw.]` |
| `iftrue` / `iffalse` | ternary branches after a comparison | `[dsPagemode.pagemode(eq)raw.view(iftrue)raw.none(iffalse)raw.all]` |
| `listcount` | count of a dataset/list | `[DsFrmDetails1(listcount)(numformat)raw.N0]` |
| `listfilter` | filter a dataset by a condition | `[DsFrmDetails1(listfilter)raw.row_error=>error(listcount)]` |
| `listsum` | sum a field across a dataset | see PWA Functions Reference |
| `numformat` | number formatting | `[dsForm.total(numformat)raw.N2]` / `raw.N0` |
| `dtformatntz` | date formatting (no timezone) | `[func.today(dtformatntz)raw.yyyy-MM-dd HH:mm:ss]` |
| `get` | property access on an object value | see PWA Functions Reference |
| `*` | multiply | `[this.PURDET_DISCOUNT_PERCENT(*)this.PURDET_GROSS_AMOUNT(/)raw.100]` |
| `/` | divide | `[this.PURDET_DISCOUNT_AMOUNT(/)this.PURDET_GROSS_AMOUNT(*)raw.100]` |
| `+` | add | `[DsFrmHeaderEntry.PUR_ITEM_VAT_AMOUNT(+)DsFrmHeaderEntry.ADDITIONAL_VAT_AMOUNT]` |
| `-` | subtract | `[this.PURDET_GROSS_AMOUNT(-)this.PURDET_DISCOUNT_AMOUNT]` |

## Arithmetic — the four rules

Confirmed from `examples/fapo-fapi-full-breakdown.md` (sections 3.8, 6.1, 7.1):

1. **Operators chain left-to-right with NO precedence.** `A(*)B(/)raw.100`
   evaluates as `(A*B)/100`. There is no parenthesis syntax — to group
   differently, split the calculation across sequenced actions.
2. **Both operands may be field references.** A `raw.` literal is not required
   on either side: `[DsA.FIELD1(+)DsA.FIELD2]` is valid.
3. **Long chains are legal.** The net-amount formula chains five terms:
   `[DsFrmDetails1(listsum)raw.PURDET_AMOUNT(+)DsFrmHeaderEntry.PUR_OTHER_AMOUNT(+)DsFrmHeaderEntry.PUR_ITEM_VAT_AMOUNT(+)DsFrmHeaderEntry.ADDITIONAL_VAT_AMOUNT(-)DsFrmHeaderEntry.PUR_DISCOUNT_AMOUNT]`
4. **No division guard.** Nothing protects `(/)` against a zero denominator;
   add guards deliberately where needed.

⚠️ **Sequencing rule (the most common source of wrong totals):** expressions
inside a single `mergedataset`/`mergedatasetarray` `data` object all evaluate
against the **pre-merge** state. If field B is derived from field A that the
same action writes, B reads the *old* A. Split into two sequenced actions.

## Whole-dataset and whole-row references

A bracket expression naming a source with **no field** resolves to the entire
dataset/row object, not a scalar:

| Form | Resolves to | Typical use |
|---|---|---|
| `"[DsFrmHeaderEntry]"` | the whole header record | stored-proc `strXmlHeader` argument |
| `"[dsGetAccountDetails]"` | the whole returned record | `mergedataset` `data` payload |
| `"[this]"` | the current row, inside a row control | stored-proc `strXmlDetails` argument |
| `"[dsVATHeaderDetails]"` | a purpose-built payload record | stored-proc argument |

This is what allows a stored proc to add an output field without any layout
change — the merge payload is the whole returned record.

## Nested bracket expressions

An expression may contain another expression, resolved innermost-first:

```
"[DsFrmDetails1.[dslist.rowindex]]"        → row N of DsFrmDetails1
"[raw.facts-row-[this.row_error]]"          → className string built from a row field
```

## Empty-string literals and the "is blank" idiom

`raw.` with nothing after it is a valid **empty-string literal**. Combined with
`ifnull` it forms the standard emptiness test used in validation:

```
"[DsFrmHeaderEntry.PUR_AC_DOCNO(ifnull)raw.]": ""
```
reads as *"coalesce the field to empty string, then compare against empty"*.

Operators can chain past it too —
`[dsDocumentMaster.DM_PREVIOUS_DOCTYPE(ifnull)raw.(eq)raw.]` coalesces, then
compares, yielding a boolean for a `hide`.

## Bare `[name]` as an ARRAY ELEMENT — action splicing

A bracket expression used as a **string element inside an action array** (rather
than as a property *value*) is not a value substitution — it splices in a stored
action list:

```json
"onFocusOut": [
  { "exec": "fillmultidataset", "args": { ... } },
  { "exec": "mergedatasetarray", "args": { ... } },
  "[fapoEvent1]"
]
```

The referenced dataset holds an action array (put there by
`setdataset ... nodeepprocess:true`, or fetched via `PWA.LoadLayout` with
`doctype:"part"`). Splices may nest — `fapoEvent1` itself ends with
`"[calcformula]"`. See `molecules/stored-action-formula.json` and
`molecules/reusable-event-fragment.json`.

🔑 **Binding site, not definition site.** A spliced fragment containing
`[this.propkey]` binds `this` where it is spliced in, which is what lets one
server-stored fragment serve every row of every page.

## `nodeepprocess` — store raw instead of resolving now

`{"exec":"setdataset","args":{"dset":"X","nodeepprocess":true,"data":[...]}}`
stores the payload **without resolving any expressions inside it**. Required
whenever the payload is itself an action list to be run later:

- stored formulas (`calcformula`, `dsFetchBeforeEvents`)
- deferred actions (`customevents`)
- dialog button handlers (`dialoginfo`) — without it, `[dscurListIndex]` and
  `[calcformula]` would resolve while the dialog is being *built* rather than
  when YES is clicked

Corollary: inside such a deferred payload the row context is gone, so
`[this.propkey]` is unavailable. Park the index in a scratch dataset first
(`dscurListIndex`) or in the `dslist` payload (`rowindex`), and read it back.

## Usage patterns confirmed in this codebase

**Conditional visibility** (`atoms/hide-conditional.json`):
```
"appendprops": { "xsUp": "[dsPagemode.pagemode(eq)raw.view]" }
```

**Dynamic style** (`molecules/dynamic-style-by-pagemode.json`):
```
"appendprops": { "style": { "pointerEvents": "[dsPagemode.pagemode(eq)raw.view(iftrue)raw.none(iffalse)raw.all]" } }
```

**Validation condition** (`molecules/validation-required-field.json`):
```
"verify": { "ref": { "[DsFrmHeaderEntry.PRQ_DIVN_DOCNO(ifnull)raw.]": "" }, ... }
```

**Row context inside a `list` template** — `this` refers to the current row:
```
"rowindex": "[this.propkey]"
"className": "[raw.facts-row-[this.row_error]]"
"data": { "TDET_DOCSRNO": "[DsFrmDetails1(listcount)(+)raw.1(numformat)raw.0]" }
```

**Dynamic label with live counts**:
```
"text": "[raw. Grouped: [dsGroupedList(listcount)(numformat)raw.N0], Filtered: [dsFilteredList(listcount)(numformat)raw.N0]]"
```

For calculation/string/list/date functions beyond what's listed here (e.g.
`datediff`, `strreplace`, `listsort`, `roundnumber`), consult the project's
PWA Functions Reference document — this file only documents operators that
were directly observed in the existing JSON files.
