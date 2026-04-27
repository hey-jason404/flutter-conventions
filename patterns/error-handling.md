# Error Handling

> **TL;DR**：DataSource throw exception → RepositoryImpl catch + map → 回 `Either<AppFailure, T>`。Domain / Presentation 層**永不 catch exception**。錯誤映射集中於 `AppFailureMapper`。

---

## 不可妥協

| # | 規則 | ADR |
|---|---|---|
| 1 | Domain 公開方法回 `Future<Either<AppFailure, T>>`，void 用 `Unit` | [ADR-007](../adr/007-either-pattern-for-domain-results.md) |
| 2 | RepositoryImpl 統一 try-catch、單一 mapper 邊界 | — |
| 3 | Presentation 層**永不直接 catch exception** | — |
| 4 | DataSource 拋 typed exception，不 wrap Either | — |
| 5 | `AppFailure` / `FieldFailure` 是 sealed class | — |

---

## 錯誤模型

### `AppFailure` —— 系統 / 業務錯誤

`sealed class`，子類列舉常見錯誤：

```dart
sealed class AppFailure {
  const AppFailure({required this.message});
  final String message;
}

final class NetworkAppFailure extends AppFailure {
  const NetworkAppFailure({super.message = 'Network error'});
}

final class UnauthorizedAppFailure extends AppFailure {
  const UnauthorizedAppFailure({super.message = 'Unauthorized'});
}

final class ServerAppFailure extends AppFailure {
  const ServerAppFailure({
    required this.statusCode,
    super.message = 'Server error',
  });
  final int statusCode;
}

final class NotFoundAppFailure extends AppFailure {
  const NotFoundAppFailure({super.message = 'Resource not found'});
}

final class ValidationAppFailure extends AppFailure {
  const ValidationAppFailure({required super.message});
}

final class UnknownAppFailure extends AppFailure {
  const UnknownAppFailure({super.message = 'Unknown error'});
}
```

### `FieldFailure` —— Value Object 驗證失敗

用於 Value Object 的 `create()` factory 回傳：

```dart
sealed class FieldFailure extends AppFailure {
  const FieldFailure({required super.message});
}

final class RequiredFieldFailure extends FieldFailure {
  const RequiredFieldFailure({super.message = 'This field is required'});
}

final class InvalidEmailFormatFailure extends FieldFailure {
  const InvalidEmailFormatFailure({super.message = 'Invalid email format'});
}

final class PasswordTooWeakFailure extends FieldFailure {
  const PasswordTooWeakFailure({super.message = 'Password too weak'});
}

final class ValueOutOfRangeFailure extends FieldFailure {
  const ValueOutOfRangeFailure({required super.message});
}
```

> ⚠️ `FieldFailure` 繼承 `AppFailure`，這樣 Value Object 驗證失敗可以直接被 `Either<AppFailure, T>` 容納。

---

## 錯誤流動鏈

```text
DataSource throws exception
        │
        ▼
RepositoryImpl catch
        │
        ▼
AppFailureMapper.toFailure(error) → AppFailure
        │
        ▼
Left(AppFailure)  ──►  Either<AppFailure, T>
        │
        ▼
UseCase 透傳
        │
        ▼
BLoC fold(failure → emit Failure state, value → emit Success)
        │
        ▼
Page 看 state 渲染 UI
```

---

## DataSource：throw exception

DataSource 只負責 IO，遇錯**直接 throw**，不要包 Either：

```dart
@LazySingleton(as: AuthRemoteDataSource)
class AuthRemoteDataSourceImpl implements AuthRemoteDataSource {
  const AuthRemoteDataSourceImpl({required Dio dio}) : _dio = dio;
  final Dio _dio;

  @override
  Future<LoginResponseDto> login({required LoginRequestDto request}) async {
    try {
      final response = await _dio.post<Map<String, dynamic>>(
        AuthApiPaths.login,
        data: request.toJson(),
      );
      return LoginResponseDto.fromJson(response.data!);
    } on DioException catch (e) {
      throw _mapDioException(e);                    // 轉成 typed exception
    }
  }

  Exception _mapDioException(DioException e) {
    if (e.type == DioExceptionType.connectionError) return const NetworkException();
    if (e.response?.statusCode == 401) return const UnauthorizedException();
    if (e.response?.statusCode == 404) return const NotFoundException();
    return ServerException(statusCode: e.response?.statusCode ?? 500);
  }
}
```

