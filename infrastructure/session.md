# Session

> **TL;DR**：`SessionContext`（read-only 介面）給 domain / data 層用；`SessionManager`（可寫）只給 auth flow 用；`Session` 只存 `accessToken` 跟 `profile`。

---

## 不可妥協

| # | 規則 |
|---|---|
| 1 | `Session` 只存 `accessToken` + `profile` —— 禁塞 app config、feature flag、UI state |
| 2 | Domain / Data 層注入 `SessionContext`（read-only），不注入 `SessionManager` |
| 3 | `SessionManager` 三個寫入方法：`startSession`、`updateProfile`、`endSession` |
| 4 | Session token 存記憶體；持久化用 `flutter_secure_storage` |
| 5 | UseCase 不自行管理 token refresh / 登出邏輯，走 `SessionManager` |

---

## 兩個介面

| | `SessionContext` | `SessionManager` |
|---|---|---|
| 用途 | 讀取目前 session 狀態 | 修改 session（登入 / 登出 / 更新 profile） |
| 注入對象 | Domain / Data 層任何需要的地方 | 只給 auth feature 的 UseCase |
| 介面 | read-only（getter + stream） | 三個寫入方法 |
| 範例 | `_sessionContext.accessToken` | `_sessionManager.startSession(...)` |

---

## `Session` 結構

```dart
@freezed
abstract class Session with _$Session {
  const factory Session({
    required String accessToken,
    required UserProfile profile,
  }) = _Session;
}

@freezed
abstract class UserProfile with _$UserProfile {
  const factory UserProfile({
    required String userId,
    required String email,
    required String displayName,
  }) = _UserProfile;
}
```

### 規則

- 只存 `accessToken`、`profile`
- **禁塞**：app config、feature flag、UI state、navigation flag、cached data

```dart
// ❌ Session 塞了不該塞的
@freezed
abstract class Session with _$Session {
  const factory Session({
    required String accessToken,
    required UserProfile profile,
    required String apiBaseUrl,                  // ❌ 屬於 AppConfig
    required Map<String, bool> featureFlags,     // ❌ 屬於 AppConfig
    required bool isOnboardingComplete,          // ❌ UI / flow state
  }) = _Session;
}
```

**為什麼錯**：

- `apiBaseUrl` 是設定不是 session
- `featureFlags` 跨 session 存在
- `isOnboardingComplete` 是 flow state，不該綁登入

---

## `SessionContext`：read-only 介面

```dart
abstract interface class SessionContext {
  // 同步讀取
  bool get isAuthenticated;
  String? get accessToken;
  UserProfile? get currentProfile;
  String? get currentUserId;

  // 訂閱（reactive）
  Stream<Session?> watchSession();
}
```

### 注入到 Domain / Data 層

```dart
// ✅ UseCase 注入 SessionContext（read-only）
@injectable
class GetMemberDataUseCase {
  const GetMemberDataUseCase(this._sessionContext, this._repository);
  final SessionContext _sessionContext;
  final MemberRepository _repository;

  Future<Either<AppFailure, MemberData>> call() async {
    if (!_sessionContext.isAuthenticated) {
      return const Left(UnauthorizedAppFailure());
    }
    return _repository.getMemberData();
  }
}

// ✅ DataSource 注入 SessionContext 用於 token interceptor
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

### 規則

- 只暴露 read-only API（getter、stream）
- **禁** 直接從 `SessionManager` cast 取得 `SessionContext`

---

## `SessionManager`：可寫介面

只給 auth feature 用：

```dart
@LazySingleton(as: SessionContext)
@LazySingleton()
class SessionManager implements SessionContext {
  Session? _session;
  final StreamController<Session?> _controller =
      StreamController<Session?>.broadcast();

  @override
  bool get isAuthenticated => _session != null;

  @override
  String? get accessToken => _session?.accessToken;

  @override
  UserProfile? get currentProfile => _session?.profile;

  @override
  String? get currentUserId => _session?.profile.userId;

  @override
  Stream<Session?> watchSession() => _controller.stream;

  // 三個寫入方法
  void startSession({required Session session}) {
    _session = session;
    _controller.add(session);
  }

  void updateProfile({required UserProfile profile}) {
    final current = _session;
    if (current == null) return;
    _session = current.copyWith(profile: profile);
    _controller.add(_session);
  }

  void endSession() {
    _session = null;
    _controller.add(null);
  }

  @disposeMethod
  Future<void> dispose() => _controller.close();
}
```

> 注意：`SessionManager` 同時 register as `SessionContext` 跟自己 —— Domain / Data 注入 `SessionContext`，auth feature 注入 `SessionManager`。

### 三個寫入方法

| 方法 | 何時呼叫 |
|---|---|
| `startSession` | 登入成功後（auth UseCase） |
| `updateProfile` | profile 更新（profile UseCase） |
| `endSession` | 登出 / token 過期 / 強制登出 |

```dart
// ✅ auth UseCase 用 SessionManager
@injectable
class LoginUseCase {
  const LoginUseCase(this._repository, this._sessionManager);
  final AuthRepository _repository;
  final SessionManager _sessionManager;

