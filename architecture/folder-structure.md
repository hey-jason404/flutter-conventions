# 目錄結構

> **TL;DR**：feature-first 結構。`lib/{app, core, ui_kit, generated, features, main.dart}`，`features/` 下每個 feature 有 `data/ domain/ presentation/` 三層。Feature 一層深，禁止 nesting。`main.dart` 是入口、`app/app.dart` 是殼、`app/bootstrap/` 集中副作用（→ [ADR-013](../adr/013-app-entry-shell-bootstrap-layering.md)）。

---

## 不可妥協

| # | 規則 | ADR |
|---|---|---|
| 1 | `features/` 一層深，禁止 sub-feature nesting | — |
| 2 | 一個 feature 一個 RepositoryImpl、最多一個 remote / 一個 local DataSource | — |
| 3 | 每個 page 一個 sub-folder，page-level BLoC 命名以 page 為主 | — |
| 4 | `presentation/` 下不放共用 `bloc/` / `pages/` / `widgets/` 目錄 | — |
| 5 | `generated/` 下檔案絕不手改 | — |
| 6 | `main.dart` 嚴格三行：binding → bootstrap → `runApp` | [ADR-013](../adr/013-app-entry-shell-bootstrap-layering.md) |
| 7 | `app.dart` 是 `const App({super.key})` 純殼，不收業務參數 | [ADR-013](../adr/013-app-entry-shell-bootstrap-layering.md) |
| 8 | 副作用集中在 `app/bootstrap/`，不散落 `main.dart` 或 widget 樹 | [ADR-013](../adr/013-app-entry-shell-bootstrap-layering.md) |

---

## 完整 `lib/` 結構

```text
lib/
├── app/
│   ├── app.dart                            # const App() — App 殼（不收業務參數）
│   ├── bootstrap/                          # 啟動副作用集中地（→ ADR-013）
│   │   ├── app_bootstrap.dart              # 對外唯一入口：bootstrap()
│   │   └── {sdk}_bootstrap.dart            # 各 SDK / 副作用一檔
│   ├── di/
│   │   ├── injection_container.dart        # GetIt instance + @InjectableInit
│   │   └── injection_container.config.dart # generated, do not edit
│   └── {shell_name}/                       # 視需要：跨 feature UI shell（→ 下節「App 殼 sub-folders」）
│
├── core/                               # 跨 feature 共用基礎設施
│   ├── config/                         # AppConfig SSOT
│   │   └── app_config.dart
│   ├── env/                            # build-time env，僅 app_config.dart 可 import
│   │   ├── env_keys.dart
│   │   ├── env_provider.dart
│   │   └── env_provider_impl.dart
│   ├── error/
│   │   ├── app_failure.dart            # AppFailure sealed class
│   │   ├── app_failure_mapper.dart     # Exception → AppFailure 映射
│   │   ├── field_failure.dart          # FieldFailure sealed class
│   │   └── network_exceptions.dart     # NetworkException sealed class
│   ├── network/
│   ├── platform/
│   │   └── {capability}/
│   │       ├── {capability}_platform.dart
│   │       └── {capability}_platform_impl.dart
│   ├── session/
│   │   ├── session.dart
│   │   ├── session_context.dart
│   │   ├── session_manager.dart
│   │   └── di/
│   │       └── session_di.dart
│   └── utils/                          # 純 Dart 工具
│
├── ui_kit/                             # 全域共用 UI 組件，無業務邏輯
│   └── {component_name}/
│       └── {component_name}_widget.dart
│
├── generated/                          # 自動生成檔案，絕不手改
│   ├── flutter_gen/
│   └── localization/
│
├── features/
│   └── {feature_name}/
│       ├── data/
│       │   ├── datasources/
│       │   │   ├── remote/
│       │   │   │   ├── {feature}_remote_datasource.dart
│       │   │   │   ├── {feature}_remote_datasource_impl.dart
│       │   │   │   └── {feature}_api_paths.dart
│       │   │   └── local/
│       │   │       ├── {feature}_local_datasource.dart
│       │   │       └── {feature}_local_datasource_impl.dart
│       │   ├── models/
│       │   │   ├── {action}_response_dto/
│       │   │   │   ├── {action}_response_dto.dart
│       │   │   │   └── {action}_response_dto.g.dart      # generated
│       │   │   ├── {action}_request_dto/
│       │   │   └── {entity}_local_dto/
│       │   ├── mappers/
│       │   │   └── {entity}_mapper.dart
│       │   └── repositories/
│       │       └── {feature}_repository_impl.dart
│       │
│       ├── domain/
│       │   ├── entities/
│       │   │   └── {entity}/
│       │   │       ├── {entity}.dart
│       │   │       └── {entity}.freezed.dart            # generated
│       │   ├── enums/
│       │   ├── repositories/
│       │   │   └── {feature}_repository.dart           # abstract interface
│       │   ├── usecases/
│       │   │   └── {verb_noun}/
│       │   │       ├── {verb_noun}_usecase.dart
│       │   │       └── {verb_noun}_input.dart
│       │   └── value_objects/
│       │       └── {value_object_name}/
│       │           ├── {value_object_name}.dart
│       │           └── {value_object_name}.freezed.dart
│       │
│       └── presentation/
│           └── {page_name}/
│               ├── {page_name}_page.dart
│               ├── bloc/
│               │   ├── {page_name}_bloc.dart
│               │   ├── {page_name}_event.dart
│               │   └── {page_name}_state.dart
│               └── widgets/
│                   └── {widget_name}_widget.dart
│
└── main.dart
```

