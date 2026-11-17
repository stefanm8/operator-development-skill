# Verification

Read this reference when choosing tests or validating operator work. Test observable reconciliation outcomes and invariants. Scale checks to the change; do not run every scenario for a small edit.

## Choose evidence for the change

| Change | Useful verification |
| --- | --- |
| Child constructor, name or label function, selection function | Table-driven cases compared against whole expected objects, plus boundary inputs |
| Desired-state function | Hand-built observed state in. Assert the children by kind and name, the gating, the status fields, and the conditions, including dependency-not-ready and cleanup inputs |
| External integration | A fake at the transport or client seam covering not-found, conflict, throttling, partial-batch errors, and lost responses |
| Adapter | A recorded request through the real adapter with an inline desired-state function; assert outcome mapping and condition timestamps |
| CRD schema | Apply valid and invalid objects to an API server when validation rules, defaults, or list semantics change |
| Wiring and owner references | A test control plane |
| Garbage collection and cascading deletion | A disposable local cluster; never a shared cluster or the current kubeconfig context unless the user names it |

Run the project's build, vet or lint, and focused test commands. Reuse existing helpers, fixtures, and fakes rather than replacing them with a preferred style.

## Writing tests

- Test the production function, and fake external boundaries at the smallest interface the code depends on.
- Prefer hand-written fakes to mocking frameworks:
  - a function-type fake for a one-method interface;
  - a route-table fake for an HTTP transport, with defaults and per-case overrides.
  
  Check that each fake route is actually exercised. A fake that answers a different route than the one it was checked against hides bugs.
- Build observed fixtures with the production constructors, then add the status the test needs. Build expected status with the same builder the code uses. The fixtures then cannot drift from what the code emits.
- Use table-driven cases with named fields and subtests. Put assertion functions in the case when outputs are rich.
- Test through the package's public API, such as a separate test package in Go, unless an internal helper has independent meaning.
- Prefer the language's standard testing tools unless the project already uses an assertion library consistently.

## Runtime-specific notes

- Test control planes, such as envtest, run an API server without the controller manager. Garbage collection never runs, and namespaces never finish deleting, so assert that owner references are set and use a fresh namespace per test.
- Fake clients do not reproduce every API server behavior. Enable status subresource handling explicitly where the fake supports it, and do not test admission validation against a fake.
- For a hook-based runtime, an adapter test with a recorded HTTP request covers the protocol. The runtime's apply and prune behavior needs a cluster.

## Invariant tests

Pin the properties that keep an operator stable, alongside the scenarios below. Each test should fail if its property breaks:

- **Identical output.** Running the pass twice on the same input produces identical output, including child order and condition order.
- **Timestamp stability.** An unchanged condition keeps its observed `lastTransitionTime`, and a status flip updates it.
- **Name stability.** Known inputs map to known outputs, so an accidental change to the name function fails before it renames children in a cluster.

## High-value scenarios

Use the relevant scenarios as acceptance cases, regression tests, or a review guide:

1. **One source fails.** One of several child sources returns an error. No existing child is removed, and a condition names the failing source.
2. **Lost create response.** The external create succeeds but the status write does not. The next pass finds the object instead of creating a duplicate.
3. **Steady state.** A pass with no spec or observed change produces no writes to children, status, or the external system.
4. **Stale cache and name collisions.** A pass runs before its own previous create is visible. Already-exists counts as success, and nothing is duplicated. A same-named object you do not own is reported, not adopted.
5. **Prerequisite not ready.** Dependents are withheld, `Ready` is `False` with a reason, and the first pass never reports ready before children are observed.
6. **Spec change.** `observedGeneration` advances, and `Ready` reflects the new generation only after the children converge.
7. **Ordered deletion.** Deleting the parent removes dependents before prerequisites. Not-found counts as done, release happens only after confirmed absence, and cleanup completes even when a related object is deleted at the same time.
8. **Adopted object deleted.** Under the default policy, deleting the resource leaves the adopted external object in place.
9. **Hostile identifiers.** Long, mixed-case, Unicode, or punctuation-heavy inputs produce valid, stable, distinct names and label values.

## Reporting

Report which commands ran and what they covered. Distinguish verified behavior from behavior that would need a real cluster, a real external system, or a runtime that is not available locally.
