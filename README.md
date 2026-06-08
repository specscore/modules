# modules

A standalone, product-agnostic module system: a registry that registers named modules by path,
namespacing, and module-qualified reference resolution over a pluggable item abstraction.

Code name `github.com/specscore/modules` (host org provisional — see the source Idea's Open Questions).
Designed to be **aligned, not coupled**: DataTug, SpecScore, and ingitdb conform to the shared
contract without code-coupling to one another.

Origin: DataTug Idea `shared-module-system` (github.com/datatug/datatug-cli, spec/ideas/shared-module-system.md).
