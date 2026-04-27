# ADR-007: Either Pattern for Domain Results

## Status

Accepted | Date: 2024-05-05 | Supersedes: -

## Context

Domain 層的方法（Repository interface、UseCase）回傳資料時必然伴隨「可能失敗」。處理失敗有三種主流模式：

### 1. 拋 Exception（Java 風）

```dart
abstract interface class AuthRepository {
  Future<User> login({required String email, required String password});
  // throws NetworkException, UnauthorizedException, ValidationException, ...
}
```

**問題**：

- 型別系統**不強制 caller 處理錯誤**（編譯期不擋忘 try-catch）
- exception 類型沒在簽章上，需讀 doc / source 才知道哪些 exception
- exception 是 implicit control flow，理解 happy path 要在腦中 mute exception 路徑

### 2. nullable + bool（C 風）

```dart
Future<(User?, AppFailure?)> login({required String email, required String password});
```

**問題**：

- 兩個 nullable 之間的關係靠約定（caller 必須記得「有值就沒錯、有錯就沒值」）
- 編譯期不擋同時取 `user` 跟 `failure`
- 不直觀

### 3. Either（FP 風）

```dart
Future<Either<AppFailure, User>> login({...});

// caller 必 fold 才能取值
result.fold(
  (failure) => emit(LoginFailure(failure: failure)),
  (user) => emit(LoginSuccess(user: user)),
);
```

**優勢**：

- 失敗顯式在 return type
- 編譯期強制 caller 處理 failure 路徑（不 fold 取不到 user）
- happy path / unhappy path 對稱、明確

### `fpdart` 提供 production-ready 實作

- 完整的 `Either<L, R>` API
- `Option`、`Unit` 等 FP 工具
- 維護健康度好（dartz 的後續維護版本）

## Decision

**Domain 層所有 public 方法回 `Future<Either<AppFailure, T>>`。void operation 用 `Future<Either<AppFailure, Unit>>`。**

```dart
// ✅ Repository
abstract interface class AuthRepository {
  Future<Either<AppFailure, User>> login({
    required String email,
    required String password,
  });

  Future<Either<AppFailure, Unit>> logout();
}

// ✅ UseCase
@injectable
class LoginUseCase {
  Future<Either<AppFailure, User>> call({required LoginInput input}) =>
      _repository.login(email: input.email, password: input.password);
}

// ✅ BLoC handler
final result = await _loginUseCase(input: input);
result.fold(
  (failure) => emit(LoginFailure(failure: failure)),
  (user) => emit(LoginSuccess(user: user)),
);
```

## Consequences

### Positive

- 失敗模式 explicit 在型別簽章
- 編譯期強制處理 failure path
- BLoC handler 可預期、可測試
- 跨 RepositoryImpl exception 處理集中於 `AppFailureMapper`，DataSource 拋 typed exception → Repository catch → Either 轉換清晰

### Negative / Trade-offs

- 第三方 SDK 多是 Future-throw 模式，Repository 邊界要做轉換（一次性投資）
- FP 不熟悉的 dev 需要 onboarding（`fold` / `map` / `flatMap`）
- 相比 Future-throw 多寫一些樣板（`Right(...)` / `Left(...)` 包裝）

## Alternatives Considered

### Future-throw（拋 Exception）

**為什麼不選**：詳見 Context #1。錯誤處理 implicit、型別系統不強制。

### 自定 Result class

```dart
sealed class Result<T> {
  const Result();
}
final class Success<T> extends Result<T> { const Success(this.value); final T value; }
final class Failure<T> extends Result<T> { const Failure(this.failure); final AppFailure failure; }
```

**為什麼不選**：

- 重複輪子，fpdart 已提供成熟實作
- 自定 class 缺少 `map` / `flatMap` / `fold` 等 FP combinator
- 跟社群 Flutter Clean Architecture 範例普遍用 `Either` 不一致

### nullable + bool

**為什麼不選**：詳見 Context #2，靠約定不靠型別。

## Related

- 規範：[`patterns/error-handling.md`](../patterns/error-handling.md)、[`architecture/domain-layer.md`](../architecture/domain-layer.md)、[`architecture/data-layer.md`](../architecture/data-layer.md)
- 套件：[`packages.md`](../packages.md)（fpdart 在白名單）
