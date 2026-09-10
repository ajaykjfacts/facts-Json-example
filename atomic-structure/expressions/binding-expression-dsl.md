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
