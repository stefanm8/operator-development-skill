# Reconciliation

Read this reference when writing or reviewing desired-state logic, failure and requeue handling, concurrency, external-system synchronization, cleanup, or readiness.

## Desired state

A pass reads spec, observed children, related objects, and configuration; builds the full desired set; then applies it and reports status. Keep it that way:

- Do not branch on whether the trigger was a create, update, or delete event. Runtimes coalesce, drop, and replay events.
- Build the whole set before applying or pruning anything. Helpers that each add children are fine; a helper's failure is handled by the failure table, not by returning less.
- Keep clock values, random values, and map iteration order out of desired objects. Sort lists that come from maps, sets, or selections.
- Use plain loops for steps that can fail. Map, filter, and for-each helpers must not swallow errors or panic.
- When a value is deterministic, such as a namespace name, call the function that produced it rather than reading it back from status. When a child allocates a value, read it from the observed child and withhold dependents until it appears. Decide explicitly what happens if that value briefly disappears, so a transient blank status does not delete a downstream object.

## Failures

Two rules apply in every runtime: never let a failure shrink what the runtime will keep, and make the failure visible in status when status can be written.

| Situation | Declarative runtime (you return the set) | Imperative runtime (you apply and prune) |
| --- | --- | --- |
| The object's own spec or configuration cannot be parsed or is invalid | Re-emit all observed children as in the next row, set a `False` condition with a specific reason, and do not requeue. Prefer schema validation so this is rare | Write a `False` condition with a specific reason; return a terminal failure if only a spec change can help; do not prune |
| One part of the set fails and has observed children | Return the full set, re-emitting that part's observed children with status and server-set metadata (`uid`, `resourceVersion`, `managedFields`, `creationTimestamp`) removed; set a `False` condition naming the part; requeue | Apply the parts that succeeded; skip pruning for the failed part; set a `False` condition; retry |
| One part fails and has no children yet | Return the set without that part, plus the condition and requeue | Apply the rest, plus the condition and retry |
| An external call fails | Return the full desired set, a `False` condition, and a requeue | Write the condition and retry |
| Status cannot be written on the failure path | Error response | Return the error |

- Wrap errors with context while preserving their identity, so not-found and conflict detection keeps working and condition messages say what failed.
- Do not discard parse errors when loading the object into a state value. A silently zero-valued field becomes a wrong external write.
- Never panic in a reconcile or hook path on malformed input. Return an error and add a schema rule that rejects it at admission.

## Outcomes

| Outcome | Meaning |
| --- | --- |
| Done | Desired state is fully expressed; nothing to wait for |
| Requeue after | Desired state is expressed; check again later, such as for a pending external operation |
| Retry | A transient failure; retry with backoff |
| Terminal | Retrying cannot help until the spec changes |
| Cleanup complete | External and staged cleanup is done; release the object |

- Use the runtime's own types for these where it has them. Where you define your own, carry the outcome in the result alongside desired state. If an outcome travels as an error value, the adapter must still keep desired children and status, and it must use the same unwrapping to detect and to extract it.
- Translate an integration's own retry or not-found errors into this vocabulary once, at the integration boundary, preserving any requested delay.
- Use exponential backoff with jitter for repeated retries unless the runtime already applies it. Never wait inside a pass for a long-running operation; record that it is pending and requeue.

## Conditions and status writes

- Desired-state functions return conditions without managing timestamps. The adapter, or the runtime's condition setter, copies `lastTransitionTime` from the observed condition when the condition's status is unchanged, and updates reason, message, and observed generation in place.
- Emit conditions in a stable order, set `observedGeneration`, and write status only when it differs from what is observed.
- When the runtime replaces status wholesale, every path returns the complete status, including requeue, failure, and cleanup paths. A path that returns an empty status erases published identifiers and conditions.
- Setting a condition type twice in one pass replaces the earlier value. A specific failure reason must not lose to a generic one set earlier.
- Keep values that change on every pass, such as last-sync timestamps, out of status. Each write triggers your own watch.

## Concurrency and caching