DataSource 內部把第三方框架的 exception（`DioException`）轉成自家 typed exception：

```dart
sealed class NetworkException implements Exception {
  const NetworkException();
}

final class NoNetworkException extends NetworkException {
  const NoNetworkException();
}

final class TimeoutException extends NetworkException {
  const TimeoutException();
}

final class UnauthorizedException extends NetworkException {
  const UnauthorizedException();
}

final class ServerException extends NetworkException {
  const ServerException({required this.statusCode});
  final int statusCode;
}

final class NotFoundException extends NetworkException {
  const NotFoundException();
}
```

---

## RepositoryImpl：統一 catch + map

RepositoryImpl 是 exception → AppFailure 的**唯一邊界**。**單一 catch block via `AppFailureMapper`**：

```dart
@LazySingleton(as: AuthRepository)
class AuthRepositoryImpl implements AuthRepository {
  const AuthRepositoryImpl({required AuthRemoteDataSource remoteDataSource})
      : _remoteDataSource = remoteDataSource;

  final AuthRemoteDataSource _remoteDataSource;

  @override
  Future<Either<AppFailure, User>> login({
    required String email,
    required String password,
  }) async {
    try {
      final dto = await _remoteDataSource.login(
        request: LoginRequestDto(email: email, password: password),
      );
      return Right(UserMapper.fromResponseDto(dto));
    } on Object catch (error) {
      return Left(AppFailureMapper.toFailure(error));
    }
  }
}
```

### `AppFailureMapper`

集中處理 exception → AppFailure 映射：

```dart
abstract final class AppFailureMapper {
  static AppFailure toFailure(Object error) {
    if (error is NetworkException) return _fromNetworkException(error);
    if (error is FormatException) return ValidationAppFailure(message: error.message);
    return UnknownAppFailure(message: error.toString());
  }

  static AppFailure _fromNetworkException(NetworkException e) {
    return switch (e) {
      NoNetworkException() => const NetworkAppFailure(),
      TimeoutException() => const NetworkAppFailure(message: 'Request timeout'),
      UnauthorizedException() => const UnauthorizedAppFailure(),
      ServerException(:final statusCode) => ServerAppFailure(statusCode: statusCode),
      NotFoundException() => const NotFoundAppFailure(),
    };
  }
}
```

### ❌ 在 RepositoryImpl catch 特定 exception

```dart
@override
Future<Either<AppFailure, User>> login({
  required String email,
  required String password,
}) async {
  try {
    final dto = await _remoteDataSource.login(
      request: LoginRequestDto(email: email, password: password),
    );
    return Right(UserMapper.fromResponseDto(dto));
  } on NoNetworkException {
    return const Left(NetworkAppFailure());
  } on UnauthorizedException {
    return const Left(UnauthorizedAppFailure());
  } on ServerException catch (e) {
    return Left(ServerAppFailure(statusCode: e.statusCode));
  } catch (e) {
    return Left(UnknownAppFailure(message: e.toString()));
  }
}
```

**為什麼錯**：每個 RepositoryImpl 都要重複這套 catch，維護成本高；新增 exception type 要找全部 RepositoryImpl 改；應集中於 `AppFailureMapper`。

---

## Domain 透傳

UseCase 通常只透傳 Repository 的 `Either`，不額外處理 failure：

```dart
@injectable
class LoginUseCase {
  const LoginUseCase(this._repository);
  final AuthRepository _repository;

  Future<Either<AppFailure, User>> call({required LoginInput input}) {
    return _repository.login(email: input.email, password: input.password);
  }
}
```

UseCase 需要驗證輸入時，Value Object 的 `FieldFailure` 也是 `AppFailure`，可直接回：

```dart
@injectable
class LoginUseCase {
  const LoginUseCase(this._repository);
  final AuthRepository _repository;

  Future<Either<AppFailure, User>> call({required LoginInput input}) {
    return EmailAddress.create(input.email).fold(
      (fieldFailure) => Future.value(Left(fieldFailure)),       // 驗證失敗直接回
      (email) => _repository.login(
        email: email.value,
        password: input.password,
      ),
    );
  }
}
```

---

