# Flutter 開發規範（主索引）

> Public、MIT、繁中為主。完整文件樹見 [README.md](./README.md)。

## 你要做的事 → 必讀檔案

| 你要做的事 | 必讀 |
|---|---|
| 寫/改 Page、Widget | `architecture/presentation-layer.md` |
| 寫/改 BLoC、Event、State、SideEffect | `patterns/state-management.md` |
| 寫/改 UseCase、Entity、Value Object | `architecture/domain-layer.md` |
| 寫/改 Repository、DataSource、Mapper | `architecture/data-layer.md` |
| 處理錯誤、Either、AppFailure | `patterns/error-handling.md` |
| 觸發導航 | `patterns/navigation.md` |
| Codegen（freezed / injectable / slang） | `patterns/code-generation.md` |
| DI 註冊 | `architecture/dependency-injection.md` |
| 網路 / API | `infrastructure/network.md` |
| Session / 登入狀態 | `infrastructure/session.md` |
| 全域設定（AppConfig） | `infrastructure/config.md` |
| Platform 差異 | `infrastructure/platform.md` |
| 命名 | `style/naming.md` |
| 排版、import、async | `style/coding-style.md` |
| 寫測試 | `patterns/testing.md` |
| 加套件 | `packages.md` |

## 不可妥協（永遠適用，不用查檔）

- ❌ 用 `Cubit` —— 一律 `Bloc<Event, State>`（→ [ADR-001](./adr/001-bloc-not-cubit.md)）
- ❌ `_buildXxx()` 拆 widget —— 抽獨立 class（→ [ADR-010](./adr/010-no-build-prefix-private-method.md)）
- ❌ Hardcoded 字串 —— 用 `context.t.*`（slang）
- ❌ Hardcoded asset path —— 用 `Assets.*`（flutter_gen）
- ❌ Domain 層 import Flutter —— 違反分層（→ [ADR-011](./adr/011-domain-zero-flutter-imports.md)）
- ❌ Features 互相 import —— 透過 domain interface 解耦（→ [ADR-012](./adr/012-features-no-cross-import.md)）
- ❌ Hand-edit 生成檔（`*.freezed.dart`、`*.g.dart`、`*.config.dart`）
- ✅ 命名：檔案 `snake_case`、class `PascalCase`、private `_camelCase`
- ✅ 所有參數 named parameters（除了 Widget `key`）（→ [ADR-008](./adr/008-named-parameters-everywhere.md)）
- ✅ Domain return type：`Future<Either<AppFailure, T>>`，void 用 `Unit`（→ [ADR-007](./adr/007-either-pattern-for-domain-results.md)）

## 立場聲明

本規範**對工具與模式有強烈立場**。如果你的團隊用不同的選擇（例如 Riverpod、GetX、Provider、Mobx），這份規範不適合直接套用。完整套件白名單見 [`packages.md`](./packages.md)，每項選擇的 rationale 見 [`adr/`](./adr/)。
