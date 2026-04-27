# Coding Style

> **TL;DR**：用 `dart format` 統一排版，所有參數一律 named，預設 `final`／`const`，避免 `dynamic`，import 分三段空行隔開。

---

## 不可妥協

| # | 規則 | ADR |
|---|---|---|
| 1 | 所有參數一律 named（除 Widget `key`） | [ADR-008](../adr/008-named-parameters-everywhere.md) |
| 2 | `dart format` 為唯一排版來源 | — |
| 3 | 預設 `const` / `final`，避免 `var` | — |
| 4 | 公開 API 顯式宣告型別；本地變數靠推斷 | — |
| 5 | 多行 collection / parameter list 加 trailing comma | — |
| 6 | Import 分三段（dart / package / project），段間空行 | — |

---

## 格式化（`dart format`）

`dart format` 是團隊唯一的排版來源 —— **不要手動微調 dart format 已處理的東西**。

```bash
# 提交前
dart format --output=none --set-exit-if-changed .

# 套用
dart format .
```

**行寬**：120 字元（Dart 3.x 預設）。120 以下不要手動斷行讓 formatter 處理。

---

## Import 順序

依下列順序分三段，**段間空一行**：

```dart
// 1. Dart SDK
import 'dart:async';
import 'dart:convert';

// 2. Flutter & 第三方 packages
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:go_router/go_router.dart';

// 3. Project（相對路徑）
import '../../domain/usecases/login_usecase.dart';
import '../bloc/login_bloc.dart';
```

```dart
// ❌ 三段混在一起
import 'package:flutter/material.dart';
import '../bloc/login_bloc.dart';
import 'dart:async';
import 'package:flutter_bloc/flutter_bloc.dart';
```

**為什麼錯**：閱讀 import block 失去結構，看不出哪些是 SDK / 第三方 / 自家 code。

---

## Named parameters

**所有 method、constructor 的參數都 named**，唯一例外是 Widget 的 `key`（Flutter convention）。

```dart
// ✅ named
class LoginUseCase {
  Future<Either<AppFailure, User>> call({required LoginInput input});
}

LoginRequestDto({
  required this.email,
  required this.password,
});

// ✅ Widget key 是唯一例外
const LoginPage({super.key});
```

```dart
// ❌ positional
class LoginUseCase {
  Future<Either<AppFailure, User>> call(String email, String password);
}

LoginRequestDto(this.email, this.password);
```

**為什麼錯**：positional 在呼叫端 `login('foo@bar.com', 'pass123')` 完全看不出哪個是 email、哪個是 password；多參數時容易 swap 順序產生 bug；之後加新參數會破壞所有 caller。

詳見 [ADR-008](../adr/008-named-parameters-everywhere.md)。

---

## `const` / `final` / `var`

**預設策略**：能 `const` 就 `const`，不行就 `final`，最後才 `var`。

```dart
// ✅ const
const SizedBox(height: 16);
const EdgeInsets.all(16);
const LoginInitial();

// ✅ final（runtime 賦值的不可變引用）
final user = await _repository.getUser(id);
final config = AppConfig.instance;

// ✅ var（極少數，需要 reassign 的本地變數）
var retries = 0;
while (retries < maxRetries) { retries++; ... }
```

```dart
// ❌ 用 var 但其實沒 reassign
var user = await _repository.getUser(id);

// ❌ 沒加 const 的 widget literal（影響 rebuild 最佳化）
SizedBox(height: 16);
EdgeInsets.all(16);
```

**為什麼錯**：`const` 失去 = 失去 widget rebuild 最佳化；`var` 而非 `final` 暗示 mutability 但其實不會改，造成讀者誤解。

---

## 型別宣告

- **公開 API**（method / function 的 return type、public field）→ **顯式宣告**
- **本地變數**（method body 內的 `final` / `var`）→ **靠型別推斷**

```dart
// ✅ 公開 method 顯式 return type
Future<Either<AppFailure, User>> login({
  required String email,
  required String password,
}) async {
  final result = await _repository.login(email: email, password: password); // 推斷
  return result;
}

// ✅ 本地變數推斷
final users = await _repository.getAllUsers();
final filtered = users.where((u) => u.isActive).toList();
```

