# Dependency Injection

> **TL;DR**：用 `get_it` + `injectable`。BLoC / UseCase = `@injectable`；Repository / DataSource = `@LazySingleton(as: Interface)`。第三方 class 透過 `@module` 註冊。

---

## 不可妥協

| # | 規則 | ADR |
|---|---|---|
| 1 | DI 唯一 entry：`lib/app/di/injection_container.dart` | — |
| 2 | First-party class 一律用 class-level annotation；`@module` 只用於第三方 | — |
| 3 | `@Named` qualifier 用 `abstract final class` 常數，禁 hard-coded string | — |
| 4 | Feature DI 放 `features/{feature}/di/`，禁出現在 `app/di/` | — |
| 5 | Annotation 變更後一律跑 `build_runner` | — |

---

## 套件對照

| Package | 角色 |
|---|---|
| `get_it` | Service locator runtime（`GetIt.instance`） |
| `injectable` | Annotation API（`@injectable`、`@LazySingleton`、`@module`...） |
| `injectable_generator` | Codegen runner，產出 `*.config.dart`（dev_dependency） |
| `build_runner` | Codegen orchestrator（dev_dependency） |

---

## 入口

唯一 entry：`lib/app/di/injection_container.dart`

```dart
// lib/app/di/injection_container.dart
import 'package:get_it/get_it.dart';
import 'package:injectable/injectable.dart';

import 'injection_container.config.dart';

final getIt = GetIt.instance;

@InjectableInit(preferRelativeImports: true)
Future<void> configureDependencies() async => getIt.init();
```

```dart
// lib/main.dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await configureDependencies();
  runApp(const App());
}
```

`injection_container.config.dart` 是 codegen 產物，**絕不手改**。

---

## Annotation 形式

**有參數用大寫 class form；無參數用小寫 shorthand**。

```dart
// ✅ 無參數 — 小寫 shorthand
@injectable
@lazySingleton

// ✅ 有參數 — 大寫 class form
@LazySingleton(as: AuthRepository)
@Singleton(dispose: _dispose)
@Named(NetworkQualifiers.public)

// ❌ 無參數但寫大寫
@Injectable()

// ❌ 有參數寫小寫（語法錯誤）
@lazySingleton(as: AuthRepository)
```

---

## 各層 annotation

| 層 | Annotation | 行為 |
|---|---|---|
| BLoC | `@injectable` | factory，每次注入新實例 |
| UseCase | `@injectable` | factory |
| Repository impl | `@LazySingleton(as: Interface)` | 單例，首次使用時建構 |
| DataSource impl | `@LazySingleton(as: Interface)` | 單例 |
| Mapper（`abstract final class` static-only） | — | 不註冊 DI |

```dart
// ✅ BLoC — @injectable（factory）
@injectable
class LoginBloc extends Bloc<LoginEvent, LoginState> {
  LoginBloc({required LoginUseCase loginUseCase})
      : _loginUseCase = loginUseCase,
        super(const LoginInitial());

  final LoginUseCase _loginUseCase;
}

// ✅ UseCase — @injectable（factory）
@injectable
class LoginUseCase {
  const LoginUseCase(this._repository);

  final AuthRepository _repository;

  Future<Either<AppFailure, User>> call({required LoginInput input}) {
    return _repository.login(email: input.email, password: input.password);
  }
}

// ✅ Repository impl — @LazySingleton bound to interface
@LazySingleton(as: AuthRepository)
class AuthRepositoryImpl implements AuthRepository {
  const AuthRepositoryImpl({required AuthRemoteDataSource remoteDataSource})
      : _remoteDataSource = remoteDataSource;

  final AuthRemoteDataSource _remoteDataSource;
}

// ✅ DataSource impl — @LazySingleton bound to interface
@LazySingleton(as: AuthRemoteDataSource)
class AuthRemoteDataSourceImpl implements AuthRemoteDataSource {
  const AuthRemoteDataSourceImpl({required Dio dio}) : _dio = dio;

  final Dio _dio;
}
```

---

## `@LazySingleton` vs `@Singleton`

預設 `@LazySingleton`（首次使用才建構）。`@Singleton` 只在以下場景用：

- 物件需要**啟動時跑 side effect**（如註冊全域 observer）
- 其他依賴在它構造之前 resolve、但需要它已存在

```dart
// ✅ 預設 — @LazySingleton
@LazySingleton(as: AuthRepository)
class AuthRepositoryImpl implements AuthRepository { ... }

// ✅ @Singleton — 啟動時建構（registers global observer）
@Singleton()
class CrashReportBlocObserver extends BlocObserver {
  @override
  void onError(BlocBase bloc, Object error, StackTrace stackTrace) {
    Logger.recordError(error, stackTrace);
    super.onError(bloc, error, stackTrace);
  }
}
```

---

## `@module`：第三方 class 註冊

`@module` **只用於無法 annotate 的第三方 class**（如 `Dio`、`SharedPreferences`）。
First-party 一律用 class-level annotation。

