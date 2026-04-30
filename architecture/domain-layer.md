# Domain Layer

> **TL;DR**：純 Dart 零 Flutter；UseCase 入口 `call()` + Input class；Repository 是 abstract interface；Entity 用 `@freezed abstract class`；Value Object 用 `@freezed abstract class` + `create()` factory 驗證。

---

## 不可妥協

| # | 規則 | ADR |
|---|---|---|
| 1 | Domain 層**零 Flutter import**、零第三方框架（除 fpdart） | [ADR-011](../adr/011-domain-zero-flutter-imports.md) |
| 2 | UseCase 入口一律 `call()` + Input class（即使單一參數） | [ADR-009](../adr/009-usecase-call-method-with-input.md) |
| 3 | Repository 是 `abstract interface class` | — |
| 4 | Entity 用 `@freezed abstract class`，禁 plain class | — |
| 5 | Domain return type：`Future<Either<AppFailure, T>>`（void 用 `Unit`） | [ADR-007](../adr/007-either-pattern-for-domain-results.md) |
| 6 | 所有方法參數用 named parameters | [ADR-008](../adr/008-named-parameters-everywhere.md) |

---

## 目錄結構

```text
features/{feature}/domain/
├── entities/
│   └── {entity}/
│       ├── {entity}.dart                     # source
│       └── {entity}.freezed.dart             # generated
├── enums/
│   └── {enum_name}.dart
├── repositories/
│   └── {feature}_repository.dart             # abstract interface class
├── usecases/
│   └── {verb_noun}/
│       ├── {verb_noun}_usecase.dart
│       └── {verb_noun}_input.dart            # 有參數時必有
└── value_objects/
    └── {value_object_name}/
        ├── {value_object_name}.dart
        └── {value_object_name}.freezed.dart
```

---

## UseCase

### 結構

- `@injectable` class
- 入口方法名一律 `call(...)`（callable convention）
- 一個 sub-directory per UseCase（`usecases/{verb_noun}/`）
- 簽章：`Future<Either<AppFailure, T>> call({required {VerbNoun}Input input})`

### Input class 必建（即使單一參數）

```dart
// ✅ Input class（即使只有一個 field 也建）
// usecases/login/login_input.dart
class LoginInput {
  const LoginInput({
    required this.email,
    required this.password,
  });

  final String email;
  final String password;
}

// usecases/login/login_usecase.dart
@injectable
class LoginUseCase {
  const LoginUseCase(this._repository);

  final AuthRepository _repository;

  Future<Either<AppFailure, User>> call({required LoginInput input}) {
    return _repository.login(email: input.email, password: input.password);
  }
}
```

```dart
// ❌ 沒有 Input class，散落 raw 參數
@injectable
class LoginUseCase {
  Future<Either<AppFailure, User>> call({
    required String email,
    required String password,
  });
}

// ❌ Input 用 @freezed
@freezed
class LoginInput with _$LoginInput {
  const factory LoginInput({
    required String email,
    required String password,
  }) = _LoginInput;
}

// ❌ 命名後綴 Params
class LoginParams { ... }
```

**為什麼錯**：

- 散落 raw 參數：之後加參數會破壞所有 caller、命名爆炸
- `@freezed` on Input：Input 是輕量 data container，不需要 `copyWith` / value-equality；codegen overhead 沒必要
- 後綴 `Params`：團隊不一致；統一用 `Input`（→ ADR-009）

### 入口必須是 `call()`

Dart callable convention：寫成 `useCase(input: ...)` 看起來像「執行該 UseCase」，比 `useCase.execute(...)` / `useCase.run(...)` 跨團隊一致。

```dart
// ✅ 呼叫端
final result = await _loginUseCase(input: input);

// ❌ 自定方法名
class LoginUseCase {
  Future<Either<AppFailure, User>> execute({required LoginInput input});
}
final result = await _loginUseCase.execute(input: input);
```

→ 詳見 [ADR-009](../adr/009-usecase-call-method-with-input.md)

### Void operation 用 `Unit`

