# Platform

> **TL;DR**：Platform 差異透過抽象介面隔離。每個 capability 一個 `{Capability}Platform` interface + impl。BLoC → UseCase → PlatformService（不 BLoC / DataSource / Repository 直接注入 platform）。`PlatformException` 在 impl 內接住，轉成 typed result。

---

## 不可妥協

| # | 規則 |
|---|---|
| 1 | Platform 差異抽象成 `{Capability}Platform` interface | 
| 2 | UseCase 注入 platform service；BLoC、DataSource、Repository **禁** 注入 |
| 3 | `PlatformException` 一律在 impl 內 catch 轉 typed result，**禁** 向上傳播 |
| 4 | Platform service 不回 `Either<AppFailure, T>`（domain 層才有 Either） |
| 5 | `kIsWeb` 用在最外層；`Platform.isIOS` / `isAndroid` 等不能在 web 環境跑 |
| 6 | 規範**不偏 iOS / Android / Web 任一平台**，範例平均涵蓋 |

---

## 架構

Platform service 是 infrastructure，不是 feature data source：

```text
BLoC → UseCase → PlatformService
```

Infrastructure 模組（如 network interceptor）也可以注入 platform service，但是**單向**：

```text
core/network/  ───►  core/platform/
core/platform/  ──►  ✗ 不依賴 core/network/
```

---

## 目錄結構

```text
core/
└── platform/
    └── {capability}/
        ├── {capability}_platform.dart       # abstract interface + result types
        └── {capability}_platform_impl.dart  # MethodChannel / EventChannel / 第三方 plugin
```

---

## Platform service 介面

`abstract class`，命名 `{Capability}Platform`（**不**加 `I` prefix）。Result enum / value type 跟介面同檔：

```dart
// core/platform/biometric_auth/biometric_auth_platform.dart

enum BiometricAuthResult { authenticated, cancelled, failed, unavailable }

abstract class BiometricAuthPlatform {
  Future<BiometricAuthResult> authenticate({required String localizedReason});
}
```

---

## MethodChannel 實作

```dart
// core/platform/device_info/device_info_platform.dart
abstract class DeviceInfoPlatform {
  Future<String> getDeviceId();
}

// core/platform/device_info/device_info_platform_impl.dart
@LazySingleton(as: DeviceInfoPlatform)
final class DeviceInfoPlatformImpl implements DeviceInfoPlatform {
  static const _channel = MethodChannel('com.example.app/device_info');

  @override
  Future<String> getDeviceId() async {
    final result = await _channel.invokeMethod<String>('getDeviceId');
    return result ?? '';
  }
}
```

---

## EventChannel 實作

`Stream<T>` 接 native 推來的事件：

```dart
// core/platform/notification/notification_platform.dart
abstract class NotificationPlatform {
  Future<void> initialize({required String userId});
  Stream<int> watchBadgeCount();
}

// core/platform/notification/notification_platform_impl.dart
@LazySingleton(as: NotificationPlatform)
final class NotificationPlatformImpl implements NotificationPlatform {
  static const _methodChannel = MethodChannel('com.example.app/notification');
  static const _badgeChannel = EventChannel('com.example.app/notification/badge');

  @override
  Future<void> initialize({required String userId}) {
    return _methodChannel.invokeMethod('initialize', {'userId': userId});
  }

  @override
  Stream<int> watchBadgeCount() {
    return _badgeChannel
        .receiveBroadcastStream()
        .map((event) => event as int);
  }
}
```

---

## 第三方 plugin 實作

Catch `PlatformException`，轉成 typed result：

```dart
// core/platform/biometric_auth/biometric_auth_platform_impl.dart
@LazySingleton(as: BiometricAuthPlatform)
final class BiometricAuthPlatformImpl implements BiometricAuthPlatform {
  BiometricAuthPlatformImpl() : _localAuth = LocalAuthentication();

  final LocalAuthentication _localAuth;

  @override
  Future<BiometricAuthResult> authenticate({required String localizedReason}) async {
    try {
      final result = await _localAuth.authenticate(
        localizedReason: localizedReason,
      );
      return result ? BiometricAuthResult.authenticated : BiometricAuthResult.failed;
    } on PlatformException catch (error) {
      return switch (error.code) {
        'UserCancelled' || 'SystemCancelled' => BiometricAuthResult.cancelled,
        _ => BiometricAuthResult.unavailable,
      };
    }
  }
}
```

> **絕不讓 `PlatformException` 漏出 impl**。`PlatformException` 是 Flutter platform channel 的細節，外部不該感知。

---

## UseCase 注入

```dart
// ✅ UseCase 注入 platform service
@injectable
class AuthenticateWithBiometricsUseCase {
  const AuthenticateWithBiometricsUseCase({required BiometricAuthPlatform platform})
      : _platform = platform;

  final BiometricAuthPlatform _platform;

  Future<BiometricAuthResult> call({required String localizedReason}) {
    return _platform.authenticate(localizedReason: localizedReason);
  }
}
```