```dart
// ❌ 公開 method 沒寫 return type
login(String email, String password) async {
  return _repository.login(email, password);
}

// ❌ 本地變數寫死型別（多餘）
final List<User> users = await _repository.getAllUsers();
```

---

## 避免 `dynamic`

`dynamic` 等於「我放棄型別檢查」。除了極少數場景（JSON parse 中間態、interop），不應出現。

```dart
// ✅ 顯式型別
Future<Either<AppFailure, User>> login({required String email, required String password});

// ❌ dynamic
Future login(dynamic params);
```

JSON parse 邊界用 `Map<String, dynamic>` 是必要的（DTO 從 JSON 來），但**不要讓 dynamic 滲入業務 code**：

```dart
// ✅ JSON 解析在 DTO factory 裡，外部見到的是 typed model
factory UserResponseDto.fromJson(Map<String, dynamic> json) => _$UserResponseDtoFromJson(json);

// ❌ dynamic 流到業務 code
class LoginUseCase {
  Future<dynamic> call(dynamic input) { ... }
}
```

---

## Trailing commas

多行 parameter list、collection literal、constructor argument 一律加 trailing comma：

```dart
// ✅
return Padding(
  padding: const EdgeInsets.all(16),
  child: Column(
    children: [
      const Text('Hello'),
      const SizedBox(height: 8),
    ],
  ),
);

LoginRequestDto({
  required this.email,
  required this.password,
});
```

```dart
// ❌ 沒有 trailing comma
return Padding(
  padding: const EdgeInsets.all(16),
  child: Column(
    children: [
      const Text('Hello'),
      const SizedBox(height: 8)
    ]
  )
);
```

**為什麼**：trailing comma 讓 `dart format` 把每個元素獨立一行 → diff 改一個元素只動一行，PR review 清晰。

---

## Async / await

**用 `await`，避免 `.then()`**。

```dart
// ✅
final result = await _useCase(input: input);
result.fold(
  (failure) => emit(LoginFailure(failure: failure)),
  (user) => emit(LoginSuccess(user: user)),
);

// ❌
_useCase(input: input).then((result) {
  result.fold(
    (failure) => emit(LoginFailure(failure: failure)),
    (user) => emit(LoginSuccess(user: user)),
  );
});
```

**為什麼錯**：`.then()` callback nesting 影響可讀性；error 處理需要再串 `.catchError()`，跟 `try-catch` 風格混在一起。

**Future 一定要被消費**（`await`、回傳、`.then()`，不能 fire-and-forget）：

```dart
// ✅ awaited
await _logger.log(event);

// ✅ returned
Future<void> emitEvent() => _logger.log(event);

// ❌ 漏掉 await，warning 也不該忽略
_logger.log(event);  // analyzer 會 warn unawaited_futures
```

---

## File organization

每個 `.dart` 檔**一個 public class**。私有 helper class 同檔可放，前提是只在這個檔用：

```dart
// ✅ login_page.dart 一個 public + 多個 private helpers
class LoginPage extends StatelessWidget {
  const LoginPage({super.key});
  @override
  Widget build(BuildContext context) => const _LoginForm();
}

class _LoginForm extends StatelessWidget {     // private, only here
  const _LoginForm();
  @override
  Widget build(BuildContext context) => const SizedBox.shrink();
}

class _SubmitButton extends StatelessWidget {  // private, only here
  const _SubmitButton();
  @override
  Widget build(BuildContext context) => const SizedBox.shrink();
}
```

```dart
// ❌ 兩個不相關的 public class 同檔
class LoginPage extends StatelessWidget { ... }
class RegisterPage extends StatelessWidget { ... }   // 應移到 register_page.dart
```

---

## Related

- [`naming.md`](./naming.md) — 命名規範完整表
- [`patterns/code-generation.md`](../patterns/code-generation.md) — `freezed` / `injectable` 等 codegen 規則
- [ADR-008: Named Parameters Everywhere](../adr/008-named-parameters-everywhere.md)