```dart
// ✅ void operation 用 Unit
@injectable
class LogoutUseCase {
  const LogoutUseCase(this._repository);
  final AuthRepository _repository;

  Future<Either<AppFailure, Unit>> call() => _repository.logout();
}
```

`Unit` 來自 `fpdart`，等於 void 但保持 `Either` 型別一致。**不要用** `Future<Either<AppFailure, void>>`（type system 處理 void 的特性會跟 `Either` 對打）。

---

## Entity

### 規則

- **一般 Entity**：`@freezed abstract class`
- **Union Entity**（多型）：`@freezed sealed class`
- **不要加 suffix**（`User` 不是 `UserEntity`）—— 命名乾淨
- 不可變、value-equality 自動由 freezed 產生

```dart
// ✅ 一般 Entity
@freezed
abstract class User with _$User {
  const factory User({
    required String id,
    required String email,
    required String displayName,
  }) = _User;
}

// ✅ Union Entity（多型）
@freezed
sealed class PaymentMethod with _$PaymentMethod {
  const factory PaymentMethod.creditCard({
    required String maskedNumber,
    required String holderName,
  }) = CreditCardPayment;

  const factory PaymentMethod.bankTransfer({
    required String accountNumber,
  }) = BankTransferPayment;
}

// usage
final method = PaymentMethod.creditCard(maskedNumber: '****1234', holderName: 'Jason');
final desc = switch (method) {
  CreditCardPayment(:final maskedNumber) => 'Card $maskedNumber',
  BankTransferPayment(:final accountNumber) => 'Bank $accountNumber',
};
```

```dart
// ❌ Entity 用 plain class
class User {
  const User({required this.id, required this.email, required this.displayName});
  final String id;
  final String email;
  final String displayName;
  // ... 手刻 ==、hashCode、copyWith ?
}

// ❌ Entity 加 Entity suffix
class UserEntity { ... }
```

**為什麼錯**：plain class 要手刻 `==` / `hashCode` / `copyWith`，容易遺漏新 field；suffix `Entity` 是冗詞，domain 裡 class 預設就是 entity。

---

## Value Object

封裝**有業務驗證規則**的 domain 欄位（email 格式、密碼強度、訂單總額範圍...）。

### 結構：兩種 constructor

- **`const` constructor**：直接構造，**不驗證**，給 Data 層 Mapper 從可信來源 build
- **`create()` factory**：驗證 + 構造，回 `Either<FieldFailure, T>`

```dart
@freezed
abstract class EmailAddress with _$EmailAddress {
  const factory EmailAddress(String value) = _EmailAddress;

  const EmailAddress._();

  static Either<FieldFailure, EmailAddress> create(String raw) {
    final trimmed = raw.trim();
    if (trimmed.isEmpty) return const Left(RequiredFieldFailure());
    if (!trimmed.contains('@')) return const Left(InvalidEmailFormatFailure());
    return Right(EmailAddress(trimmed));
  }
}
```

```dart
// ✅ UseCase 用 Value Object 驗證再呼叫 Repository
@injectable
class LoginUseCase {
  const LoginUseCase(this._repository);
  final AuthRepository _repository;

  Future<Either<AppFailure, User>> call({required LoginInput input}) {
    return EmailAddress.create(input.email).fold(
      (fieldFailure) => Future.value(Left(fieldFailure)),
      (email) => _repository.login(email: email.value, password: input.password),
    );
  }
}
```

### 規則

- **只驗證業務規則**（格式、範圍、domain constraint），不驗 UI 層的事（譬如「使用者沒填」這種 UI state）
- 一個 sub-directory per Value Object（`value_objects/{value_object_name}/`）

---

## Repository Interface

### 規則

- **`abstract interface class {Feature}Repository`**
- 簽章只用 **domain types**（Entity、Value Object、`Either`）—— **禁出現 DTO**
- 命名 `{Feature}Repository`（不帶 `Impl`）

```dart
// ✅ Repository interface
abstract interface class AuthRepository {
  Future<Either<AppFailure, User>> login({
    required String email,
    required String password,
  });

  Future<Either<AppFailure, Unit>> logout();

  Future<Either<AppFailure, User>> getCurrentUser();
}
```

