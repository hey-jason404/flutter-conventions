# Network

> **TL;DR**：用 `dio` 為唯一 HTTP client；DataSource 是唯一 IO 邊界；API path 集中於 `<feature>_api_paths.dart`；DataSource 拋 typed exception，由 `AppFailureMapper` 轉成 `AppFailure`。

---

## 不可妥協

| # | 規則 |
|---|---|
| 1 | `dio` 為唯一 HTTP client，禁直接用 `http` package、`HttpClient`、其他 |
| 2 | 業務 code 內**禁止** `Dio()` 實體化 —— 一律 DI 注入 |
| 3 | API path 集中於 `{feature}_api_paths.dart`，禁字面值散落 method |
| 4 | DataSource 不回 `Either`，拋 typed exception |
| 5 | Base URL / headers / timeout 從 `AppConfig` 來，不 hardcoded |

---

## Dio 配置

### Base 設定

統一 Dio instance（DI 注入），base URL / timeout / common header 在建立時設好：

```dart
@module
abstract class NetworkModule {
  @lazySingleton
  Dio provideDio(AppConfig config) {
    return Dio(BaseOptions(
      baseUrl: config.apiBaseUrl,
      connectTimeout: const Duration(seconds: 10),
      receiveTimeout: const Duration(seconds: 30),
      sendTimeout: const Duration(seconds: 30),
      headers: {
        'Content-Type': 'application/json',
      },
    ))
      ..interceptors.add(LogInterceptor(requestBody: true, responseBody: true));
  }
}
```

### 多 base URL（多 endpoint）

如果 app 對接多個後端 server（譬如 `auth-server` + `payment-server`），用 `@Named` qualifier 區分多個 Dio instance：

```dart
abstract final class NetworkQualifiers {
  static const String authenticated = 'authenticated';
  static const String public = 'public';
  static const String payment = 'payment';
}

@module
abstract class NetworkModule {
  @Named(NetworkQualifiers.authenticated)
  @lazySingleton
  Dio provideAuthenticatedDio(AppConfig config) {
    return Dio(BaseOptions(baseUrl: config.apiBaseUrl))
      ..interceptors.add(AuthInterceptor());
  }

  @Named(NetworkQualifiers.public)
  @lazySingleton
  Dio providePublicDio(AppConfig config) {
    return Dio(BaseOptions(baseUrl: config.apiBaseUrl));
  }

  @Named(NetworkQualifiers.payment)
  @lazySingleton
  Dio providePaymentDio(AppConfig config) {
    return Dio(BaseOptions(baseUrl: config.paymentApiUrl))
      ..interceptors.add(PaymentSignatureInterceptor());
  }
}
```

DataSource 注入指定 qualifier：

```dart
@LazySingleton(as: AuthRemoteDataSource)
class AuthRemoteDataSourceImpl implements AuthRemoteDataSource {
  const AuthRemoteDataSourceImpl({
    @Named(NetworkQualifiers.public) required Dio dio,
  }) : _dio = dio;
  // ...
}
```

詳細 DI 規則：[`architecture/dependency-injection.md`](../architecture/dependency-injection.md)。

---

## DataSource 寫法

### Interface

```dart
abstract interface class AuthRemoteDataSource {
  Future<LoginResponseDto> login({required LoginRequestDto request});
  Future<GetUserProfileResponseDto> getProfile();
}
```

### Implementation

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
      throw _mapDioException(e);                    // throws typed exception
    }
  }

  @override
  Future<GetUserProfileResponseDto> getProfile() async {
    try {
      final response = await _dio.get<Map<String, dynamic>>(
        AuthApiPaths.getProfile,
      );
      return GetUserProfileResponseDto.fromJson(response.data!);
    } on DioException catch (e) {
      throw _mapDioException(e);
    }
  }

  Exception _mapDioException(DioException e) {
    if (e.type == DioExceptionType.connectionError) return const NoNetworkException();
    if (e.type == DioExceptionType.receiveTimeout ||
        e.type == DioExceptionType.sendTimeout) return const TimeoutException();
    final code = e.response?.statusCode ?? 500;
    if (code == 401) return const UnauthorizedException();
    if (code == 404) return const NotFoundException();
    return ServerException(statusCode: code);
  }
}
```

### Query parameters（GET 含參數）

```dart
@override
Future<GetProductListResponseDto> getProducts({
  required GetProductListRequestDto request,
}) async {
  try {
    final response = await _dio.get<Map<String, dynamic>>(
      ProductApiPaths.list,
      queryParameters: request.toJson(),
    );
    return GetProductListResponseDto.fromJson(response.data!);
  } on DioException catch (e) {
    throw _mapDioException(e);
  }
}
```

---

## API path 集中

每個 feature 一個 path 集中檔：`{feature}_api_paths.dart`

```dart
// data/datasources/remote/auth_api_paths.dart
abstract final class AuthApiPaths {
  static const String login = '/auth/login';
  static const String logout = '/auth/logout';
  static const String getProfile = '/auth/profile';
  static const String refreshToken = '/auth/refresh';
}
```

```dart
// ❌ hardcoded path 字面值在 DataSource method
@override
Future<LoginResponseDto> login({required LoginRequestDto request}) async {
  final response = await _dio.post<Map<String, dynamic>>(
    '/auth/login',                           // ❌
    data: request.toJson(),
  );
  return LoginResponseDto.fromJson(response.data!);
}
```

**為什麼錯**：rename 路徑時 grep 不全；無法集中審視一個 feature 的所有 endpoint。

### 動態 path（含 path parameter）

```dart
abstract final class ProductApiPaths {
  static const String list = '/products';
  static String detail(String id) => '/products/$id';
  static String reviews(String productId) => '/products/$productId/reviews';
}

