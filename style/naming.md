# 命名規範

> **TL;DR**：檔名 `snake_case`、class `PascalCase`、變數和方法 `camelCase`、私有成員 `_camelCase`。BLoC/UseCase/DTO 有額外的命名 pattern。

---

## 不可妥協

| # | 規則 | ADR |
|---|---|---|
| 1 | 檔名一律 `snake_case.dart` | — |
| 2 | Class 一律 `PascalCase` | — |
| 3 | 變數 / 方法一律 `camelCase`，私有 `_camelCase` | — |
| 4 | BLoC Event 過去式，無 `Event` suffix | [ADR-004](../adr/004-bloc-event-past-tense-naming.md) |
| 5 | 命令式動詞放 SideEffect 不放 Event | [ADR-005](../adr/005-imperative-actions-as-side-effect.md) |

---

## 完整命名表

| 元素 | 規範 | 範例 |
|---|---|---|
| 檔案 | `snake_case.dart` | `login_bloc.dart` |
| Class | `PascalCase` | `LoginBloc` |
| 變數 / 方法 | `camelCase` | `isLoading`、`onSubmit` |
| 私有成員 | `_camelCase` | `_loginUseCase`、`_email` |
| 常數 | `camelCase` | `maxRetryCount` |
| Enum 值 | `camelCase` | `enum LoginStatus { initial, loading, success, failure }` |
| BLoC Event | `{Page}{Context?}{PastVerb}` | `LoginSubmitted`、`LoginEmailChanged` |
| BLoC State | `{Page}Initial / Loading / Success / Failure` | `LoginSuccess` |
| BLoC SideEffect | `{Page}{ImperativeVerb}{Object?}` | `LoginNavigateToHome`、`LoginShowBiometricError` |
| Repository 介面 | `{Feature}Repository` | `AuthRepository` |
| Repository 實作 | `{Feature}RepositoryImpl` | `AuthRepositoryImpl` |
| UseCase | `{VerbNoun}UseCase` | `LoginUseCase` |
| Response DTO | `{Entity}ResponseDto` | `UserResponseDto` |
| Request DTO | `{Action}RequestDto` | `LoginRequestDto` |
| Local DTO（持久化） | `{Entity}LocalDto` | `UserLocalDto` |
| Mapper | `{Entity}Mapper` | `UserMapper` |
| DI Module | `{Feature}Module` | `AuthModule` |

---

## 細節說明

### 檔名與 Class 名對應

一個 `.dart` 檔通常含一個主要 class，檔名是 class 的 `snake_case` 形式：

```text
✅ login_bloc.dart        contains  class LoginBloc
✅ user_response_dto.dart contains  class UserResponseDto
✅ auth_repository.dart   contains  abstract interface class AuthRepository

❌ LoginBloc.dart         (PascalCase 檔名違反 Dart 慣例)
❌ login-bloc.dart        (kebab-case 檔名違反 Dart 慣例)
```

### Private 命名

Dart 用 `_` 前綴標示「library-private」（不是 class-private）：

```dart
// ✅ private member of class
class LoginBloc {
  final LoginUseCase _loginUseCase;
  void _handleSubmit(LoginSubmitted event) { ... }
}

// ✅ library-private top-level
const _maxRetries = 3;
class _LocalHelper { ... }
```

### Enum 值

```dart
// ✅ camelCase
enum LoginStatus { initial, loading, success, failure }

// ❌ PascalCase
enum LoginStatus { Initial, Loading, Success, Failure }

// ❌ SCREAMING_SNAKE_CASE
enum LoginStatus { INITIAL, LOADING, SUCCESS, FAILURE }
```

---

## Related

- [`coding-style.md`](./coding-style.md) — 排版、import、async/await
- [`patterns/state-management.md`](../patterns/state-management.md) — BLoC 命名 pattern 完整規則
- [`architecture/domain-layer.md`](../architecture/domain-layer.md) — UseCase / Repository 命名 pattern