```dart
// ❌ Repository 簽章用 DTO
abstract interface class AuthRepository {
  Future<Either<AppFailure, LoginResponseDto>> login({         // ❌ 用了 DTO
    required LoginRequestDto request,                          // ❌ 用了 DTO
  });
}
```

**為什麼錯**：DTO 是 Data 層概念；讓 Domain 知道 DTO = Domain 依賴 Data 的 type，違反依賴方向。

```dart
// ❌ Repository 簽章用 Flutter type
abstract interface class AuthRepository {
  Future<Either<AppFailure, BuildContext>> login({...});       // ❌
}
```

**為什麼錯**：Domain 層零 Flutter import。

---

## Enum

放 `domain/enums/`，值用 `camelCase`：

```dart
// ✅ camelCase 值
enum LoginStatus { initial, loading, success, failure }

enum UserRole { admin, member, guest }
```

```dart
// ❌ PascalCase / SCREAMING_SNAKE_CASE
enum LoginStatus { Initial, Loading, Success, Failure }
enum LoginStatus { INITIAL, LOADING, SUCCESS, FAILURE }
```

---

## Domain 零 Flutter 邊界

Domain 層**唯一允許**的 import：

- `dart:` 標準庫（`dart:async`、`dart:convert`、...）
- `package:fpdart/fpdart.dart`（`Either`、`Option`、`Unit`）
- `package:freezed_annotation/freezed_annotation.dart`（codegen annotation）
- 同 feature 的 domain 全部
- 其他 feature 的 domain entity / enum / VO / UseCase（Repository interface 不可跨 feature import）

**禁止** import：

- `package:flutter/...`
- `dart:ui`
- `package:dio/...`、`package:http/...`
- 任何 UI / network / persistence 套件
- 同 feature 的 data 層

```dart
// ✅ 允許
import 'dart:async';
import 'package:fpdart/fpdart.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import '../entities/user.dart';
import '../../../auth/domain/entities/user/user.dart';                                    // 跨 feature entity ✅
import '../../../auth/domain/usecases/get_current_user/get_current_user_usecase.dart';    // 跨 feature UseCase ✅

// ❌ 禁止
import 'package:flutter/material.dart';
import 'dart:ui';
import 'package:dio/dio.dart';
import '../../data/datasources/remote/auth_remote_datasource.dart';
import '../../../auth/domain/repositories/auth_repository.dart';                           // 跨 feature Repository interface ❌
```

→ 詳見 [ADR-011](../adr/011-domain-zero-flutter-imports.md)、[ADR-014](../adr/014-features-cross-import-rules.md)

---

## 常見錯誤

```dart
// ❌ UseCase 沒 Input class
class LoginUseCase {
  Future<Either<AppFailure, User>> call({
    required String email,
    required String password,
  });
}
```

```dart
// ❌ Repository 用 specific exception
abstract interface class AuthRepository {
  Future<User> login({required String email, required String password});
  // throws NetworkException, ValidationException...
}
```
**為什麼錯**：Domain 層應透過 `Either` 顯式回 failure；throw exception 是 implicit 錯誤路徑，型別系統不會強制 caller 處理。

```dart
// ❌ Entity 帶業務邏輯
@freezed
abstract class User with _$User {
  const factory User({required String id, required String email}) = _User;

  const User._();

  bool get isValidEmail => email.contains('@');     // ❌ 應該是 Value Object
}
```
**為什麼錯**：Entity 應只描述「是什麼」，業務邏輯放 UseCase / Value Object。

---

## Related

- [`overview.md`](./overview.md) — 三層架構、依賴方向
- [`data-layer.md`](./data-layer.md) — Repository 實作、Mapper、DTO
- [`patterns/error-handling.md`](../patterns/error-handling.md) — `Either` / `AppFailure` / `FieldFailure`
- [`patterns/code-generation.md`](../patterns/code-generation.md) — `freezed` 規範
- [ADR-007](../adr/007-either-pattern-for-domain-results.md)、[ADR-009](../adr/009-usecase-call-method-with-input.md)、[ADR-011](../adr/011-domain-zero-flutter-imports.md)、[ADR-014](../adr/014-features-cross-import-rules.md)