// usage
final response = await _dio.get(ProductApiPaths.detail(productId));
```

---

## HTTP exception → AppFailure 映射

DataSource throw typed exception，在 RepositoryImpl catch 後透過 `AppFailureMapper` 轉成 `AppFailure`。詳見 [`patterns/error-handling.md`](../patterns/error-handling.md)。

```dart
// 1. Network exception types（in core/error/）
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

// 2. AppFailureMapper 集中映射
abstract final class AppFailureMapper {
  static AppFailure toFailure(Object error) {
    if (error is NetworkException) {
      return switch (error) {
        NoNetworkException() => const NetworkAppFailure(),
        TimeoutException() => const NetworkAppFailure(message: 'Request timeout'),
        UnauthorizedException() => const UnauthorizedAppFailure(),
        ServerException(:final statusCode) => ServerAppFailure(statusCode: statusCode),
        NotFoundException() => const NotFoundAppFailure(),
      };
    }
    return UnknownAppFailure(message: error.toString());
  }
}
```

---

## Interceptor

### Auth Interceptor（自動加 token）

```dart
class AuthInterceptor extends Interceptor {
  AuthInterceptor({required SessionContext sessionContext})
      : _sessionContext = sessionContext;

  final SessionContext _sessionContext;

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    final token = _sessionContext.accessToken;
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    handler.next(options);
  }
}
```

### Logging（dev only）

```dart
@module
abstract class NetworkModule {
  @lazySingleton
  Dio provideDio(AppConfig config) {
    final dio = Dio(BaseOptions(...));
    if (config.isDebug) {
      dio.interceptors.add(LogInterceptor(requestBody: true, responseBody: true));
    }
    return dio;
  }
}
```

> ⚠️ **禁** production log 打 request / response body（可能含敏感資料）。Logging 必依 `AppConfig.isDebug` 開關。

---

## 常見錯誤

```dart
// ❌ 業務 code 直接 Dio() 實體化
class LoginUseCase {
  Future<User> call(...) async {
    final dio = Dio();                             // ❌
    final response = await dio.post('/auth/login');
    return User.fromJson(response.data);
  }
}
```
**為什麼錯**：跳過 DataSource 邊界；UseCase 不該知道 HTTP；新 instance 缺 interceptor、timeout、base URL 設定。

```dart
// ❌ DataSource 包 Either
abstract interface class AuthRemoteDataSource {
  Future<Either<AppFailure, LoginResponseDto>> login({...});       // ❌
}
```
**為什麼錯**：`Either` 是 domain 層概念；DataSource 只負責 IO，錯誤模型轉換是 RepositoryImpl 的事。

```dart
// ❌ hardcoded URL / port
final dio = Dio(BaseOptions(
  baseUrl: 'https://api.example.com',                              // ❌
  connectTimeout: const Duration(seconds: 30),                     // ❌ magic number
));
```
**為什麼錯**：不同環境（dev / staging / prod）會有不同 URL；timeout 寫死沒彈性；應該從 `AppConfig` 注入。

---

## Related

- [`architecture/data-layer.md`](../architecture/data-layer.md) — DataSource、Repository 完整規範
- [`architecture/dependency-injection.md`](../architecture/dependency-injection.md) — Dio DI 註冊
- [`patterns/error-handling.md`](../patterns/error-handling.md) — exception → AppFailure 映射
- [`infrastructure/config.md`](./config.md) — AppConfig 注入 base URL / timeout
- [`infrastructure/session.md`](./session.md) — Session token interceptor
- [`packages.md`](../packages.md) — `dio` 套件白名單