---

## 程式入口與 App 殼

> 詳細決策見 [ADR-013: App Entry / Shell / Bootstrap Layering](../adr/013-app-entry-shell-bootstrap-layering.md)。

App 啟動拆成三層，責任彼此不重疊：

| 層 | 責任 | 產出 |
|---|---|---|
| **入口（Entry）** | 拉起 Flutter runtime → 交棒 bootstrap → `runApp` | `lib/main.dart` |
| **啟動（Bootstrap）** | 所有「widget 樹掛起來之前」必須做完的副作用 | `lib/app/bootstrap/` |
| **殼（Shell）** | App widget 樹根：Scopes、全域 `BlocProvider`、`MaterialApp`、Theme、Router 接線 | `lib/app/app.dart` |

### `main.dart` 三行守則

```dart
// ✅ main.dart 允許的全部內容
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await bootstrap();
  runApp(const App());
}
```

```dart
// ❌ 不准在 main.dart 出現
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  FirebaseMessaging.onBackgroundMessage(handler);   // 屬 bootstrap
  await configureDependencies();
  Bloc.observer = getIt<MyObserver>();              // 屬 bootstrap
  final router = getIt<GoRouter>();                 // 不准 getIt
  runApp(App(router: router));                      // App 不准收業務參數
}
```

### `app/app.dart` 殼職責

- `class App extends StatelessWidget` + `const App({super.key})`
- **不收業務參數**（router / observer / key 都不從建構式進來）
- 內部子 widget 可以 `getIt<...>()` 取依賴
- 只描述 widget 樹結構：Scopes → 全域 `BlocProvider` → `MaterialApp.router`
- **不做副作用**（不 `start()`、不註冊 callback、不賦值 observer）

```dart
// ✅ const 殼，從 DI 取依賴
class App extends StatelessWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context) {
    return _AppScopes(
      child: _AppBlocProviders(
        child: _AppMaterialApp(),
      ),
    );
  }
}
```

```dart
// ❌ 收業務參數
class App extends StatelessWidget {
  const App({required this.router, super.key});
  final GoRouter router;
  ...
}
```

### `app/bootstrap/` 集中副作用

凡是「對 SDK / 全域單例做設定」的動作，全部進 `app/bootstrap/`，由單一入口 `Future<void> bootstrap()` 對外暴露。包括：DI 初始化、SDK callback 註冊、`Bloc.observer` 賦值、global error handler、長駐 service `start()` 等。

> 詳細的可做 / 禁止 / 灰色地帶清單與啟動順序，見 [`infrastructure/bootstrap.md`](../infrastructure/bootstrap.md)。

### `getIt` 邊界（本層級內）

| 位置 | 允許？ | 說明 |
|---|---|---|
| `main.dart` | ❌ 禁用 | 入口三行守則已隱含；明文禁止避免日後鬆動 |
| `app/bootstrap/` | ✅ 允許 | 副作用 setup 需要從容器取出 service / observer 實例 |
| `app/app.dart` | ✅ 允許 | 殼層子 widget 需要接線全域依賴 |

> 📌 `features/` / `core/` / 其他層的 `getIt` 規則不在本章節範圍。

### 判斷某段 code 屬於哪一層

- 不需要 `BuildContext`？→ 屬 bootstrap，不准放 widget。
- SDK 全域 callback 註冊 / observer 賦值？→ 屬 bootstrap，不准放 main。
- 描述 widget 樹結構？→ 屬 shell。

---

## App 殼 sub-folders（跨 feature UI 殼）

