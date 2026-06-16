# Coming from Temporal: Hello World

This guide maps the [`temporalio/samples-go/helloworld`](https://github.com/temporalio/samples-go/tree/main/helloworld) example to this Resonate example. If you already know how Temporal's hello-world works and want to understand what changes in Resonate, start here.

## The pattern

The hello-world pattern is the simplest durable-execution shape: one function runs, produces a result, the caller reads it. In Temporal that means a Workflow that schedules an Activity; in Resonate it is a single Go function registered with the runtime and invoked via a durable promise.

## Side by side

### Temporal (`samples-go/helloworld`)

`helloworld/helloworld.go` — workflow + activity in one file:

```go
package helloworld

import (
	"context"
	"time"

	"go.temporal.io/sdk/activity"
	"go.temporal.io/sdk/workflow"
)

// Workflow is a Hello World workflow definition.
func Workflow(ctx workflow.Context, name string) (string, error) {
	ao := workflow.ActivityOptions{
		StartToCloseTimeout: 10 * time.Second,
	}
	ctx = workflow.WithActivityOptions(ctx, ao)

	logger := workflow.GetLogger(ctx)
	logger.Info("HelloWorld workflow started", "name", name)

	var result string
	err := workflow.ExecuteActivity(ctx, Activity, name).Get(ctx, &result)
	if err != nil {
		logger.Error("Activity failed.", "Error", err)
		return "", err
	}

	logger.Info("HelloWorld workflow completed.", "result", result)

	return result, nil
}

func Activity(ctx context.Context, name string) (string, error) {
	logger := activity.GetLogger(ctx)
	logger.Info("Activity", "name", name)
	return "Hello " + name + "!", nil
}
```

`helloworld/worker/main.go` — registers both on a named task queue:

```go
// c, err := client.Dial(...) — omitted for brevity
w := worker.New(c, "hello-world", worker.Options{})
w.RegisterWorkflow(helloworld.Workflow)
w.RegisterActivity(helloworld.Activity)
err = w.Run(worker.InterruptCh())
```

`helloworld/starter/main.go` — starts the workflow and reads the result:

```go
// c, err := client.Dial(...) — omitted for brevity
workflowOptions := client.StartWorkflowOptions{
    ID:        "hello_world_workflowID",
    TaskQueue: "hello-world",
}
we, err := c.ExecuteWorkflow(context.Background(), workflowOptions, helloworld.Workflow, "Temporal")
// ...
var result string
err = we.Get(context.Background(), &result)
```

### Resonate (this example)

`main.go` — register, run, and read result in a single program:

```go
type GreetArgs struct {
	Name string `json:"name"`
}

func greet(_ *resonate.Context, args GreetArgs) (string, error) {
	return fmt.Sprintf("hello, %s!", args.Name), nil
}

func main() {
	r, err := resonate.New(resonate.Config{
		URL: "http://localhost:8001",
	})
	if err != nil {
		log.Fatalf("resonate.New: %v", err)
	}
	defer func() { _ = r.Stop() }()

	greetFn, err := resonate.Register(r, "greet", greet)
	if err != nil {
		log.Fatalf("Register: %v", err)
	}

	ctx := context.Background()
	id := fmt.Sprintf("hello-%d", time.Now().UnixNano())

	h, err := greetFn.Run(ctx, id, GreetArgs{Name: "world"})
	if err != nil {
		log.Fatalf("Run: %v", err)
	}

	out, err := h.Result(ctx)
	if err != nil {
		log.Fatalf("Result: %v", err)
	}
	fmt.Println(out)
}
```

## Concept mapping

| Temporal | Resonate | Notes |
|---|---|---|
| `workflow.Context` | `*resonate.Context` | Both are durable-execution contexts passed as the first argument |
| `workflow.ActivityOptions{StartToCloseTimeout: ...}` | _(not required)_ | Timeout policy is not a per-call concern at this level in Resonate |
| `workflow.WithActivityOptions(ctx, ao)` | _(not required)_ | No options wrapping needed before scheduling work |
| `workflow.ExecuteActivity(ctx, Activity, name).Get(ctx, &result)` | `greetFn.Run(ctx, id, args)` + `h.Result(ctx)` | `Run` creates the durable promise; `Result` blocks until it is settled |
| `w.RegisterWorkflow(...)` + `w.RegisterActivity(...)` | `resonate.Register(r, "greet", greet)` | One registration call; no workflow/activity type distinction. The function name is the dispatch key. Returns `*RegisteredFunc[A,R]`; its `.Run` returns `*TypedHandle[R]` with `Result(ctx) (R, error)`. For cross-process dispatch use `r.RPC`, which returns an untyped `*Handle` — call `h.Result(ctx, &out)` or `resonate.ResultOf[R](ctx, h)`. |
| `worker.New(c, "hello-world", worker.Options{})` | `resonate.New(resonate.Config{URL: ...})` | Connects to the server at the given URL; no queue name required here |
| `client.StartWorkflowOptions{ID: "...", TaskQueue: "..."}` | `id` string passed to `fn.Run` | The promise ID is the idempotency key; re-running with the same ID attaches to the existing execution |
| Separate starter + worker processes | Single process (in this example) | Resonate supports separate processes too; this example combines them for simplicity |

## Porting it, step by step

1. **Remove the workflow/activity split.** Delete `Workflow` and `Activity` as separate types. Write one Go function — `greet` here — with the signature `func(_ *resonate.Context, args T) (R, error)`. (The `*resonate.Context` argument is optional: the SDK accepts no-args, ctx-only, args-only, or ctx+args.)

2. **Replace `workflow.WithActivityOptions` + `workflow.ExecuteActivity`.** In Temporal the workflow body wraps context with timeout options before calling an activity. In Resonate the function body is the unit of work; call `resonate.Register` once at startup and invoke via `greetFn.Run`.

3. **Replace the task queue with a function name.** `worker.New(c, "hello-world", ...)` ties a Temporal worker to a named queue. `resonate.Register(r, "greet", greet)` ties a Resonate worker to a function name. There is no separate queue declaration.

4. **Replace `StartWorkflowOptions{ID: ...}` with a plain string ID.** Pass the ID directly to `fn.Run(ctx, id, args)`. If you call `Run` again with the same ID before the first execution completes, you attach to the existing promise rather than starting a new one — idempotency is a first-class property, not something to wire up separately.

5. **Replace `we.Get(ctx, &result)` with `h.Result(ctx)`.** The handle returned by `Run` carries the typed result. `h.Result(ctx)` blocks until the promise is settled and returns the value or error.

6. **Point at the Resonate server.** `resonate.New(resonate.Config{URL: "http://localhost:8001"})` replaces `client.Dial(...)`. Run `resonate dev` locally to start a dev server on that port.

## What's different (and why)

**No workflow/activity type distinction.** Temporal models durable execution as two separate concepts: a workflow (the coordinator, runs in a replay-safe context) and an activity (the side-effect unit, runs exactly once per attempt). Resonate collapses this: any registered function runs durably. When you need a function to call a sub-function durably, you use `ctx.Run` inside the parent — the SDK handles the promise chaining. For this hello-world example the distinction doesn't matter in practice, but it becomes relevant when you start nesting calls.

**No activity options required.** Temporal requires at least one of `StartToCloseTimeout` or `ScheduleToCloseTimeout` in `workflow.ActivityOptions` before scheduling an activity. Resonate does not require per-call timeout declarations at this level; the server and worker have their own liveness and task TTL settings.

**The ID is the idempotency key.** Temporal uses two IDs: the Workflow ID (the caller-supplied, stable business/idempotency key) and the Run ID (generated per execution attempt); routing is done by Task Queue. Resonate's single promise ID plays the role of Temporal's Workflow ID — it is the stable identity of the execution. Call `Run` with the same ID twice and the second call returns a handle to the already-running (or completed) promise. This example generates a fresh ID on every run with `time.Now().UnixNano()`; in a real workflow you would derive a stable, meaningful ID from your business domain.

**Single process in this example.** The Temporal sample separates the worker and starter into two `main` packages. This Resonate example registers and invokes in one `main` function. That is a choice made for simplicity here, not an architectural constraint — larger Resonate programs split worker registration and client invocation across processes. On the client side, use `r.RPC` instead of `greetFn.Run`:

```go
h, err := r.RPC(ctx, id, "greet", GreetArgs{Name: "world"})
if err != nil {
    log.Fatalf("RPC: %v", err)
}
// Option A — untyped out-param
var out string
err = h.Result(ctx, &out)
// Option B — generic helper
// out, err := resonate.ResultOf[string](ctx, h)
```

`r.RPC` returns an untyped `*Handle`; `h.Result` takes a pointer to decode into, unlike the typed `h.Result(ctx)` returned by `greetFn.Run`.

## Notes & coverage

- **Task queue vs. function name.** Temporal routes work to workers by matching task queue names. Resonate routes work by matching the registered function name. If you run multiple workers registering the same function name, the server load-balances across them. There is no separate queue to declare or configure.

- **This example pins to a pre-release SDK commit.** `resonate-sdk-go` has no semver tag yet. The `go.mod` in this repo pins a specific commit. Expect API changes before `v0.1.0`.

- **Replay model.** Temporal replays the workflow function from the event history on worker restart. Resonate also re-executes the entire function body from the top on resume (same structural model as Temporal). The difference is the short-circuit mechanism: instead of an event log, Resonate checks the durable promise cache by child ID — children whose promises are already settled are skipped inline. The practical consequence is the same as in Temporal: side effects outside a `ctx.Run` / `ctx.RPC` / `ctx.Sleep` boundary will execute again on every resume. Wrap external mutations (DB writes, emails, payments) in `ctx.Run` so the durable promise records the result and the re-execution path short-circuits them.

## Further reading

- Concept-level guide (all SDKs): https://docs.resonatehq.io/evaluate/coming-from/temporal
- Temporal sample: https://github.com/temporalio/samples-go/tree/main/helloworld
- This example's README
