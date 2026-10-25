# Go TUI Template Examples

## Typed preparation, conditional sections, and nested components

This complete example uses Lip Gloss v2. Use the dependency version already present in the target project. The snapshot contains display inputs, not a live service or the root application model.

```go
package main

import (
	"bytes"
	"fmt"
	"strings"
	"text/template"

	"charm.land/lipgloss/v2"
)

type snapshot struct {
	Running bool
	Target  string
	Notice  string
}

type detailsData struct {
	Target string
}

type viewData struct {
	Title   string
	Status  string
	Details detailsData
	Notice  string
	Footer  string
}

var titleStyle = lipgloss.NewStyle().Bold(true)
var mutedStyle = lipgloss.NewStyle().Faint(true)

const screenTemplateText = `{{define "details"}}Target: {{.Target}}{{end}}{{.Title}}
Status: {{.Status}}
{{template "details" .Details}}{{if .Notice}}
{{.Notice}}{{end}}
{{.Footer}}`

var screenTemplate = template.Must(template.New("screen").Parse(screenTemplateText))

func prepareViewData(state snapshot) viewData {
	status := "Stopped"
	if state.Running {
		status = "Running"
	}
	return viewData{
		Title:   titleStyle.Render("Connection"),
		Status:  status,
		Details: detailsData{Target: state.Target},
		Notice:  state.Notice,
		Footer:  mutedStyle.Render("Esc: back"),
	}
}

func renderView(data viewData) (string, error) {
	var output bytes.Buffer
	if err := screenTemplate.Execute(&output, data); err != nil {
		return "", fmt.Errorf("render screen: %w", err)
	}
	return output.String(), nil
}

func main() {
	state := snapshot{Running: true, Target: "localhost:8080"}
	withoutNotice, err := renderView(prepareViewData(state))
	if err != nil {
		panic(err)
	}
	if strings.Contains(withoutNotice, "\n\n") {
		panic("absent notice introduced an empty row")
	}
	state.Notice = "Received literal {{.Status}} from upstream"
	withNotice, err := renderView(prepareViewData(state))
	if err != nil {
		panic(err)
	}
	if !strings.Contains(withNotice, state.Notice) {
		panic("notice was not preserved literally")
	}
	fmt.Println(withNotice)
}
```

The newline before the notice is inside its `if`, so an absent notice introduces no extra row. The template owns section order and separators. Data fields do not carry hidden joining newlines. The example's panic calls are executable-example checks, not a prescribed production rendering-error policy.

The example intentionally omits terminal bounds to isolate template behavior. For a real bounded screen, allocate and fit the blocks before rendering as described below.

## Connect templates to allocated areas

Use the existing layout implementation. The following is a boundary sketch, not a second layout engine or an API to add wholesale:

```text
terminal area
  -> subtract actual outer frame dimensions
  -> prepare and style title, notice, and footer
  -> measure fixed blocks at the inner width
  -> budget separators declared by the template
  -> assign the remaining area to the active child
  -> resize the child and obtain its bounded body
  -> fit all blocks to their allocated areas
  -> execute the template with the prepared values
  -> render the outer frame
```

For already bounded blocks, the parent template can expose their placement directly:

```gotemplate
{{.Header}}
{{.Body}}{{if .Notice}}
{{.Notice}}{{end}}
{{.Footer}}
```

Here `Header`, `Body`, and `Footer` are present blocks without joining newlines; `Notice` is empty when absent. A single joining newline connects the last row of one block to the first row of the next: it does not add an empty row beyond the sum of their measured heights. Two newlines introduce one additional blank row, which layout must budget. If a block can disappear because its allocated area is empty, make its placement conditional too and measure the resulting separators consistently. Do not blindly use this fixed shape for zero-sized areas.

Measure wrapped, styled block content rather than raw text. A nested horizontal header may use Lip Gloss `JoinHorizontal` for alignment, then be passed as `Header`; the parent template still declares where that header belongs. The child can use its own template without exposing its internal state to the parent.

Keep ANSI-aware truncation and overflow policy in layout. Applying `.Width()` or `.Height()` to a style is not a substitute for an explicit bound when content can overflow. Check the final output, including the template's separators and the outer frame, at small sizes.

## Return the result through Bubble Tea

Keep rendering and framework adaptation small. Preserve the project's existing failure policy and view options rather than returning a partially rendered result.

- Bubble Tea v1: pass the rendered string back from `View() string`.
- Bubble Tea v2: construct `tea.NewView(rendered)` in `View() tea.View`, preserving options such as alternate-screen mode and the window title.

If preparation changes pagination or input sizing, perform it during initialization/update handling and store the presentation for `View` to read. Template execution must not execute `tea.Cmd`, change navigation, or call application services.
