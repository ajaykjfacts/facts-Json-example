# Atomic Structure Library — PWA JSON Layout Engine

This library is the reusable building-block catalog for this project's custom
PWA JSON layout engine (React + DevExpress). It was extracted directly from
the 128 existing layout files in the project root — every atom/molecule/section
in this folder cites the real file(s) it was observed in. Nothing here was
invented; where a pattern did not exist in the codebase, it was left out.

Use this library BEFORE writing any new page JSON. Follow the reuse order:

    ATOM exists?        -> reuse it
    MOLECULE exists?     -> reuse it
    SECTION exists?       -> reuse it
    Nothing close exists?  -> compose a new one FROM these atoms, don't invent
                              a new component type or property.

## Folder layout

- `atoms/` — smallest reusable component patterns (one PWA `type` each: `lbl`,
  `entry`, `dxnumberbox`, `dxdtpick`, `lkup`, `icbtn`, `btn`, `hide`, `sect`).
- `actions/` — the executable action vocabulary used inside `whenclick`,
  `whenchange`, `whenload`, `wheninit`, `onChange`, `onFocusOut` arrays
  (every distinct `exec` value found in the codebase).
- `expressions/` — the bracketed binding-expression mini-language
  (`[dataset.field(op)raw.value]`) used in `appendprops`, `verify.ref`,
  `bind` text, and list-row `this.*` context references.
- `molecules/` — atoms combined into a meaningful unit: a labeled field, a
  lookup field with clear button, a grid row template, a row delete/edit
  action, a required-field validation, a per-row grid validation.
- `sections/` — molecules combined into a page area: a form section, a
  line-items grid section, a filter bar section.
- `page/` — the two real page-root shapes found in the codebase (bare layout
  array vs. full page object with `title`/`wheninit`/`whenload`/`appbar`).
- `index.json` — machine-readable catalog of every entry above (id, level,
  componentType, description, file path, source references) so a new task
  can search "does this already exist?" without re-scanning all 128 files.

## Key findings that shape this library

1. **Field pattern** — `entryLabelOuterDiv > entryLabelDiv > lbl` +
   `entryValueDiv > control` is the single most repeated structure in the
   codebase (~1700 occurrences). It is captured as `atoms/field-label-wrapper.json`
   + `atoms/field-value-wrapper.json`, and pre-composed per control type in
   `molecules/field-*.json`.
2. **No DevExpress data-grid widget is used anywhere.** Line-item "grids" are
   built from `type: "list"` repeating a `sect`/`div`/control row template
   per dataset row (see `molecules/grid-row-template.json`). The only real
   DevExpress grid-shaped component in the whole codebase is a single
   `dxpivotgrid` (report use case, not captured here as a general atom since
   it has exactly one usage site).
3. **Validation is action-based, not declarative.** There is no
   `required`/`datatype`/`length` schema; validation runs as
   `validatedataset` (single field) or `forLoop` + `validatedataset` (per grid
   row) action lists that push messages into the `snackinfo` dataset. See
   `actions/validatedataset.json` and `molecules/validation-*.json`.
2. **One binding DSL powers both conditional visibility and validation** —
   `appendprops.xsUp` (show/hide) and `verify.ref` (validation) both use the
   same `[dataset.field(op)raw.value]` bracket syntax. See
   `expressions/binding-expression-dsl.md`.
5. **Two page-root shapes exist**: a bare `sect` array (`*-general-sect.json`
   files) and a full page object with `title`/`wheninit`/`whenload`/`appbar`/
   `layout` (dashboards, `wf-print.json`, `ppr-ppr-general-sect.json`). Both
   are captured in `page/`.

## Template placeholder convention

Every `template` block in this library is valid JSON. Values that must be
filled in per use are written as `"{{PARAM_NAME}}"` string placeholders (or,
where the real value is itself an array/object, a comment field next to it
explains what to substitute). Replace every `{{...}}` before using a template
in a real page — never ship a placeholder into production JSON.
