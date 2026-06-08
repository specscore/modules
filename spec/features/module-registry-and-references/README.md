---
format: https://specscore.md/feature-specification
status: Approved
---

# Feature: Module Registry and References

> [SpecScore.**Studio**](https://specscore.studio): | [Explore](https://specscore.studio/app/github.com/specscore/modules/spec/features/module-registry-and-references?op=explore) | [Edit](https://specscore.studio/app/github.com/specscore/modules/spec/features/module-registry-and-references?op=edit) | [Ask question](https://specscore.studio/app/github.com/specscore/modules/spec/features/module-registry-and-references?op=ask) | [Request change](https://specscore.studio/app/github.com/specscore/modules/spec/features/module-registry-and-references?op=request-change) |
**Status:** Approved
**Date:** 2026-06-04
**Owner:** alex
**Source Ideas:** —
**Supersedes:** —

## Summary

A standalone, product-agnostic module system. It registers named modules by path, treats each module as a pure namespace for items, and resolves module-qualified references of the form `module:item/path:field` to a concrete item location and an optional field. The `module` part is either a local registry key or an external `host/org/repo` coordinate (with an optional `@version` pin), so a reference can point at an item in another repository. Item location is delegated to a host-provided locator, so any consumer — DataTug entities, SpecScore features, ingitdb collections — shares the same registry, grammar, and resolution rules without coupling to one another.

## Problem

Several sibling tools each need to organize items into named modules and resolve references across them — sometimes across repositories — but no neutral shared contract exists. Embedding the concept inside any one product (e.g. ingitdb's `.ingitdb/root-collections.yaml`) couples every other adopter to that product. Without a shared registry format, reference grammar, and resolution semantics — pinned by conformance vectors — implementations drift and the goal of "declare a module/item once, recognized by every tool" is impossible.

This Feature originates from DataTug's `shared-module-system` Idea (`github.com/datatug/datatug-cli`, `spec/ideas/shared-module-system.md`; cross-repo — structured Source-Ideas linking is an Open Question). DataTug and SpecScore are the two first consumers that prove this contract before ingitdb migrates onto it. The reference implementation to extract from is `ingitdb-cli/pkg/ingitdb/config` (`RootConfig`, `ResolveNamespaceImports`).

## Behavior

### Module registry

A project declares its modules in a single registry file.

#### REQ: registry-location

The system MUST read the module registry from `.modules/registry.yaml` at the project root. The file MUST be a flat YAML map of `<module-name>: <path>` entries — no wrapper key, no nested structure.

#### REQ: registry-name-grammar

A local module name (a registry key) MUST consist of one or more characters from letters, digits, `_`, `-`, and `.`. It MUST NOT contain `/`, `:`, or a `..` segment. Module names are case-sensitive.

#### REQ: registry-entries-unique

Module names MUST be unique within a registry, and no two modules MAY resolve to the same base path. A registry violating either constraint is a configuration error.

#### REQ: single-module-default

When `.modules/registry.yaml` is absent, the system MUST expose exactly one module — the reserved `default` module — rooted at the project root (the degenerate single-module case).

### Path containment

A module's storage MUST stay inside its own repository; reaching another repository is done only through the external reference form, never through filesystem escape.

#### REQ: registry-path-containment

Each registry `<path>` MUST be relative to the project root and MUST resolve to a location at or under that root. A path that is absolute, or that escapes the root through a `..` segment, MUST be rejected as a configuration error.

#### REQ: reference-no-traversal

A `module` (local or external) MUST NOT contain a `..` segment. A reference that attempts filesystem traversal MUST be rejected as malformed, with no filesystem access.

### Reference grammar

A module-qualified reference addresses an item, and optionally a field, inside a module.

#### REQ: reference-grammar

A reference MUST match `reference := module ":" item-path [ ":" field ]`, where `item-path := segment ( "/" segment )*`. A `segment` and a `field` MUST NOT contain `:` or `/`. The `field` part is OPTIONAL.

#### REQ: reference-parse-syntactic

A reference MUST be parseable structurally by splitting on `:` into exactly two parts (`module`, `item-path`) or three parts (`module`, `item-path`, `field`) — with no registry, schema, or filesystem access required to determine its structure. More than two `:` separators is a malformed reference.

#### REQ: local-vs-external-module

A `module` that contains no `/` is a LOCAL module — a key resolved against this repository's registry (or the `default` module). A `module` that contains `/` is an EXTERNAL repository coordinate. The presence of `/` is the sole discriminator.

#### REQ: external-module-form

An external `module` MUST take the form `host/org/repo` — where `host` contains a `.` (e.g. `github.com`) — optionally followed by `/<in-repo-module>` (a single name obeying `registry-name-grammar`) and optionally suffixed with `@<version>`. A `module` containing `/` that does not match this shape MUST be rejected as malformed. With no `/<in-repo-module>`, the reference targets the external repository's `default` module.

#### REQ: external-version-optional

The `@<version>` suffix is OPTIONAL and pins the external repository to a version (tag, semver, or commit). `<version>` MUST NOT contain `:` or `/`. Its absence means unpinned; the meaning of "unpinned" is a resolution-policy concern (see `external-resolution-out-of-scope`).

### Namespacing and identity

#### REQ: pure-namespacing

Modules are pure namespaces and locations. The same item name under two different modules denotes two DIFFERENT items. The system MUST NOT impose cross-module inheritance, overriding, or identity merging. (Richer identity, if ever needed, is a future contract revision, not host-local behavior.)

### Resolution

#### REQ: reference-resolution

Given a parsed reference to a LOCAL module and a host-provided item locator, the system MUST resolve `module` to its base path via the registry, delegate `item-path` to the locator to obtain a concrete item location under that base path, and return the resolved `(module base path, item location, optional field token)`. Interpreting the `field` token against the located item is the host's responsibility.

#### REQ: resolution-errors

Resolution MUST distinguish, as separate typed errors: a malformed reference, an unknown module (not in the registry / not the default), and an item the locator cannot find. A failing resolution MUST NOT return a partial success.

#### REQ: external-resolution-out-of-scope

This contract defines the ADDRESSING and PARSING of external references only. Fetching an external repository's content, applying `@version`, caching, offline behavior, and trust of third-party definitions are OUT OF SCOPE and delegated to a separate resolver Feature. An implementation MAY parse an external reference and report it as not-resolved-here without violating conformance.

### Conformance

#### REQ: conformance-vectors

The contract MUST be accompanied by conformance vectors mapping `(registry, reference)` inputs to either resolved parts or a specific typed error. The vectors — not any prose or single implementation — are the normative authority an implementation conforms to.

## Acceptance Criteria

### AC: registry-loaded (verifies REQ:registry-location)

**Given** a project whose `.modules/registry.yaml` contains `payments: modules/payments` and `todo.tasks: modules/todo`
**When** the system loads the registry
**Then** it exposes exactly two modules, `payments` and `todo.tasks`, each mapped to its declared path.

### AC: module-name-with-slash-rejected (verifies REQ:registry-name-grammar)

**Given** a registry containing the key `pay/ments: modules/x`
**When** the registry is validated
**Then** the system reports a malformed-module-name error because a local module name MUST NOT contain `/`.

### AC: duplicate-path-rejected (verifies REQ:registry-entries-unique)

**Given** a registry containing `a: modules/shared` and `b: modules/shared`
**When** the registry is validated
**Then** the system reports a duplicate-path configuration error.

### AC: default-module-without-registry (verifies REQ:single-module-default)

**Given** a project rooted at `/proj` with no `.modules/registry.yaml`
**When** the modules are enumerated
**Then** exactly one module named `default` is exposed, rooted at `/proj`.

### AC: registry-path-escape-rejected (verifies REQ:registry-path-containment)

**Given** a registry containing `x: ../outside`
**When** the registry is validated
**Then** the system rejects it as a configuration error because the path escapes the project root.

### AC: absolute-registry-path-rejected (verifies REQ:registry-path-containment)

**Given** a registry containing `x: /etc/secrets`
**When** the registry is validated
**Then** the system rejects it as a configuration error because absolute paths are not permitted.

### AC: traversal-reference-rejected (verifies REQ:reference-no-traversal)

**Given** the reference `../../something:foo:bar`
**When** it is parsed
**Then** the system reports a malformed-reference error, using no filesystem access.

### AC: full-reference-parsed (verifies REQ:reference-grammar)

**Given** the reference `todo.tasks:records/tasks:id`
**When** it is parsed
**Then** it yields module `todo.tasks`, item-path `records/tasks`, and field `id`.

### AC: field-optional (verifies REQ:reference-grammar)

**Given** the reference `payments:currency`
**When** it is parsed
**Then** it yields module `payments`, item-path `currency`, and no field.

### AC: too-many-colons-malformed (verifies REQ:reference-parse-syntactic)

**Given** the reference `a:b:c:d`
**When** it is parsed
**Then** the system reports a malformed-reference error, using no registry or filesystem access.

### AC: local-module-no-slash (verifies REQ:local-vs-external-module)

**Given** the reference `payments:currency:id`
**When** it is classified
**Then** `payments` is treated as a local registry key because it contains no `/`.

### AC: external-reference-parsed (verifies REQ:external-module-form)

**Given** the reference `github.com/specscore/finance:currency:id`
**When** it is parsed
**Then** it yields external repo `github.com/specscore/finance`, its `default` module, item-path `currency`, and field `id`.

### AC: external-with-module-and-version (verifies REQ:external-module-form, REQ:external-version-optional)

**Given** the reference `github.com/specscore/finance/payments@v1.2.0:currency:id`
**When** it is parsed
**Then** it yields host/org/repo `github.com/specscore/finance`, in-repo module `payments`, version `v1.2.0`, item-path `currency`, and field `id`.

### AC: malformed-external-rejected (verifies REQ:external-module-form)

**Given** the reference `notahost/x:currency:id`
**When** it is parsed
**Then** the system reports a malformed-reference error because the external form requires a dotted `host/org/repo`.

### AC: same-name-different-modules-distinct (verifies REQ:pure-namespacing)

**Given** registries with modules `payments` and `billing`, each containing an item `currency`
**When** `payments:currency` and `billing:currency` are resolved
**Then** they resolve to two distinct item locations with no merged or inherited identity between them.

### AC: reference-resolved-via-locator (verifies REQ:reference-resolution)

**Given** module `payments` resolved to `/proj/modules/payments` and a host locator that maps item-path `currency` to `entities/currency/currency.entity.json`
**When** `payments:currency:id` is resolved
**Then** the result is `(/proj/modules/payments, /proj/modules/payments/entities/currency/currency.entity.json, "id")`.

### AC: unknown-module-error (verifies REQ:resolution-errors)

**Given** a registry without a module named `ghost`
**When** `ghost:currency:id` is resolved
**Then** the system returns an unknown-module typed error distinct from a malformed-reference error and from an item-not-found error.

### AC: external-reference-not-resolved-here (verifies REQ:external-resolution-out-of-scope)

**Given** the parsed external reference `github.com/specscore/finance:currency:id`
**When** local resolution is attempted
**Then** the reference parses successfully but is reported as not-resolved-here, without the implementation being marked non-conformant.

### AC: conformance-vector-pass (verifies REQ:conformance-vectors)

**Given** the published conformance vectors for registry-and-reference resolution
**When** an implementation runs every vector
**Then** it produces the expected resolved parts or the expected typed error for each, and any deviation marks the implementation non-conformant.

## Rehearse Integration

All acceptance criteria are testable as pure functions (registry parsing/validation, reference parsing/classification, and local resolution against a stub locator). Per-AC Rehearse stubs are deferred: the `conformance-vectors` REQ is the realized test surface — these ACs become entries in the published conformance-vector set at implement time, so scaffolding parallel stubs now would duplicate that artifact.

## Open Questions

- External CONTENT resolution — fetching the remote repository, applying `@version`, caching, offline behavior, and trust of third-party definitions — is delegated to a separate resolver Feature (out of scope here).
- `@version` token format (semver vs tag vs commit SHA) and the resolution policy when a reference is unpinned — to pin at resolver-Feature time.
- Deeper external module nesting (more than one in-repo module segment after `host/org/repo`) — deferred; the MVP supports at most one in-repo module name.
- Giving an external repository a short LOCAL alias in `.modules/registry.yaml` (convenience over repeating full coordinates) — deferred.
- May a reference omit the `module` segment for the `default` module (e.g. a bare `item/path`)? Positional `:` parsing makes `currency:id` ambiguous (module+item vs default-item+field), so omission is deferred.
- The Go library API shape: registry loader, `Resolve()` signature, and whether the item locator is a defined interface or a callback. (Implementation concern; specified at implement time.)
- Whether `.modules/` also carries settings (e.g. a configurable default-module name) alongside `registry.yaml`.
- Structured cross-repo `Source Ideas` linking back to DataTug's `shared-module-system` Idea (currently recorded in prose).

---
*This document follows the https://specscore.md/feature-specification*