  Future<Either<AppFailure, User>> call({required LoginInput input}) async {
    final result = await _repository.login(
      email: input.email,
      password: input.password,
    );
    return result.map((session) {
      _sessionManager.startSession(session: session);
      return session.profile.toUser();
    });
  }
}
```

### ❌ 一般 feature 注入 `SessionManager`

```dart
@injectable
class GetProductListUseCase {
  const GetProductListUseCase(this._sessionManager, this._repository);     // ❌
  final SessionManager _sessionManager;
  final ProductRepository _repository;
}
```

**為什麼錯**：product feature 不該有改 session 的能力；應注入 `SessionContext` 唯讀使用。

---

## Session 生命週期

```text
App 啟動
    │
    ▼
（檢查持久化的 token）
    │
    ├─── 無 token ─►  unauthenticated state
    │
    └─── 有 token ─►  call API 驗證 → 取 profile
                          │
                          ▼
                    SessionManager.startSession(session: ...)
                          │
                          ▼
                    authenticated state
                          │
        ┌─────────────────┼──────────────────┐
        ▼                 ▼                  ▼
   profile 更新      token 過期          使用者登出
   updateProfile     endSession           endSession
```

---

## 持久化

Session 預設**只存記憶體**。需要持久化（譬如 app restart 維持登入）才寫盤：

```dart
// ✅ token 用 flutter_secure_storage
class SessionStorage {
  SessionStorage({required FlutterSecureStorage storage}) : _storage = storage;
  final FlutterSecureStorage _storage;

  static const _tokenKey = 'session_access_token';

  Future<String?> readToken() => _storage.read(key: _tokenKey);
  Future<void> writeToken(String token) => _storage.write(key: _tokenKey, value: token);
  Future<void> clearToken() => _storage.delete(key: _tokenKey);
}
```

```dart
// ❌ token 用 SharedPreferences（明文）
final prefs = await SharedPreferences.getInstance();
prefs.setString('access_token', token);
```

**為什麼錯**：SharedPreferences 在 iOS / Android 是明文存儲；token 屬於敏感資料，必用加密儲存。

---

## Token refresh / 強制登出

### Token refresh

由 `AuthInterceptor` 偵測 401 → 嘗試 refresh → 重發原 request：

```dart
class AuthInterceptor extends Interceptor {
  AuthInterceptor({
    required SessionContext sessionContext,
    required SessionManager sessionManager,
    required AuthRepository authRepository,
  });

  @override
  Future<void> onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode == 401) {
      final refreshed = await _tryRefreshToken();
      if (refreshed) {
        return handler.resolve(await _retryRequest(err.requestOptions));
      }
      _sessionManager.endSession();
    }
    handler.next(err);
  }
  // ...
}
```

> 細節依專案需求；本檔僅描述「token refresh 應在 interceptor 層集中處理」。

### 強制登出

如果有「全域強制登出」需求（譬如 server 端踢人），由收到對應 signal 的元件呼叫 `_sessionManager.endSession()`，後續 router / app shell 監聽 `watchSession()` 處理導航。

---

## 常見錯誤

```dart
// ❌ Repository 注入 SessionManager
@LazySingleton(as: ProductRepository)
class ProductRepositoryImpl implements ProductRepository {
  const ProductRepositoryImpl({
    required ProductRemoteDataSource ds,
    required SessionManager sessionManager,                // ❌
  });
}
```
**為什麼錯**：product repository 不該有改 session 的能力；應注入 `SessionContext`。

```dart
// ❌ UseCase 自行 refresh token
@injectable
class GetMemberDataUseCase {
  Future<Either<AppFailure, MemberData>> call() async {
    final token = _sessionContext.accessToken;
    if (_isExpired(token)) {
      await _refreshToken();                               // ❌
    }
    // ...
  }
}
```
**為什麼錯**：token refresh 是 cross-cutting concern，應在 interceptor 層集中處理；散落 UseCase 各處會重複、不一致。

```dart
// ❌ Session 帶 cache data
@freezed
abstract class Session with _$Session {
  const factory Session({
    required String accessToken,
    required UserProfile profile,
    required List<Product> recentProducts,                 // ❌ cache 不屬於 session
  }) = _Session;
}
```
**為什麼錯**：cache 該由 Repository 處理，跟 session lifecycle 解耦。

---

## Related

- [`architecture/dependency-injection.md`](../architecture/dependency-injection.md) — `SessionManager` 雙重註冊（自己 + as `SessionContext`）
- [`infrastructure/network.md`](./network.md) — `AuthInterceptor` 用 `SessionContext` 取 token
- [`infrastructure/config.md`](./config.md) — Session vs AppConfig 分工
- [`packages.md`](../packages.md) — `flutter_secure_storage` 套件白名單