`app.dart` 本身只是 const 殼（[ADR-013](../adr/013-app-entry-shell-bootstrap-layering.md)）：`MaterialApp` / Scopes / 全域 `BlocProvider` 結構而已。中大型 App 常需要更具體的「跨 feature 殼」承載：

- **Tab navigation shell**：bottom nav + AppBar slot + body，依 session 過濾可見 tab、登入態切換預設 tab
- **App-level overlay shell**：global loading / snackbar / failure dialog catalog / 路由切換（session 過期、無網路、維護中）/ auto-logout 倒數
- **特定 navigation pattern shell**：side drawer + content、master-detail 等

這類 shell **業務上有自己的 page-level state**，命名上自成 capability，既不屬於某一個 feature（會造成 features 互相 import shell，違反 [ADR-014](../adr/014-features-cross-import-rules.md)），也不該整包塞進 `app.dart`（殼會膨脹得不可讀）。

### 解法：在 `app/` 下開 shell folder

```text
app/
├── app.dart                          # const 殼（不變，→ ADR-013）
├── bootstrap/                        # 副作用（不變）
├── di/                               # DI（不變）
└── {shell_name}/                     # 跨 feature shell
    ├── {shell_name}.dart             # shell widget 入口
    ├── bloc/                         # 視需要（有狀態才建）
    │   ├── {shell_name}_bloc.dart
    │   ├── {shell_name}_event.dart
    │   ├── {shell_name}_state.dart
    │   └── {shell_name}_side_effect.dart
    └── widgets/                      # 視需要（拆出子 widget 才建）
        └── {widget_name}_widget.dart
```