```dart
// ✅ 第三方 class — SharedPreferences 沒 source 可 annotate
@module
abstract class StorageModule {
  @preResolve
  Future<SharedPreferences> provideSharedPreferences() =>
      SharedPreferences.getInstance();
}

// ❌ @module 包 first-party class — 應直接 annotate class
@module
abstract class AuthModule {
  @lazySingleton
  AuthRepository provideAuthRepository(AuthRemoteDataSource ds) =>
      AuthRepositoryImpl(remoteDataSource: ds);
}
```

**為什麼錯**：first-party class 用 `@module` 是繞遠路；直接在 class 上 `@LazySingleton(as: AuthRepository)` 更短、更明確。

---

## `@Named` qualifier

同型別多註冊用 `@Named` 區分。**字串常數放 `abstract final class`，不 hard-code。**

```dart
// ✅ 常數集中
abstract final class NetworkQualifiers {
  static const String authenticated = 'authenticated';
  static const String public = 'public';
}

// ✅ module 用常數
@module
abstract class NetworkModule {
  @Named(NetworkQualifiers.authenticated)
  @lazySingleton
  Dio provideAuthenticatedDio() => Dio()..interceptors.add(AuthInterceptor());

  @Named(NetworkQualifiers.public)
  @lazySingleton
  Dio providePublicDio() => Dio();
}

// ✅ consumer 用 qualifier 注入
@LazySingleton(as: AuthRemoteDataSource)
class AuthRemoteDataSourceImpl implements AuthRemoteDataSource {
  const AuthRemoteDataSourceImpl({
    @Named(NetworkQualifiers.authenticated) required Dio dio,
  }) : _dio = dio;

  final Dio _dio;
}

// ❌ hard-coded qualifier string
@Named('authenticated')
Dio provideMemberDio() => Dio();
```

**為什麼錯**：字串散落各處，rename 時 grep 不到全部用處；typo 不會被編譯期捕捉。

---

## `@preResolve` 與 `@disposeMethod`

### `@preResolve`：async 初始化

`@module` 內如果初始化是 async（譬如 `SharedPreferences.getInstance()`），加 `@preResolve`，generator 會在 `configureDependencies()` 內 await。

```dart
@module
abstract class StorageModule {
  @preResolve
  Future<SharedPreferences> provideSharedPreferences() =>
      SharedPreferences.getInstance();
}
```

### `@disposeMethod`：清理資源

長期持有 stream / controller / connection 的 singleton，需要 `@disposeMethod` 標 cleanup。

```dart
@lazySingleton
class SessionManager {
  final StreamController<Session?> _controller =
      StreamController<Session?>.broadcast();

  Stream<Session?> get sessionStream => _controller.stream;

  @disposeMethod
  Future<void> dispose() async {
    await _controller.close();
  }
}
```

> ⚠️ 不要在 UI code 手動呼叫 `dispose()` 或 `getIt.reset()`。Lifecycle 由 DI container 管。

---

## DI 檔案位置

| 範圍 | 路徑 | Class 名 |
|---|---|---|
| Feature | `lib/features/{feature}/di/{feature}_di.dart` | `{Feature}Di` |
| Core 基礎設施 | `lib/core/{module}/di/{module}_di.dart` | `{Module}Di` |
| Core 多 provider | `lib/core/{module}/di/{module}_{provider}_di.dart` | `{Module}{Provider}Di` |
| App composition root | `lib/app/di/injection_container.dart` | —（entry） |

```text
// ✅ 結構
lib/
├── app/di/injection_container.dart
├── core/
│   ├── network/di/
│   │   ├── authenticated_network_di.dart      → AuthenticatedNetworkDi
│   │   └── public_network_di.dart             → PublicNetworkDi
│   └── storage/di/storage_di.dart             → StorageDi
└── features/
    └── auth/di/auth_di.dart                   → AuthDi
```

Feature DI **必須**在自己的 `di/` 內，不放在 `app/di/`。

---

## `build_runner`

Annotation 變更後一律跑：

```bash
dart run build_runner build --delete-conflicting-outputs
```

需要跑的場景：

- 新增 / 修改 / 刪除任何 `@injectable`、`@LazySingleton`、`@Singleton`、`@module`
- 改 `@injectable` class 的 constructor 簽章
- 新增 / 改名 / 刪除 `@Named` qualifier

→ 詳見 [`patterns/code-generation.md`](../patterns/code-generation.md)

---

## 禁止行為

| ❌ | 為什麼 |
|---|---|
| 在 generated `*.config.dart` 之外手寫 registration | 失去 single source of truth；新人讀不出哪些 class 註冊了 |
| `@module` 包 first-party class | 應直接 annotate class |
| 在 widget 用 `getIt<T>()` | 走 `BlocProvider` 或 widget constructor 注入 |
| Hard-coded qualifier string | rename 不安全、typo 編譯期不擋 |
| 該 `@LazySingleton` 卻寫 `@Singleton` | 失去 lazy 啟動，無理由的 startup cost |
| Feature 註冊放 `app/di/` | 違反 feature 自治原則 |
| UI 手動呼叫 `dispose()` 或 `getIt.reset()` | DI container 才是 lifecycle owner |
| 多個 `configureDependencies()` 入口 | 必須唯一 |

---

## Related

- [`overview.md`](./overview.md) — 三層架構與 DI 角色
- [`patterns/code-generation.md`](../patterns/code-generation.md) — codegen 流程
- [`packages.md`](../packages.md) — DI 套件白名單
