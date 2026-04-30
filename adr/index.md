# Architecture Decision Records

本目錄收錄所有設計決策的 rationale，讓使用者了解為什麼選擇這些工具與模式。每份 ADR 用 [MADR](https://adr.github.io/madr/) 標準格式。

---

## 索引

| # | Title | Status | Topic |
|---|-------|--------|-------|
| [001](./001-bloc-not-cubit.md) | BLoC over Cubit | Accepted | state-management |
| [002](./002-bloc-event-plain-sealed-class.md) | BLoC Event: Plain Sealed Class | Accepted | state-management |
| [003](./003-bloc-state-freezed-sealed-class.md) | BLoC State: Freezed Sealed Class | Accepted | state-management |
| [004](./004-bloc-event-past-tense-naming.md) | BLoC Event Past Tense Naming | Accepted | state-management |
| [005](./005-imperative-actions-as-side-effect.md) | Imperative Actions as SideEffect | Accepted | state-management |
| [006](./006-side-effect-plain-sealed-class.md) | SideEffect: Plain Sealed Class | Accepted | state-management |
| [007](./007-either-pattern-for-domain-results.md) | Either Pattern for Domain Results | Accepted | error-handling |
| [008](./008-named-parameters-everywhere.md) | Named Parameters Everywhere | Accepted | coding-style |
| [009](./009-usecase-call-method-with-input.md) | UseCase: `call()` Method with Input Class | Accepted | domain-layer |
| [010](./010-no-build-prefix-private-method.md) | No `_build` Prefix Private Methods for Widgets | Accepted | presentation-layer |
| [011](./011-domain-zero-flutter-imports.md) | Domain Layer: Zero Flutter Imports | Accepted | architecture |
| [012](./012-features-no-cross-import.md) | Features Never Cross-Import | Accepted | architecture |
| [013](./013-app-entry-shell-bootstrap-layering.md) | App Entry / Shell / Bootstrap Layering | Accepted | architecture |
