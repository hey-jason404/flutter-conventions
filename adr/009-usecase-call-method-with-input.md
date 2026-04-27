# ADR-009: UseCase `call()` Method with Input Class

## Status

Accepted | Date: 2024-05-20 | Supersedes: -

## Context

UseCase 設計有兩個邊界決策：

### 1. UseCase 的入口方法名

可選方案：

- `execute(...)` —— Java / 設計模式書常見
- `run(...)` —— 簡潔但跟 Dart `Future.run` 撞名
- `invoke(...)` —— Java reflection 風
- `call(...)` —— Dart 的 callable convention

`call()` 的優勢：呼叫端寫成 `useCase(input: ...)`，**看起來像「執行該 UseCase」**。

```dart
final result = await loginUseCase(input: input);    // ✅ 自然
final result = await loginUseCase.execute(input);   // ❌ 為什麼要說 execute？
```

### 2. 參數結構

**選項 A：散落 named parameters**

```dart
class LoginUseCase {
  Future<Either<AppFailure, User>> call({
    required String email,
    required String password,
  });
}
```

**選項 B：統一 Input class**

```dart
class LoginInput {
  const LoginInput({required this.email, required this.password});
  final String email;
  final String password;
}

class LoginUseCase {
  Future<Either<AppFailure, User>> call({required LoginInput input});
}
```

### 為何 Input class 更穩

- **加新參數不破壞 caller**：加進 Input class，所有 caller 自動沿用 Input 即可（改 Input class 簽章一處）
- **測試 setup 更乾淨**：建一個 `LoginInput` 物件，比建一堆 `email: ..., password: ...` 散落呼叫
- **跨 UseCase 風格統一**：所有 UseCase 都是 `call({required Input input})`，新人看一個就懂全部
- **複雜參數結構自然容納**：當 UseCase 需要 5+ 個參數，散落 named 變很長；Input class 自然把它組織在一起

### 即使單一參數也建 Input class？

是的，理由：

- 一致性 > 例外（特殊情境免 Input 會讓「什麼時候要 Input」變主觀判斷）
- 將來加參數成本是零（已經有 Input class）
- boilerplate 成本低（5 行 code）

### Input 用 `@freezed`？

不用。Input 是輕量資料容器，無需 `copyWith` / value-equality；同 [ADR-002](./002-bloc-event-plain-sealed-class.md) 對 Event 的論點。

## Decision

**所有 UseCase：**

1. 入口方法命名一律 `call(...)`
2. 有參數時必建對應 Input class（即使單一參數）
3. Input class 用 plain class，**禁** `@freezed`
4. Input 命名 `{VerbNoun}Input`（如 `LoginInput`、`SearchProductsInput`）
5. 簽章：`Future<Either<AppFailure, T>> call({required {VerbNoun}Input input})`，void 用 `Unit`

```dart
// ✅
class LoginInput {
  const LoginInput({required this.email, required this.password});
  final String email;
  final String password;
}

@injectable
class LoginUseCase {
  const LoginUseCase(this._repository);
  final AuthRepository _repository;

  Future<Either<AppFailure, User>> call({required LoginInput input}) {
    return _repository.login(email: input.email, password: input.password);
  }
}

// ✅ 無參數的 UseCase
@injectable
class LogoutUseCase {
  const LogoutUseCase(this._repository);
  final AuthRepository _repository;

  Future<Either<AppFailure, Unit>> call() => _repository.logout();
}
```

```dart
// ❌ 散落 raw 參數
class LoginUseCase {
  Future<Either<AppFailure, User>> call({
    required String email,
    required String password,
  });
}

// ❌ 自定方法名
class LoginUseCase {
  Future<Either<AppFailure, User>> execute({required LoginInput input});
}

// ❌ Input 用 @freezed
@freezed
class LoginInput with _$LoginInput {
  const factory LoginInput({...}) = _LoginInput;
}

// ❌ 命名後綴 Params
class LoginParams {...}
```

## Consequences

### Positive

- 跨 UseCase 命名 / 簽章完全一致
- 加參數不破壞 caller
- 測試 setup 統一
- callable convention 讓呼叫端讀起來自然

### Negative / Trade-offs

- 單一參數的 UseCase 也要建 Input class（5 行 boilerplate）
- 對只熟悉「直接傳參數」風格的 dev 需要 onboarding

## Alternatives Considered

### 自由命名（execute / run / invoke）

**為什麼不選**：團隊不一致；callable convention 是 Dart language feature，不用白不用。

### 散落 named parameters

**為什麼不選**：詳見 Context #2。加新參數會在簽章變動，caller 全要改。

### 複雜場景才建 Input、簡單場景直接傳

**為什麼不選**：「複雜」邊界主觀；後期常因需求變化要重構成 Input；不如一開始統一。

### Input 用 `@freezed`

**為什麼不選**：Input 是輕量 data container，不需要 `copyWith` / equality；codegen overhead 沒必要。同 [ADR-002](./002-bloc-event-plain-sealed-class.md)。

## Related

- 規範：[`architecture/domain-layer.md`](../architecture/domain-layer.md)、[`style/naming.md`](../style/naming.md)
- 相關 ADR：
  - [ADR-007: Either Pattern for Domain Results](./007-either-pattern-for-domain-results.md)
  - [ADR-008: Named Parameters Everywhere](./008-named-parameters-everywhere.md)
