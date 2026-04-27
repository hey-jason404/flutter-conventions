# ADR-008: Named Parameters Everywhere

## Status

Accepted | Date: 2024-05-12 | Supersedes: -

## Context

Dart 的方法 / constructor 參數有兩種：positional 跟 named。

### Positional 的問題

```dart
class LoginUseCase {
  Future<Either<AppFailure, User>> call(String email, String password);
}

// 呼叫端
final result = await loginUseCase('foo@bar.com', 'pass123');
```

問題：

1. **可讀性差**：`loginUseCase('foo@bar.com', 'pass123')` 看不出哪個是 email、哪個是 password
2. **多參數時容易 swap**：`login(password, email)` 編譯通過（兩者都是 String），但功能錯誤
3. **加新參數破壞 caller**：function 加第三個參數，所有 call site 必須改
4. **boolean 參數最糟**：`featureToggle(true)` 完全看不出語意

### Named 的優勢

```dart
class LoginUseCase {
  Future<Either<AppFailure, User>> call({
    required String email,
    required String password,
  });
}

// 呼叫端
final result = await loginUseCase(
  email: 'foo@bar.com',
  password: 'pass123',
);
```

優勢：

1. **呼叫端意圖明確**：每個參數都有 label
2. **不會 swap 順序**：name-based 而非 position-based
3. **加新參數安全**：只要新參數有 default value 或 nullable 不破壞 caller
4. **boolean 自帶語意**：`featureToggle(enabled: true)`

### Widget `key` 是唯一例外

Flutter framework 的 Widget `key` 在 super constructor 是 positional：

```dart
const LoginPage({super.key});
```

這是 Flutter convention，無法改。所有自定 widget 沿用即可。

## Decision

**所有 method、function、constructor 參數一律使用 named parameters，唯一例外是 Widget 的 `key`。**

```dart
// ✅
class LoginUseCase {
  Future<Either<AppFailure, User>> call({
    required String email,
    required String password,
  });
}

LoginRequestDto({
  required this.email,
  required this.password,
});

const LoginPage({super.key});                     // Widget key 例外
```

```dart
// ❌
class LoginUseCase {
  Future<Either<AppFailure, User>> call(String email, String password);
}

LoginRequestDto(this.email, this.password);
```

## Consequences

### Positive

- 呼叫端可讀性大幅提升
- ordering bug 消失
- 加參數不破壞 caller（前提是 named + default 或 nullable）
- IDE 自動補全更友善（labels 出現）
- code review 時意圖明確

### Negative / Trade-offs

- 寫起來稍微囉嗦（每個 call site 寫 `email: 'foo'`）
- 對只接觸過其他語言（如 Java）的 dev 可能不習慣
- 簡單情境的 helper function 也要 named（一致性 > 簡潔性）

## Alternatives Considered

### 預設 positional，「重要的」才 named

**為什麼不選**：

- 「重要」是主觀判斷
- 邊界一旦鬆動，團隊不一致
- code review 時要每次糾結「這個算重要嗎」
- 不如統一規則

### Positional + 第一個 String 參數放 label

**為什麼不選**：偽 named；型別系統不強制；沒 named parameters 的 IDE 提示

## Related

- 規範：[`style/coding-style.md`](../style/coding-style.md)
- 相關 ADR：[ADR-009: UseCase Call Method with Input Class](./009-usecase-call-method-with-input.md)（Input class 也是基於同思路：明確、加參數不破壞 caller）
