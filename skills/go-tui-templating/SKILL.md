---
name: go-tui-templating
description: Design or refactor declarative Go terminal views with standard text/template, Bubble Tea, and Lip Gloss. Use when screen structure should be visible in templates while presentation data, styling, and bounded layout remain separate.
---

# Go TUI Templating

Make the visual hierarchy readable in a template: headings, content, optional messages, and help should appear in their display order. Use Go's standard `text/template`; do not introduce a custom template engine or DSL. Apply this approach to root views, individual screens, and substantial nested components without requiring a particular application architecture.

## Keep responsibilities distinct

- **Application/event handling:** owns operations, navigation, cancellation, mutable application state, and snapshots. Templates must not call services or change state.
- **Presentation preparation:** a function such as `prepareViewData` turns snapshots and component output into a typed presentation struct. Compute status labels, formatted values, display flags, and message text here.
- **Layout:** determines available areas, measures styled content, and allocates width and height. Parents allocate areas; children render within them. Keep geometry calculations out of templates.
- **Styles:** Lip Gloss styles supply borders, padding, margins, colors, and alignment independently of business data.
- **Template execution:** describes semantic structure, repetition, and optional sections, then writes the prepared representation to a buffer.

These are logical responsibilities, not a requirement for five packages, interfaces, or forwarding methods. Use a few concrete types and functions near the view when sufficient. Preparation may orchestrate measurement and rendering without moving their algorithms into the template.

## Describe structure in templates

Use named fields such as `Title`, `Status`, `Body`, `Messages`, and `Footer`. An unexported `viewData` type can have exported fields accessible to the template. Prefer typed nested structs and slices over an unstructured map when the shape is known.

Use `{{if .Message}}`, `{{range .Items}}`, and `{{template "component" .Component}}` for presentation decisions. Compute business decisions and meaningful booleans before execution. Keep template functions limited to presentation needs; do not expose model methods, services, or arithmetic helpers for layout.

Do not hide the hierarchy inside a single preassembled `.Content` field or loops that concatenate all screen sections in Go. A `.Body` containing an independently rendered child is appropriate when the parent declares where that child belongs. Retain polymorphic calls to active child views rather than inspecting their concrete types to render them.

Use Lip Gloss `JoinHorizontal` or `JoinVertical` when they implement a real geometric arrangement, such as aligned columns with different heights. Do not use chains of joins, `fmt.Sprintf`, or concatenation as a substitute for the template's semantic screen structure. Formatting a scalar value in Go is appropriate.

## Parse once and execute deliberately

For a static programmer-owned template, parse once with `template.Must(template.New("screen").Parse(source))`. A parse panic then identifies an invalid static definition during initialization. Do not reparse the same template for each Bubble Tea frame.

Execute using a local `bytes.Buffer` and check the error. `template.Must` handles parsing errors only. Follow the application's established rendering-error policy; never silently display a partially written buffer. For runtime-supplied templates, return parse errors instead of treating them as impossible.

Values passed to `Execute` are literal data: braces inside an error message or child body are not recursively evaluated. Use `text/template`, not `html/template`, so terminal text and ANSI styles are not HTML-escaped. Terminal control characters in untrusted text still need the application's normal sanitization before styling; template execution does not sanitize them.

## Integrate bounded layout

1. Read terminal dimensions from the existing Bubble Tea event flow. Subtract actual frame decoration using the applicable Lip Gloss frame-size API.
2. Prepare and style fixed content, then measure it at its available width. Include wrapping, borders, padding, margins, and template separators in the measurement. Use terminal-cell measurements such as `lipgloss.Width` and `lipgloss.Height`, not byte length.
3. Allocate fixed blocks their measured space and flexible blocks the remainder. Pass the allocated area to each child; apply the existing clipping, scrolling, or pagination policy when content does not fit.
4. Prepare bounded block values and execute a template that places them in the same structure the layout measured. Account for template-owned newline separators exactly once. Do not append hidden separator newlines to every data field and also add them in the template.
5. Apply the outer frame after assembly if that is how the layout budget was calculated. Verify the final styled output fits the terminal; Lip Gloss width or height styling alone is not a guarantee of clipping arbitrary content.

Do not replace measured content with fixed reservation constants. Recalculate when dimensions or visible content change, including notices, errors, busy states, and statistics. Avoid a second hand-maintained ordering of sections that can drift from the template; share prepared section data where useful without building a layout DSL.

Preserve existing event-loop semantics. If sizing changes widget state such as pagination or input overflow, prepare those changes in initialization/update processing before the next input event. Keep `View` observational; do not move operations or state changes into it merely to prepare a template. Caching presentation is optional and must be invalidated for every relevant event if used.

## Reuse at the appropriate boundary

A root template can place an active screen's rendered body. A screen can use its own template for local fields and hints. A repeated nested component can use `{{define}}` and `{{template}}` within a parsed template set. Simple scalar labels do not need their own templates, interfaces, or files.

Read [the worked examples](references/examples.md) for typed data preparation, conditional sections, nested templates, execution, and the boundary between allocated areas and rendered blocks. The examples do not require copying a project-specific layout package.

Inspect the project's dependency versions before adapting framework examples. Bubble Tea v1 returns a string from `View`; v2 returns `tea.View`, commonly constructed with `tea.NewView(rendered)`. Preserve view options and command handling. Do not mix import paths or upgrade dependencies solely to introduce templates.

## Verify the representation

Test observable output: section order, optional sections without accidental blank rows, literal template delimiters in data, and error handling. For bounded views, cover narrow and short areas, long and multiline text, ANSI styling, and wide Unicode characters. Check focus and pagination after resize when widget sizing is involved.

Check that the template still communicates the screen hierarchy after preparation is extracted. Reject changes that merely move the old concatenation into `prepareViewData`, hide conditional structure in preassembled strings, calculate geometry in the template, or introduce registries and generic renderers without a concrete need.
