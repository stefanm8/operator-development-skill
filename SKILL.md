---
name: operator-development
description: Kubernetes operator and controller design, reconciliation, CRD API, and verification standards. Use when writing, reviewing, or refactoring Kubernetes controllers or operators (controller-runtime, kubebuilder, client-go, kopf, or hook-based sync controllers), reconcile loops, CRD types and schemas, status conditions, finalizers, or code that keeps external systems in step with Kubernetes objects. Not for Helm or Kustomize packaging or cluster troubleshooting.
---
# Operator Development Guidelines

Build operators whose reconciliation is deterministic, whose parts compose, and whose failures are visible and recoverable. These defaults apply to reconciler frameworks, hand-rolled informer loops, event-handler frameworks, and hook-based declarative controllers that receive observed state and return desired state. Apply them within the project's established runtime and structure. A pass is one run of a controller's logic for one object, whatever triggered it.

## Discover the project

Before changing reconcile behavior, establish:

- **The runtime contract.** How your code is invoked and what the runtime does with your output. Is a pass declarative (you return the full desired set and the runtime applies and prunes) or imperative (you apply each object and prune yourself)? How are requeues, retries, terminal failures, and cleanup expressed? Is status replaced or patched? If it is replaced, every path must return all the status it wants kept.
- **The existing shapes.** Which controllers exist, which shape each one is, and how the runtime's request reaches domain code.
- **Shared helpers.** Name generation, label and annotation contracts, condition handling, readiness, and external API clients.
- **Wiring and tests.** Where controllers are constructed and registered, how credentials reach clients, where CRDs and generated code live, and how tests fake external systems.

Follow one existing controller end to end before introducing a pattern. When the project carries two generations of the same mechanism, such as two adapters, two wiring paths, or two condition helpers, build on the better one and do not add a third. A small schema or constructor change needs only the relevant part of this discovery.

Record useful project conventions and exceptions in `.local/operator-development/SKILL.md` when they would help subsequent work. Update an existing record rather than duplicating it. Include actual paths, commands, runtime semantics, and relevant exceptions; do not copy this skill or turn assumptions into project rules.

## Architecture

| Layer | Owns | Must not |
| --- | --- | --- |
| Composition root | Configuration, clients and credentials, wiring controllers to the runtime, process lifecycle | Live in domain packages or rely on import side effects |
| Adapter | Turning the runtime's request into a call, applying the result, mapping outcomes, stabilizing condition timestamps, logging with object identity | Make domain decisions |
| Desired state | A function from spec, observed children, related objects, configuration, and injected clients to desired children, status, and outcome | Write to the cluster, perform I/O except through injected clients, read the clock, or derive desired objects from its own previous status |
| External clients | Side effects in external systems behind interfaces as small as the call site needs | Leak transport details or credentials into domain code |

Use the runtime's own primitives where it has them, such as result and terminal-error types, condition setters, and finalizer and owner-reference helpers. Write an adapter only where the runtime hands you a protocol envelope to translate. Wire types may appear in a handler's signature; child constructors, naming, selection, and readiness helpers take plain object types so they can be reused across controllers and runtimes.

The desired-state function is the unit of reuse. A factory receives configuration and clients once and returns it:

```go
func ComposeWidget(cfg Config) ComposeFunc {
	return func(ctx context.Context, in Observed) (Desired, error) {
		var out Desired
		add := func(o *Object) *Object {
			o.SetLabels(merged(o.GetLabels(), ownedBy(in.Parent)))
			out.Children = append(out.Children, o)
			return o
		}

		sa := add(NewServiceAccount(in.Parent, cfg.Project))

		// Gate the dependent on what the prerequisite reports, found by the name just built.
		email := emailOf(in.Child(sa))
		if email == "" {
			out.Status.SetCondition(NotReady(ReasonPrerequisiteNotReady, sa))
			return out, nil
		}
		add(NewBinding(in.Parent, email))
		out.Status.Set("serviceAccountEmail", email)

		out.Status.SetCondition(Readiness(in, out.Children))
		return out, nil
	}
}
```

A thin adapter serves it: a hook handler that decodes and encodes, or a reconcile function that applies, prunes, and writes status. Tests call the returned function directly. Adapt the form to the project's language and runtime; keep the separation.

## Choose the controller shape

| Shape | Use when | Defining rule |
| --- | --- | --- |
| Composite | A parent resource owns a set of child resources | Compute the complete desired child set on every pass |
| Decorator | Behavior attaches to objects you do not own | Never take over the target's lifecycle; write only where you own the output |
| External-resource sync | One custom resource represents one object in an external system | Observe before acting; creation is idempotent; cleanup is confirmed |
| One-shot action | An imperative request must run once and record its outcome | Immutable request, terminal conditions, idempotency key from the object |

