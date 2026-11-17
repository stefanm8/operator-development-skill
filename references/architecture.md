# Architecture

Read this reference when adding a controller, wiring dependencies, organizing packages, or reviewing an operator process. Preserve the project's established layout; these defaults describe where responsibilities belong.

## Layout

A new operator, or a new controller in an existing one, can use:

```text
cmd/<binary>/                   Process entry: signal handling, then Run(ctx, args)
<app>/                          Composition root: configuration, clients, wiring, server or manager
pkg/<runtime-adapter>/          Request decoding, response encoding, outcome mapping, condition timestamps
pkg/k8s/{names,labels,conditions}/
                                Leaf helpers with no project imports
pkg/<domain>/                   Desired-state functions, child constructors, selection, contracts
pkg/<domain>/<integration>/     External client and sync steps
crds/ or config/crd/            One CRD per file
```

- Dependencies point inward: composition root → adapters and domain → leaf helpers.
- Group code by domain and integration, not by technical layer. A package should not import a sibling integration only for a constant; move the contract to the package that owns it.
- Do not create empty layers or wrappers for anticipated reuse. Extract shared code when a second consumer exists.

## Adapters

- Share outcome mapping and condition-timestamp handling across adapters; only decoding and encoding differ by controller shape.
- In Go, a function type with a method that implements the runtime's interface, such as an HTTP handler, lets the desired-state function itself be registered. Other languages can use a wrapper or decorator to the same effect.
- For HTTP hooks, bound request bodies and return a client error for undecodable input.
- When migrating to a better adapter, move every caller and delete the old one. Two adapters with different status semantics produce controllers that disagree.

## Dependencies and wiring

- Build each controller through a factory that takes typed configuration and the narrowest clients it needs, does one-time setup, and returns the handler or reconciler.
- Keep credentials in transports, token sources, or client constructors. Wrap the provided inner transport rather than a global default, and do not mutate the caller's request.
- Use functional options when a constructor has meaningful defaults, applying the defaults after the options.
- Pass the process context into client construction and every outbound call. Do not store a context in a long-lived struct.
- Wire controllers with direct constructor calls on the router or manager, so every route or watch is greppable in one place. Keep tenant identifiers, URLs, project IDs, and namespaces in configuration. If a registration table is needed, detect duplicate registrations at startup.
- Read secrets from environment variables or mounted files. Do not use secret values as command-line flag defaults, where help output prints them. Keep flag and variable names consistent.

## Process hygiene

- Keep `main` small: create a context cancelled on interrupt and termination signals, call the run function, and exit non-zero on error. Kubernetes stops pods with SIGTERM.
- Make readiness reflect real dependencies and keep liveness cheap. Exclude probe endpoints from access logs.
- For HTTP hooks, set server read, write, and idle timeouts, keep the write timeout above the slowest external call a pass can make, and drain within a bounded time on shutdown.
- Install one structured logger in the composition root, make it the default, and expose a level setting. Adapters derive a per-request logger carrying the object's kind, namespace, name, and generation. Use consistent keys and log each request once.
- Use least-privilege RBAC: no wildcards, and no list or watch on Secrets unless required. Use the runtime's leader election, metrics, and events where it provides them.

## Naming

- Name factories by role: `ComposeX` or `ReconcileX` for desired-state functions, `DecorateX` for decorators, `SelectX` for related-object selection, and `NewX` for pure child constructors.
- Name one-method interfaces by their method with an `-er` suffix, and add an `XFunc` adapter when plain functions must satisfy one.
- Prefix contract constants by kind, such as `LabelX`, `AnnotationX`, `ConditionTypeX`, and `ConditionReasonX`, and declare group/version/kind identifiers once.
- Do not shadow standard or widely used package names with project packages or locals.

## Code conventions

- Prefer guard clauses and early returns. Split a long desired-state function into child construction, status computation, and cleanup.
- Wrap errors with context while preserving their identity. Use typed or sentinel errors for conditions callers must detect; do not match error strings.
- Delete dead code, such as unused helpers, fakes, and superseded adapters, when you work in the area.
- When changing continuous integration, run vet, a static analyzer, and the race detector, and check that dependency manifests are tidy.
