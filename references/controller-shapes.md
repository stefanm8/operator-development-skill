# Controller shapes

Read this reference when choosing how a controller should be structured, reviewing whether it follows the rules of its shape, or mapping a concept onto the project's runtime. Keep each controller to one shape and connect controllers through resources, labels, and status rather than shared code paths. Related objects are an input any shape may declare.

## Composite: parent owns children

A parent custom resource expands into child resources that are created, updated, and pruned to match.

- Compute the complete desired child set on every pass. In a declarative runtime, return it; the runtime deletes what is missing. In an imperative runtime, apply each object and prune only children carrying your ownership label that are absent from the set.
- Derive child names from the parent's identity and spec through the project's deterministic name function. Do not use generated random names for owned children.
- Mark ownership with a controller owner reference where scope allows and a prefixed label naming the parent. A namespaced owner can own only objects in its own namespace, and a cluster-scoped child cannot have a namespaced owner. When an owner reference is impossible, label the child with the parent's identity, map child events to the parent by that label, and delete children by label from the parent's cleanup.
- Every child kind needs a watch that triggers the parent and the matching RBAC. A controller that creates role bindings also needs permission to bind the roles they reference.
- Gate dependents on the observed state of their prerequisites, such as an identity that must exist before a binding to it. Look the prerequisite up by the name of the desired object just built rather than recomputing it, and report the withheld dependent with a condition reason.
- Readiness covers every desired child, not only the observed ones (see Reconciliation).

## Decorator: attach to objects you do not own

The controller adds behavior to existing objects selected by kind, labels, or annotations.

- Never take over the target's lifecycle: do not make yourself its controller owner and do not delete it.
- Configure the behavior through prefixed annotations, labels, or a separate custom resource, not through the target's own spec. Document those keys as an API and handle their removal by undoing what you added.
- Write output only where you own it: annotations, a separate custom resource, or child objects that name the target as owner. Writing into another controller's status is fragile, because built-in kinds prune unknown status fields and the owner's full status update replaces yours. Add your own condition there, with server-side apply and a dedicated field manager, only when the owner is known to preserve it.
- Keep the decision whether to act as a separate, tested predicate. Read and compare, or record a hash of what was last applied, before each external side effect, so periodic passes do not repeat writes against rate-limited APIs.
- Treat the target as possibly missing or not ready. A decorator that only derives status from related objects is a projection; keep it pure and sorted.

## External-resource sync: one resource, one external object

A custom resource represents one object in an external system, such as a SaaS team, a cloud identity, or a repository.

- Parse the object once into a plain state value. Drive the external system with small steps, each of which observes or acts and hands off to the next, under a loop that checks cancellation; or compute the action with a pure function over normalized observed and desired values. Every step is idempotent, and tests run through a fake at the transport.
- Creation is idempotent and adoption is explicit, with a deletion policy every integration honors. Details are in Reconciliation.
- Publish the external ID and other allocated values in status, and expose the important ones as printer columns.
- Prefer one custom resource per external object to a parent that loops over many. Removing an entry from the parent's spec then deletes one child, whose own cleanup handles the external object, and pruning never needs to list the external system or touch objects nobody declared.

## One-shot action

An imperative request, such as running a workflow, opening a change request, or provisioning on demand, runs once and records what happened.

- Model the request as a resource with an immutable spec, enforced in the schema.
- Use terminal conditions, such as `Complete` and `Failed`, read them at the start of each pass, and never re-run once terminal.
- Derive an idempotency key from the object's identity and pass it downstream.
- Record outputs and links to external artifacts in status. Clean up through a time-to-live or explicit deletion.
- When the action arrives through a webhook rather than a resource, keep the handler a thin adapter over a function with injected clients, so it can be tested without the external services.

## Related objects

A parent's desired state may depend on objects it neither owns nor decorates, such as referenced teams or groups that match a selector.

- Declare the selection from the parent's spec: explicit references, names, or label selectors. Keep the selection function pure and test it on its own.
- Encode selector values with the same function the producer uses to write labels, so both sides agree after sanitization and hashing.
- Treat selected objects as possibly absent, not ready, or being deleted. An absent reference is an expected state with a condition reason.
- Reject malformed selection input with an error and a schema rule rather than a crash. Sort anything derived from the selected set.

## Runtime mapping

Map the project's runtime before writing code. The concepts are the same; the mechanisms differ.

| Concept | Reconciler framework (for example controller-runtime) | Hand-rolled informer loop (for example client-go) | Hook-based declarative controller | Event-handler framework (for example kopf) |
| --- | --- | --- | --- | --- |
| Run the desired-state function | From the reconcile function; server-side apply each object, then prune by label or owner | From the worker's sync function; apply and prune yourself | The sync hook body; the runtime applies and prunes | One idempotent function called from create, update, and resume handlers; apply and prune with an API client yourself |
| Requeue after a delay | Result with a requeue-after value and a nil error | Add the key back after a delay | Requeue interval in the hook response | A timer, or a temporary error with a delay for genuine waits |
| Retry with backoff | Return an error; the result is then ignored | Add the key back through the rate limiter | Error response; the runtime retries | Raise a temporary error with a delay computed from the handler's retry count; plain exceptions retry at a fixed per-handler delay |
| Terminal failure | Terminal error type, plus a condition | Forget the key, plus a condition | Condition and no requeue | Permanent error, plus a condition |
| Child and related watches | Owned-type watches, or watches with a mapping function and field index | Secondary informers whose handlers enqueue the parent key by owner reference or index | Child and related kinds declared in the controller configuration, or a hook that returns selectors | Handlers or indexes on the child and related kinds that re-trigger the parent; status and child changes do not fire the parent's change handlers |
| Cleanup | Finalizer helpers, handled when the deletion timestamp is set | Manual finalizer handling | Deletion-in-progress signal in the request; report cleanup complete in the response | Delete handler with a framework-managed finalizer |
| Ownership | Controller-reference helper plus labels | Controller owner reference plus labels | Runtime-managed ownership plus labels | Adopt helper to stamp owner, namespace, and labels; set the name first, because an unnamed child gets a generated one |