Related objects, such as referenced teams or groups matched by a selector, are an input any shape may declare, not a shape of their own. Prefer composing shapes over one large controller: a parent that needs an external object usually emits a child resource handled by a dedicated external-sync controller with its own cleanup.

## Composition and reuse

- Build each child with a pure constructor, `NewX(parent, …)`. Apply shared metadata such as ownership labels through one `add` closure or one pass over the assembled set.
- When children come from several independent sources, such as backends or tenants, define a small interface, an `XFunc` adapter so plain functions satisfy it, and a combinator. Whether an instance handles an input is an explicit, tested predicate, distinct from handling it and producing no children. Combinators report which instances failed and never return a silently shortened list.
- Take dependencies as consumer-defined interfaces, such as a single `Do(request)` method. Keep credentials in transports or token sources.
- Define each cross-controller contract once: label and annotation keys, group/version/kind identifiers, condition types and reasons, and label-value encodings. A producer writing a label and a consumer selecting on it call the same encoding function.
- Parse configuration and your own spec into typed values once, at the edge. Use untyped access for foreign kinds you do not want to depend on.
- Wire controllers in the composition root with direct constructor calls, passing plain values and clients. Keep environment-specific values in configuration. Add a registration table only when deployments must enable subsets; never register through import side effects.
- When two controllers need the same mechanism, such as readiness aggregation or condition setting, share one helper rather than letting variants with different semantics grow.

## Reconciliation invariants

1. **Level-triggered and idempotent.** Each pass derives everything from current spec and observed state. Running it twice produces identical output and no writes. Every write must be safe to repeat, because passes are not transactional.
2. **Failures never shrink the desired set.** A part that cannot be computed is carried forward from what is observed or excluded from pruning, never silently dropped, because dropped children are deleted along with whatever their cleanup manages. Report the failure in a condition whenever status can be written. Follow the failure table in [Reconciliation](references/reconciliation.md).
3. **Deterministic output.** Sort anything derived from maps, sets, or selections, keep condition order stable, and derive names from identity. Do not derive desired objects from your own previous status. Status may be read for controller-allocated identifiers and pending-operation markers, terminal conditions of one-shot actions, and previous conditions to preserve their timestamps; the last belongs to the adapter or the runtime's condition setter.
4. **Outcomes travel with desired state.** Requeue and cleanup-complete are outcomes, not failures; they never discard desired children or status. Translate integration-specific errors into the runtime's outcome vocabulary once, at the integration boundary.
5. **Conditions change only when observations change.** Preserve `lastTransitionTime` unless the condition's status flips, set `observedGeneration`, and keep values that change every pass out of status. Anything else turns your own watch into a hot loop. Never put secret values in status, conditions, events, or logs.
6. **Expect concurrency and stale reads.** Treat conflicts as normal retries. Treat already-exists for a deterministic name as success only when the live object carries your ownership marker; otherwise report it and never adopt, update, or delete it. Compare against live objects in a way that ignores server defaults, and keep one writer per field.
7. **Cleanup is ordered, idempotent, and confirmed.** Guard cleanup before the first external side effect, treat not-found as done, release only after confirming absence, and never block on a related object that may be deleted at the same time.

## Reviewing

Rank findings by consequence: (1) deletes, duplicates, or orphans children or external objects; (2) stuck or premature cleanup; (3) wrong or flapping status and write churn; (4) composition and reuse; (5) conventions, only within the requested scope. Check the runtime contract before judging a pattern; a rule about pruning or status replacement depends on it.

## Read supporting guidance

Read only the supporting guidance relevant to the change:

- For per-shape rules, owner-reference limits, related objects, and how concepts map onto each runtime, read [Controller shapes](references/controller-shapes.md).
- For the failure table, outcomes, concurrency, external systems, cleanup, and readiness, read [Reconciliation](references/reconciliation.md).
- For CRD schemas, generated code, spec and status, conditions, names, labels, and versions, read [API design](references/api-design.md).
- For layout, adapters, wiring, process hygiene, and naming conventions, read [Architecture](references/architecture.md).
- When choosing tests or validating an implementation, read [Verification](references/verification.md).

## Completion

Define the observable outcome in terms of the custom resource: which children or external objects exist, which status and conditions are reported, and what happens on deletion. Verify the paths the change affects, such as success, failure, a repeated pass with no change, and cleanup. A schema-only, naming, or constructor change needs only its focused check. Run the project's build, vet or lint, and focused tests; report what was verified and what would need a real cluster or external system to confirm.