- Conflicts on update are normal. Avoid them with patches or server-side apply, which skip read-modify-write races. Where the runtime logs returned errors at error level, requeue a conflict after a short delay instead of returning it.
- Reads usually come from a cache and can be stale. After a create, expect already-exists on the next pass. Treat it as success only when the live object carries your owner reference or ownership label; otherwise set a `False` condition and never adopt, update, or delete it. Do not assume an object you just wrote is visible yet.
- Compare desired objects with live ones in a way that ignores server-populated defaults and metadata, or use server-side apply. Naive equality against defaulted live objects updates forever.
- With server-side apply, send the full intent for each object every time, because fields you stop sending are removed. Use a stable field manager and force conflicts on objects you own. In Go, use apply configurations or untyped objects rather than typed structs, which send zero values.
- Keep one writer per field: one controller owner per object, one controller per status field, and separate field managers when several controllers touch one object.
- Behavior configured through annotations or labels does not change `metadata.generation`. Event filters based on generation drop those changes, and `observedGeneration` cannot reflect them.

## External systems

- **Steps.** Parse the object once into a plain state value. Each step observes or acts and hands off to the next, and the loop checks cancellation. Finalization is another entry step.
- **Idempotent creation.** Use a deterministic external name, a client idempotency key, or lookup before create. If none exists and you can persist state before acting, record a pending marker first and reconcile it on the next pass. Assume any create response can be lost; an ID held only in status is a hint that a lookup must be able to recover.
- **Ownership.** Know which external objects you own: an allocated ID, a deterministic name, or a tag in the external system. Do not delete external objects you cannot prove you created.
- **Adoption and deletion policy.** Adopt only through an explicit, immutable reference in spec, paired with a deletion policy that every integration honors. Default adopted objects to abandon.
- **Drift.** Normalize both sides into one representation, such as enum casing, ordering, and defaults, then patch only what differs.
- **Identity changes.** When a spec field that identifies the external object changes, such as a member's email, either make the field immutable in the schema or remove the old external object using the identifier recorded in status. Otherwise the old object is orphaned.
- **Rate limits and time.** Use a per-resource resync interval with a floor, back off on throttling, and skip calls when the observed state already matches. Set client timeouts below the runtime's limit for one pass, so a slow upstream produces a requeue rather than a response the runtime never receives.

When reviewing an integration client, check for exact-match lookups by natural key rather than the first fuzzy-search result, status codes checked before decoding, every page of a paginated list read, and per-item errors inside successful batch responses treated as failures. Error messages copied into conditions must not contain credentials or signed URLs.

## Cleanup

- Guard cleanup, such as with a finalizer, before the first external side effect, and only while the object is not already being deleted.
- Make cleanup idempotent, treat not-found as done, and release only after confirming absence. A delete call returning is not confirmation.
- When teardown order matters, shrink the desired set in stages: withhold dependents first, then their prerequisites, and report cleanup complete only when no children remain. This works with background deletion, because owned children remain until the owner's finalizers clear. Foreground deletion deletes owned children at once, regardless of your finalizer. If order must hold under foreground deletion, give each prerequisite a finalizer that holds it until its dependents are gone, or do not owner-reference children that must outlive the start of teardown.
- Store what cleanup needs, such as the external ID, on the object being cleaned up. Never make cleanup wait on a sibling or related object that may be deleted at the same time; treat a missing dependency as already cleaned up.
- Honor the deletion policy. An abandoned external object survives deletion of its resource.

## Readiness

- Aggregate over the desired children. A desired child that has not been observed yet is not ready.
- Use per-kind rules. A custom resource is ready when its `Ready` condition is `True` for its current generation. Built-in workloads report their own status: for example, a Deployment is ready when its `observedGeneration` matches and its updated, ready, and available replica counts all equal `spec.replicas`. A kind without status is ready once observed with the desired spec. A shared status library can supply these rules.
- Zero desired children is a deliberate state; decide whether it means ready and say so in the reason.
- Use one shared aggregation helper with stable reasons, such as `ChildNotReady` with the child's kind and name in the message.