## BLoC：fold Either

BLoC handler 透過 `fold` 分支處理：

```dart
Future<void> _onLoginSubmitted(
  LoginSubmitted event,
  Emitter<LoginState> emit,
) async {
  emit(const LoginLoading());

  final result = await _loginUseCase(
    input: LoginInput(email: event.email, password: event.password),
  );

  result.fold(
    (failure) {
      emit(LoginFailure(failure: failure));
      emitSideEffect(LoginShowError(message: failure.message));
    },
    (user) {
      emit(LoginSuccess(user: user));
      emitSideEffect(const LoginNavigateToHome());
    },
  );
}
```

```dart
// ❌ BLoC 用 try-catch
Future<void> _onLoginSubmitted(LoginSubmitted event, Emitter<LoginState> emit) async {
  emit(const LoginLoading());
  try {
    final user = await _loginUseCase(...);
    emit(LoginSuccess(user: user));
  } catch (e) {
    emit(LoginFailure(failure: UnknownAppFailure(message: e.toString())));
  }
}
```

**為什麼錯**：UseCase 已經回 `Either`，再用 try-catch 重複處理；型別系統強制 fold 是 Either pattern 的價值，繞過去等於白用。

---

## Presentation：永不 catch

```dart
// ❌ Page 內 try-catch
@override
Widget build(BuildContext context) {
  return ElevatedButton(
    onPressed: () async {
      try {
        await getIt<LoginUseCase>()(input: ...);          // ❌
      } catch (e) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('$e')),
        );
      }
    },
    child: const Text('Submit'),
  );
}
```

**為什麼錯**：

1. Page 應該只 wire UI ↔ BLoC，業務 / 錯誤都透過 BLoC State / SideEffect 表達
2. 直接呼 UseCase 跳過 BLoC，狀態管理失控
3. 業務錯誤訊息散落 Page，無法測試

---

## 完整鏈範例

```dart
// 1. DataSource throw
@LazySingleton(as: AuthRemoteDataSource)
class AuthRemoteDataSourceImpl implements AuthRemoteDataSource {
  @override
  Future<LoginResponseDto> login({required LoginRequestDto request}) async {
    try {
      final response = await _dio.post<Map<String, dynamic>>(...);
      return LoginResponseDto.fromJson(response.data!);
    } on DioException catch (e) {
      throw _mapDioException(e);     // throws typed NetworkException
    }
  }
}

// 2. RepositoryImpl catch + map
@LazySingleton(as: AuthRepository)
class AuthRepositoryImpl implements AuthRepository {
  @override
  Future<Either<AppFailure, User>> login({...}) async {
    try {
      final dto = await _remoteDataSource.login(request: ...);
      return Right(UserMapper.fromResponseDto(dto));
    } on Object catch (error) {
      return Left(AppFailureMapper.toFailure(error));
    }
  }
}

// 3. UseCase 透傳
@injectable
class LoginUseCase {
  Future<Either<AppFailure, User>> call({required LoginInput input}) =>
      _repository.login(email: input.email, password: input.password);
}

// 4. BLoC fold
Future<void> _onLoginSubmitted(LoginSubmitted event, Emitter<LoginState> emit) async {
  emit(const LoginLoading());
  final result = await _loginUseCase(input: LoginInput(...));
  result.fold(
    (failure) {
      emit(LoginFailure(failure: failure));
      emitSideEffect(LoginShowError(message: failure.message));
    },
    (user) {
      emit(LoginSuccess(user: user));
      emitSideEffect(const LoginNavigateToHome());
    },
  );
}

// 5. Page 看 state
BlocBuilder<LoginBloc, LoginState>(
  builder: (context, state) => switch (state) {
    LoginLoading() => const Center(child: CircularProgressIndicator()),
    LoginFailure(:final failure) => _ErrorView(failure: failure),
    _ => const _LoginForm(),
  },
);
```

---

## Related

- [`architecture/data-layer.md`](../architecture/data-layer.md) — RepositoryImpl 的 catch + map 細節
- [`architecture/domain-layer.md`](../architecture/domain-layer.md) — Repository interface、UseCase 簽章
- [`state-management.md`](./state-management.md) — BLoC handler 的 fold pattern
- [ADR-007](../adr/007-either-pattern-for-domain-results.md)
