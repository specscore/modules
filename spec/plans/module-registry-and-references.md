# Plan: Module Registry and References

**Status:** Under Review
**Source Feature:** module-registry-and-references
**Date:** 2026-06-04
**Owner:** alex
**Supersedes:** —

## Summary

Decompose the approved Module Registry and References contract into six linearly-ordered tasks that build a dependency-light Go library (`github.com/specscore/modules`, go 1.26): reference parsing and classification, registry loading with path containment, local resolution over a pluggable item locator, distinct typed errors, and a conformance-vector harness.

## Approach

Tasks are ordered by dependency: the pure reference parser (local, then external) comes first, then registry loading and path containment, then the resolver that ties registry + parser + a host-provided locator together, and finally the conformance-vector harness that exercises all of it. ingitdb-cli's `pkg/ingitdb/config` is consulted as PRIOR ART for the flat `name: path` registry-loading pattern only (Task 3); the library MUST NOT depend on or import ingitdb-cli, because that would re-couple this standalone system to a product (the intended direction is the reverse — ingitdb later migrates onto this library). Everything else — the `module:item/path:field` grammar, local-vs-external classification, the external `host/org/repo[/module]@version` form, path containment (deliberately stricter than ingitdb's permissive absolute/`~` paths), typed errors, and the locator interface — is net-new. All 19 source ACs are covered; none deferred.

## Tasks

### Task 1: Reference parser — structural grammar (local)

**Verifies:** module-registry-and-references#ac:full-reference-parsed, module-registry-and-references#ac:field-optional, module-registry-and-references#ac:too-many-colons-malformed, module-registry-and-references#ac:traversal-reference-rejected, module-registry-and-references#ac:local-module-no-slash

Implement the core reference parser: split on `:` into two or three parts (module, item-path, optional field), reject more than two colons and any `..` traversal segment as malformed, and classify a module containing no `/` as local. Pure function — no registry or filesystem access.

### Task 2: Reference parser — external repo form

**Verifies:** module-registry-and-references#ac:external-reference-parsed, module-registry-and-references#ac:external-with-module-and-version, module-registry-and-references#ac:malformed-external-rejected

Extend the parser to classify a module containing `/` as external, parsing `host/org/repo` with an optional `/in-repo-module` segment and an optional trailing `@version` (split on `@`, then on `/`; require a dotted host). Reject `/`-bearing modules that do not match the external shape as malformed.

### Task 3: Registry loader & validation

**Verifies:** module-registry-and-references#ac:registry-loaded, module-registry-and-references#ac:module-name-with-slash-rejected, module-registry-and-references#ac:duplicate-path-rejected, module-registry-and-references#ac:default-module-without-registry

Read `.modules/registry.yaml` as a flat `name: path` map, validate the local module-name grammar and both name and path uniqueness, and synthesize the reserved `default` module rooted at the project root when the file is absent. Adapt the flat-map reading pattern from ingitdb-cli `pkg/ingitdb/config` as prior art only — no dependency.

### Task 4: Path containment

**Verifies:** module-registry-and-references#ac:registry-path-escape-rejected, module-registry-and-references#ac:absolute-registry-path-rejected

Enforce that each registry path is relative and resolves at or under the project root, rejecting absolute paths and `..`-escapes as configuration errors. This intentionally diverges from ingitdb's permissive absolute/`~` path handling.

### Task 5: Resolver, item-locator interface & typed errors

**Verifies:** module-registry-and-references#ac:reference-resolved-via-locator, module-registry-and-references#ac:same-name-different-modules-distinct, module-registry-and-references#ac:unknown-module-error, module-registry-and-references#ac:external-reference-not-resolved-here

Define the host-provided item-locator interface and the resolver: map a local module to its base path, delegate the item-path to the locator, and return `(base path, item location, optional field token)` or a distinct typed error (malformed / unknown-module / item-not-found). The same item name under different modules resolves to distinct locations; a parsed external reference reports not-resolved-here without erroring as malformed.

### Task 6: Conformance vectors & runner

**Verifies:** module-registry-and-references#ac:conformance-vector-pass

Author `vectors.yaml` mapping `(registry, reference)` inputs to expected resolved parts or expected typed errors as the normative authority, plus a runner that executes every vector and fails on any deviation.

## Open Questions

None at this time.

---
*This document follows the https://specscore.md/plan-specification*
