# API design

Read this reference when adding or changing a custom resource definition, status fields, conditions, generated names, or label and annotation contracts. Treat the CRD as a public API: invariants that can be enforced at admission belong in the schema, not in reconcile code.

## Changing a CRD

- If CRDs, deep-copy code, or clients are generated, edit the source types and markers and run the generator. Never hand-edit generated output.
- A field added to a served version is optional or defaulted, and the code handles it being absent on existing objects.
- Declare every field the code reads. The API server prunes unknown fields, so a field the code accepts but the schema omits is silently dropped.
- Keep the code's validation and the schema's in agreement. If the code would reject an input, the schema should reject it first.

## Spec and status

- `spec` holds desired state. `status` holds observations and values the controller allocated, such as external IDs. Do not store desired state in status, or controller memory in annotations when a status field fits.
- Enable the status subresource. Writes to the main resource then ignore status, and `metadata.generation` advances on every change except to metadata and status.
- Publish `status.observedGeneration` and set `observedGeneration` on each condition, so readers can tell whether status describes the current spec.
- Use conditions as the primary signal. A phase field may be a derived summary for display.
- Name single references `fooRef` and lists `fooRefs`, and assume a referenced object may not exist.
- Make every field explicitly optional or required. Put defaults only on optional fields, and never override a value the user set.

## Schema and validation

- Use structural schemas. Reserve preserve-unknown-fields for genuinely open subtrees, such as an embedded foreign object. Validation rules cannot see undeclared fields beneath it, and for embedded resources only `apiVersion`, `kind`, `metadata.name`, and `metadata.generateName` are visible.
- Use enums for closed sets, `required` for mandatory fields, `format` for dates and similar values, and `maxItems` and `maxLength` on anything a validation rule iterates.
- Declare list semantics: `listType: map` with explicit keys for lists of objects identified by a field, and `listType: set` for unique scalars. Duplicates are then rejected at admission instead of producing colliding children.
- Enforce immutability with transition rules such as `self == oldSelf` on identity fields, adoption references, and one-shot request specs.
  - A rule on an optional field does not run when the field is added or removed. To lock it after creation, also add a rule on the parent object, such as `has(self.fooRef) == has(oldSelf.fooRef)`.
  - A rule on a list-map key cannot fire, because a changed key is a different item.

## Conditions

- Each condition has `type`, `status` (`True`, `False`, or `Unknown`), a CamelCase `reason`, a human-readable `message`, `lastTransitionTime`, and `observedGeneration`.
- Declare `status.conditions` as `listType: map` keyed by `type`, with the required fields and a status enum.
- Name condition types as adjectives or past-tense states, such as `Ready`, `Synced`, or `Complete`. Set the primary condition on the first pass, even as `Unknown`.
- Define condition types and reasons as shared constants. Keep the set of reasons small, remove synonyms, and make each reason correspond to a distinct action a reader could take.

## Names

- Generate child and external names with one deterministic function. It joins the identity parts, sanitizes characters, enforces the target's length and first-character rules, and always appends a short hash of the original parts. Always hashing keeps names distinct after lossy sanitization or truncation, and keeps a name's form from changing when an input crosses a length threshold.
- Give each target its own constraint: general object names, label-length names such as Services, and external systems with their own limits.
- Treat the function's output as frozen. Changing it renames every child on upgrade, which deletes and recreates them. Pin known outputs with a test.
- Derive a child's name from the identity that should map to exactly one object. When two parents could claim the same external object, decide whether the name includes the parent, and document the choice.

## Labels and annotations

- Use prefixed keys under a domain you control, defined once as constants shared by every producer and consumer.
- Label values have stricter rules than names. Use one sanitizer that enforces length and alphanumeric start and end, and add a hash when sanitization is lossy, so two inputs cannot collapse to the same value and select the wrong objects.
- Use labels for selection and ownership, and annotations for configuring decorated objects and for non-identifying metadata. Annotation-driven behavior is an API: document, validate, and handle removal of its keys.

## Versions and presentation

- Start new kinds at an alpha version and keep exactly one storage version. Plan conversion before an incompatible schema change, and migrate stored objects before removing a served version.
- Add printer columns for the primary condition and the most useful identifiers, such as the external ID. Add short names and categories when kinds share names across groups.
- Keep one CRD per file, named after its kind and group, and include every file in the deployment's manifest list.
