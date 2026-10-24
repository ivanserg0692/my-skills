# Go Behavior Contracts: Worked Examples

Each Go block below is a complete, independent package. The examples use only the standard library and do not require a particular framework. Read the relevant section rather than treating every example as a required architecture.

## Consumer-owned interfaces, DI, DIP, and composition

A reporting component needs a way to write output. It owns the contract; its caller chooses the implementation. In a multi-package application, `Writer` and `Report` belong to the consuming feature, while the file adapter and wiring can live elsewhere. This one-package example keeps the mechanics visible.

```go
package reporting

import (
    "bytes"
    "io"
    "os"
)

type Writer interface {
    Write([]byte) (int, error)
}

// Output combines capabilities only for a consumer that needs both.
type Output interface {
    Writer
    Flush() error
}

type Report struct {
    writer Writer
}

func NewReport(writer Writer) *Report {
    return &Report{writer: writer}
}

func (r *Report) Render(text string) error {
    data := []byte(text)
    n, err := r.writer.Write(data)
    if err == nil && n != len(data) {
        return io.ErrShortWrite
    }
    return err
}

type fileOutput struct {
    file *os.File
}

func (o *fileOutput) Write(data []byte) (int, error) {
    return o.file.Write(data)
}

func (o *fileOutput) Flush() error {
    return o.file.Sync()
}

var _ Output = (*fileOutput)(nil)
var _ Writer = (*bytes.Buffer)(nil)

func RenderToFile(path, text string) error {
    file, err := os.Create(path)
    if err != nil {
        return err
    }
    output := &fileOutput{file: file}
    renderErr := NewReport(output).Render(text)
    if renderErr == nil {
        renderErr = output.Flush()
    }
    closeErr := file.Close()
    if renderErr != nil {
        return renderErr
    }
    return closeErr
}
```

`Report` depends only on writing, even though `fileOutput` provides flushing too. This is ISP; forcing `Report` to accept `Output` would enlarge its dependency unnecessarily. Constructor injection is DI. The consumer-owned behavior contract supports DIP when the package graph follows that boundary. Choosing `fileOutput` or `bytes.Buffer` supplies interface polymorphism. `Output` demonstrates interface composition without imposing it on every consumer.

In ordinary code with exactly the standard writing contract, use `io.Writer` rather than redeclaring it. The local declaration illustrates implicit satisfaction: neither implementation needs an `implements` declaration. The methods on `fileOutput` adapt an owned resource and establish a capability boundary; their short length alone does not make them disposable.

## Replace a synchronous request hop with a focused dependency

Suppose a component currently returns a save request, which a caller immediately unpacks and dispatches to an application coordinator. If that request has no queueing or recording role, the component can invoke a consumer-owned interface directly.

```go
package editor

import (
    "context"
    "errors"
)

type Document struct {
    Text string
}

// Task is returned now and executed later by the application's scheduler.
type Task func(context.Context) error

type SaveRequester interface {
    SaveDocument(Document) Task
}

type Editor struct {
    draft Document
    actions SaveRequester
}

func NewEditor(actions SaveRequester) *Editor {
    return &Editor{actions: actions}
}

func (e *Editor) SetText(text string) {
    e.draft.Text = text
}

func (e *Editor) Save() Task {
    return e.actions.SaveDocument(e.draft)
}

type Store interface {
    Write(context.Context, Document) error
}

type application struct {
    store Store
    busy bool
}

func (a *application) SaveDocument(document Document) Task {
    // This state change occurs synchronously when the editor requests a save.
    a.busy = true
    return func(ctx context.Context) error {
        // Resource effects occur only when the caller executes the task.
        if err := ctx.Err(); err != nil {
            return err
        }
        return a.store.Write(ctx, document)
    }
}

var _ SaveRequester = (*application)(nil)

type memoryStore struct {
    document Document
}

func (s *memoryStore) Write(_ context.Context, document Document) error {
    s.document = document
    return nil
}

func ExampleFlow() error {
    store := &memoryStore{}
    app := &application{store: store}
    editor := NewEditor(app)
    editor.SetText("draft")
    task := editor.Save()
    if !app.busy || store.document.Text != "" {
        return errors.New("save request must mark busy before executing storage")
    }
    err := task(context.Background())
    // This example runs the task inline. An event-loop application would
    // process a completion message before changing UI state instead.
    app.busy = false
    return err
}
```

The editor sees neither the application's concrete type nor its store. The application owns save coordination, and the editor owns its draft. `Task` represents deferred execution, not an adapter that hides request dispatch. This flow intentionally distinguishes a synchronous method call from an asynchronous message protocol; it does not prescribe background execution or completion handling.

When adapting an existing event loop, preserve its input gates, completion messages, cancellation, and order of navigation. Do not mutate event-loop state inside a background task just to remove a dispatcher.

`Editor.Save` is an intentional component action, exposed to callers that should not access its private draft or dependency. In contrast, a coordinator method `SaveDocument(d) { return a.save(d) }` can often be removed by renaming the private implementation to `SaveDocument` and updating its callers. Keep a separate method if it adds validation, navigation, or different error semantics.

## Method sets and typed nil values

```go
package contracts

import "errors"

type Closer interface {
    Close() error
}

type connection struct {
    closed bool
}

func (c *connection) Close() error {
    c.closed = true
    return nil
}

var _ Closer = (*connection)(nil)

// A connection value does not implement Closer: Close has a pointer receiver.
// An addressable connection variable can still use value.Close() syntax.

func absentConnection() Closer {
    return nil
}

func CheckNil() error {
    var pointer *connection
    var value Closer = pointer
    if value == nil {
        return errors.New("typed nil unexpectedly became a nil interface")
    }
    // Calling value.Close() here would dereference a nil pointer.
    if absentConnection() != nil {
        return errors.New("absence should be returned as a nil interface")
    }
    return nil
}
```

A compile-time assertion establishes method-set compatibility, not non-nil availability or behavioral correctness. Constructors should make required dependencies explicit. Optional dependencies need a documented absence convention; do not add reflection-based nil checks to every constructor.

## When a concrete type, function, or switch is simpler

Use a concrete value when the implementation itself is the useful concept, such as a local parser with no substitutable behavior boundary. Inject a function for a single independent operation, such as obtaining the current time; there is no need to manufacture a clock hierarchy unless consumers need a larger protocol.

Keep a short switch at an external message boundary when its cases adapt framework events to application behavior. Replace recurring variant dispatch when components can own the different behavior behind a meaningful contract. A registry plus one handler struct per case can be more complex than the original switch.

Do not merge methods solely because their bodies are short: `Refresh()` that clears an error before loading and `Load()` used during initialization have different synchronous effects. Redirecting initialization through refresh silently changes behavior. Conversely, a private `updateChild(msg)` that only returns `child.Update(msg)` can usually be inlined without losing a boundary.