如果 platform 取得的資料要傳給後端 API，由 UseCase 取得後組裝 Input 傳給 Repository：

```dart
@injectable
class LoginUseCase {
  const LoginUseCase({
    required AuthRepository repository,
    required DeviceInfoPlatform deviceInfoPlatform,
  })  : _repository = repository,
        _deviceInfoPlatform = deviceInfoPlatform;

  final AuthRepository _repository;
  final DeviceInfoPlatform _deviceInfoPlatform;

  Future<Either<AppFailure, User>> call({required LoginInput input}) async {
    final deviceId = await _deviceInfoPlatform.getDeviceId();
    return _repository.login(
      email: input.email,
      password: input.password,
      deviceId: deviceId,
    );
  }
}
```

### ❌ 注入到 DataSource / Repository / BLoC

```dart
@LazySingleton(as: AuthRemoteDataSource)
class AuthRemoteDataSourceImpl implements AuthRemoteDataSource {
  const AuthRemoteDataSourceImpl({
    required Dio dio,
    required DeviceInfoPlatform deviceInfo,         // ❌
  });
}
```

**為什麼錯**：DataSource 應該只負責 IO；混入 platform 細節後，DataSource 同時依賴 HTTP + native channel，testability / 職責糊掉。

---

## Network interceptor 用 platform

例外場景：interceptor 是 cross-cutting，可以注入 platform：

```dart
// core/network/interceptors/device_id_interceptor.dart
@LazySingleton()
final class DeviceIdInterceptor extends Interceptor {
  const DeviceIdInterceptor({required DeviceInfoPlatform platform})
      : _platform = platform;

  final DeviceInfoPlatform _platform;

  @override
  Future<void> onRequest(
    RequestOptions options,
    RequestInterceptorHandler handler,
  ) async {
    final deviceId = await _platform.getDeviceId();
    options.headers['X-Device-Id'] = deviceId;
    handler.next(options);
  }
}
```

依賴方向是單向 `core/network/ → core/platform/`，不可反向。

---

## Platform 判斷

### `kIsWeb` 在最外層

```dart
// ✅ kIsWeb 在最外
if (kIsWeb) {
  return _webImpl();
} else if (Platform.isIOS) {
  return _iosImpl();
} else if (Platform.isAndroid) {
  return _androidImpl();
}
```

```dart
// ❌ web 環境執行 Platform.isXxx 會炸
if (Platform.isIOS) { ... }                       // ❌ 在 web 是 unsupported
```

**為什麼錯**：`Platform.*`（來自 `dart:io`）在 web 是 unsupported，runtime 會 throw。`kIsWeb` 是 compile-time const，先檢查它是 safe pattern。

### Platform 中性立場

範例平均涵蓋 iOS / Android（不偏一方）：

```dart
// ✅ 平均覆蓋
return switch (defaultTargetPlatform) {
  TargetPlatform.iOS => const _IosWidget(),
  TargetPlatform.android => const _AndroidWidget(),
  _ => const _DefaultWidget(),
};
```

---

## 常見錯誤

```dart
// ❌ Page 散落 platform 判斷
class LoginPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    if (Platform.isIOS) {
      return _IosLoginForm();                      // ❌ 應抽 platform impl
    }
    return _AndroidLoginForm();
  }
}
```

**為什麼錯**：platform 條件 widget tree 散落各 page；應該抽到 platform abstraction，page 只看抽象介面。

```dart
// ❌ Platform service 回 Either
abstract class BiometricAuthPlatform {
  Future<Either<AppFailure, BiometricAuthResult>> authenticate({...});       // ❌
}
```

**為什麼錯**：`Either<AppFailure, T>` 是 domain 層概念；platform service 屬於 infrastructure，回 typed result enum 即可。

```dart
// ❌ PlatformException 漏出去
@LazySingleton(as: BiometricAuthPlatform)
class BiometricAuthPlatformImpl implements BiometricAuthPlatform {
  @override
  Future<BiometricAuthResult> authenticate({...}) async {
    final result = await _localAuth.authenticate(...);    // 沒 catch
    return result ? BiometricAuthResult.authenticated : BiometricAuthResult.failed;
  }
}
```

**為什麼錯**：`PlatformException` 漏到 UseCase 後，UseCase 又要 catch；應該在 impl 內接住轉 typed result。

---

## Related

- [`architecture/dependency-injection.md`](../architecture/dependency-injection.md) — Platform service 註冊
- [`architecture/domain-layer.md`](../architecture/domain-layer.md) — UseCase 注入 platform
- [`patterns/error-handling.md`](../patterns/error-handling.md) — `PlatformException` 不漏出 impl 的設計動機