結構與下方 [Page 結構](#page-結構) 對齊（page-folder convention）。視業務需要可加更多 sub-folder（例：`failure/` 失敗 routing/catalog、`dispatcher/` 對 features 暴露的 facade interface），但保持 page-folder 主軸。

### 與 `app.dart` 的關係

- `app.dart` 仍**不收業務參數**，仍只描述 widget 樹結構（ADR-013 不變）
- shell widget 由 `app.dart` 內部組裝（透過 `MaterialApp.builder` 包 router child，或 router shell route）
- shell 自己的 BLoC 由 shell 上方的 `BlocProvider` 提供，shell 內部 widget 透過 `context.read` 取用

### Shell vs Feature page 區分

| 維度 | Feature page | Shell |
|---|---|---|
| 屬於 | 單一 feature 的業務功能 | 跨 feature 的 UI 結構 / 全域行為 |
| 位置 | `features/{name}/presentation/{page}/` | `app/{shell_name}/` |
| BLoC 命名 | `{page}Bloc` | `{shell}Bloc` |
| Routing | router 表項 | `MaterialApp.builder` 或 router shell route |
| 對 features 對外 API | 無（內部使用） | 視需要開 facade interface 給 features |

### Cross-import 規則

跨 feature import 規則（[ADR-014](../adr/014-features-cross-import-rules.md)）對 shell **同樣適用**：features 不可 import shell 的 BLoC / state / event / sideEffect 本體。若需要觸發 shell 行為（例如要 shell 顯示 global loading、推 failure dialog），shell 自己對外暴露 facade interface（例：`{Shell}Dispatcher`），features 依賴該 interface 而非 shell BLoC 本體。

### 何時開 vs 何時不開

- **開**：跨多個 feature 共用一塊 UI / 行為，且自身有 page-level state（tab 過濾、global loading token 計數、auto-logout 計時等）
- **不開**：純 widget tree 結構描述（仍屬 `app.dart`）；單一 feature 內部使用（仍屬 `features/{name}/presentation/`）；無業務的 framework wrapper（屬 `core/{capability}/`）

```dart
// ✅ 適合 shell
class TabShellBloc extends Bloc<TabShellEvent, TabShellState> { ... }   // app/tab_shell/bloc/
```

```dart
// ❌ 不適合 shell
class LoginBloc extends Bloc<LoginEvent, LoginState> { ... }            // 屬 features/login/
mixin LifecycleObserver { ... }                                         // 屬 core/{capability}/（無業務）
```

---

## Feature 模板

### 單頁 feature

```text
features/login/
├── data/
│   ├── datasources/remote/
│   │   ├── login_remote_datasource.dart
│   │   ├── login_remote_datasource_impl.dart
│   │   └── login_api_paths.dart
│   ├── models/
│   │   └── login_response_dto/
│   ├── mappers/
│   │   └── user_mapper.dart
│   └── repositories/
│       └── login_repository_impl.dart
├── domain/
│   ├── entities/user/
│   ├── repositories/login_repository.dart
│   └── usecases/
│       └── login/
│           ├── login_usecase.dart
│           └── login_input.dart
└── presentation/
    └── login_page/
        ├── login_page.dart
        ├── bloc/
        │   ├── login_bloc.dart
        │   ├── login_event.dart
        │   └── login_state.dart
        └── widgets/
            └── login_form_widget.dart
```

### 多頁 feature

```text
features/profile/
├── data/...
├── domain/...
└── presentation/
    ├── profile_overview_page/
    │   ├── profile_overview_page.dart
    │   ├── bloc/
    │   └── widgets/
    └── profile_edit_page/
        ├── profile_edit_page.dart
        ├── bloc/
        └── widgets/
```

每頁有自己的 `{page}_bloc.dart` 跟 `widgets/`。多頁共用的 widget 放在主頁的 `widgets/`。

---

## Page 結構

每個 page 有自己的目錄，命名 `{page_name}/`：

```text
presentation/login_page/
├── login_page.dart                # ✅ 必有
├── bloc/                          # 視需要建（有狀態的 page 必有）
│   ├── login_bloc.dart
│   ├── login_event.dart
│   └── login_state.dart
└── widgets/                       # 視需要建（拆出子 widget 才建）
    └── login_form_widget.dart
```

### 規則

- `{page_name}_page.dart`：page 入口 widget，必有
- `bloc/`：有狀態才建（純展示 page 可省）
- `widgets/`：page 內部的子 widget；跨 page 共用的 widget 放主 page 的 `widgets/`
- BLoC 命名以 **page 為主**：`LoginBloc`，**不是** `AuthBloc`

```dart
// ✅ page-level BLoC
class LoginBloc extends Bloc<LoginEvent, LoginState> { ... }
class ProfileEditBloc extends Bloc<ProfileEditEvent, ProfileEditState> { ... }

// ❌ feature-level BLoC（多 page 共用同一 BLoC）
class AuthBloc extends Bloc<AuthEvent, AuthState> { ... }
```

**為什麼錯**：feature-level BLoC 一旦多頁共用，state 變大，每頁只用其中一部分；rebuild 範圍失控；測試時要 mock 整個 feature 的所有事件。

---

## `core/` 該放什麼

跨 feature 共用的**基礎設施**（不是業務）：

- 全域 error 型別（`AppFailure`、`FieldFailure`、`NetworkException`）
- Network client（Dio instance、interceptor）
- Session（`SessionContext` 介面、`SessionManager` 實作）
- AppConfig（全域設定）
- Platform abstraction（`BiometricCapability`、`PushNotificationService` 等抽象介面）
- 純 Dart utility

**不放**：

- Feature 業務邏輯（屬於 `features/{name}/`）
- UI 共用組件（屬於 `ui_kit/`）

---

## `generated/` 規則

- **絕不手改任何檔案**
- 各 codegen tool 的 output 集中於 `lib/generated/`：
  - `flutter_gen/`：asset、color、font 的 typed access
  - `localization/`：slang 產生的 i18n class
- `*.freezed.dart`、`*.g.dart`、`*.config.dart` 直接放在 source 旁邊
- 是否 commit codegen 產物：**團隊自定**（規範不強制），但全 repo 一致

---

## ❌ 老式扁平結構

```text
// ❌ presentation 下層共用 bloc/ pages/ widgets/
features/auth/presentation/
├── bloc/
│   ├── login_bloc.dart
│   └── register_bloc.dart
├── pages/
│   ├── login_page.dart
│   └── register_page.dart
└── widgets/
    └── login_form.dart
```

**為什麼錯**：

- BLoC 跟 Page、Widget 分散三個資料夾，找一個 page 相關 code 要跨目錄
- BLoC 要起 feature 等級的名（`AuthBloc`），但實際只服務某一頁，命名跟責任脫鉤
- 多 page 加進來後 `bloc/` 變成大雜燴

---

## Related

- [`overview.md`](./overview.md) — 三層架構、依賴方向
- [`presentation-layer.md`](./presentation-layer.md) — Page / BLoC / Widget 結構
- [`data-layer.md`](./data-layer.md) — DataSource / Repository / Mapper 結構
- [`domain-layer.md`](./domain-layer.md) — Entity / UseCase / Repository 結構
- [ADR-013: App Entry / Shell / Bootstrap Layering](../adr/013-app-entry-shell-bootstrap-layering.md) — 程式入口與 App 殼三層分離決策
