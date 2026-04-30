# App Bootstrap

> **TL;DR**：`app/bootstrap/` 是 app 啟動時所有副作用的集中地。對外只暴露 `Future<void> bootstrap()` 一個入口，由 `main.dart` 呼叫。內部依「外部 SDK / 副作用種類」分檔，每個子 bootstrap 是 top-level function，做完就走，不留回傳值。

> 詳細決策見 [ADR-013: App Entry / Shell / Bootstrap Layering](../adr/013-app-entry-shell-bootstrap-layering.md)。

---

## 不可妥協

| # | 規則 |
|---|---|
| 1 | `app/bootstrap/` 對外只暴露 `Future<void> bootstrap()` 一個入口 |
| 2 | 子 bootstrap 是 top-level function，**不註冊 DI、不在 widget 內呼叫** |
| 3 | 子 bootstrap **不回傳值**、**不下傳依賴**給 `App` 或 widget 樹 |
| 4 | `configureDependencies()` 必為 `bootstrap()` 內第一個動作 |
| 5 | `setupCrashReporting()` 必為第二個動作（要在其他副作用前才能捕捉它們的錯誤） |
| 6 | bootstrap 內**不觸發業務邏輯**（UseCase / Repository / BLoC event） |
| 7 | bootstrap 內**不做 navigation**、**不建構 widget** |

---

## ✅ 可以做的事

### 1. DI 容器初始化

```dart
await configureDependencies();
```

必為 `bootstrap()` 第一個動作，後續才能 `getIt<...>()`。

### 2. 全域 observer / handler 賦值

```dart
Bloc.observer = getIt<CrashReportBlocObserver>();
FlutterError.onError = (details) { ... };
PlatformDispatcher.instance.onError = (error, stack) { ... };
```

### 3. 外部 SDK 全域 callback / background handler 註冊

```dart
@pragma('vm:entry-point')
Future<void> _firebaseBackgroundHandler(RemoteMessage message) async { ... }

void setupFirebaseMessaging() {
  FirebaseMessaging.onBackgroundMessage(_firebaseBackgroundHandler);
}
```

> 📌 `@pragma('vm:entry-point')` 的 top-level handler **必須**宣告在 bootstrap 子檔。

### 4. 啟動「不依賴 BuildContext」的長駐 service

```dart
Future<void> setupPushNotification() async {
  getIt<PushTokenRefreshService>().start();
  await getIt<PushNotificationDisplayService>().start();
  await getIt<PushNotificationTapService>().start();
}
```

任何「app process 活著就該跑」的 background service 應在 bootstrap 啟動，**不該掛在 widget 的 `didChangeDependencies` / `initState`**。

### 5. 系統層 chrome 設定

```dart
await SystemChrome.setPreferredOrientations([DeviceOrientation.portraitUp]);
SystemChrome.setSystemUIOverlayStyle(SystemUiOverlayStyle.dark);
```

僅限**全域不變**的 chrome 設定。頁面層級的 chrome 由各 page 自行控制。

### 6. Logger / telemetry 開機

```dart
void setupLogger() {
  Logger.level = kDebugMode ? Level.debug : Level.info;
  Logger.addOutput(getIt<RemoteLogSink>());
}
```

---

## ❌ 禁止做的事

### 1. 任何 widget 樹相關操作

```dart
// ❌ runApp 只屬 main.dart
Future<void> bootstrap() async {
  ...
  runApp(const App());
}
```

```dart
// ❌ 建構 widget / Navigator / MaterialApp
final app = MaterialApp(...);
```

### 2. 業務邏輯 / domain 操作

```dart
// ❌ 呼叫 UseCase
await getIt<LoginUseCase>().call(input);

// ❌ 觸發 BLoC event
getIt<AppBloc>().add(const AppStarted());
```

> 📌 `AppBloc` 的初始 event 由 `BlocProvider.create` 觸發，不是 bootstrap。

### 3. 導航相關

```dart
// ❌ 建構 GoRouter（屬 app/router/ + DI）
final router = GoRouter(routes: [...]);

// ❌ 觸發 navigation
context.go(...);  // bootstrap 沒有 BuildContext，這行根本寫不出來，但若用 NavigatorKey 也禁止
```

### 4. 阻塞性 / 不確定耗時的網路 I/O

```dart
// ❌ 同步等使用者資料
final user = await api.fetchUser();

// ❌ 等使用者授權結果
await Permission.notification.request();
```

權限請求 / 預載資料 / API 預取應在進入需要的頁面時做，不是啟動時。bootstrap 卡住 = 首屏空白時間延長。

### 5. 條件分支業務決策

```dart
// ❌ 判斷 session 決定 routing
if (session.isAuthenticated) goToHome();
else goToLogin();
```

這類決策屬 router redirect 或 `AppBloc` / orchestration。

### 6. DI 註冊

```dart
// ❌ 在 bootstrap 內手動註冊 DI
getIt.registerSingleton<MyService>(MyServiceImpl());
```

DI 註冊一律走 `@injectable` annotation + `configureDependencies()`，詳見 [`architecture/dependency-injection.md`](../architecture/dependency-injection.md)。

### 7. 把取出的依賴外傳

```dart
// ❌ bootstrap 回傳依賴
Future<AppDependencies> bootstrap() async {
  ...
  return AppDependencies(router: getIt<GoRouter>(), ...);
}
```

```dart
// ❌ 把 getIt 取出的東西塞給 App
runApp(App(router: bootstrapResult.router));
```

bootstrap 是「做完就走」，**不留回傳值、不下傳依賴**。需要的 widget 自己從 `getIt` 取。

### 8. Build flavor / 環境耦合的 if-else 散落

```dart
// ❌ 各 bootstrap 子檔都 if (kDebugMode)
void setupCrashReporting() {
  if (kDebugMode) { /* dev */ } else { /* prod */ }
}
```

環境差異應該透過 DI 註冊不同實作來吸收，bootstrap 對所有 flavor 行為一致。

---

## 灰色地帶（個案判斷）

| 情境 | 建議 | 理由 |
|---|---|---|
| 權限請求（push / location / camera） | ❌ 不在 bootstrap | 進入相關功能前才請求，避免啟動卡權限對話框 |
| 預載 user profile / settings | ❌ 不在 bootstrap | 由 splash page 或 `AppBloc.AppStarted` 處理 |
| App 版本檢查 / 強制更新 | ❌ 不在 bootstrap | 屬於進 app 後第一個 page 的責任，避免阻塞啟動 |
| Local DB 開檔（Hive / Drift open） | ✅ 可以 | 但走 DI lazy 註冊更好；只有「必須在 DI 之前就開」才放 bootstrap |
| Remote config 預取 | ✅ 可以 | 但要設 timeout，避免阻塞啟動 |
| A/B test SDK init | ✅ 可以 | 通常需要在第一個 `runApp` 之前完成才能影響首屏 |

---

## 子檔切分原則

`app/bootstrap/` 內部依「外部 SDK / 副作用種類」分檔。每個子檔對應一個明確的責任，例如：

```text
app/bootstrap/
├── app_bootstrap.dart                     # 對外唯一入口：bootstrap()
├── crash_reporting_bootstrap.dart         # GlobalErrorHandler + Bloc.observer
├── firebase_messaging_bootstrap.dart      # FCM background handler 註冊
├── push_notification_bootstrap.dart       # PushToken / Display / Tap services 啟動
└── system_chrome_bootstrap.dart           # SystemChrome 全域設定
```

### 命名

- 子檔：`{capability}_bootstrap.dart`，內含 `setup{Capability}()` 或 `start{Capability}()` 等 top-level function
- 入口：`app_bootstrap.dart`，唯一 `Future<void> bootstrap()`

### `app_bootstrap.dart` 範例

```dart
Future<void> bootstrap() async {
  await configureDependencies();
  setupCrashReporting();
  setupFirebaseMessaging();
  await setupPushNotification();
  await setupSystemChrome();
}
```

### 為什麼分檔

- 每加一個 SDK / 副作用只動一個檔，merge conflict 機率低
- 讀一個 30 行的子檔比讀 200 行的混合檔好懂
- 子檔可獨立 grep / 替換 / 移除

---

## 啟動順序

`bootstrap()` 內固定順序：

1. **`configureDependencies()`** — DI 必須先就位，後續才能 `getIt`
2. **`setupCrashReporting()`** — 要在其他副作用前，才能捕捉它們的錯誤
3. **其餘 SDK / service bootstrap** — 順序由各副作用之間的依賴決定

### 為什麼順序固定

- DI 沒就位 → 任何 `getIt` 都會炸
- Crash reporting 沒就位 → 後面 bootstrap 出錯會靜默失敗，事後查不到

---

## `getIt` 在 bootstrap 內的使用規則

- 每個子 bootstrap function 內部允許 `getIt<X>()` 取出 service / observer，做完 setup 就結束
- ❌ 不准把 `getIt` 取出的東西當參數傳出 bootstrap function 之外
- ❌ 不准在 `bootstrap/` 之外的地方包裝「取 + 呼叫」的 helper（避免規避邊界）

```dart
// ✅ bootstrap 內取出，做完即丟
void setupCrashReporting() {
  Bloc.observer = getIt<CrashReportBlocObserver>();
  getIt<GlobalErrorHandler>().setup();
}
```

```dart
// ❌ 取出後外傳
class BootstrapResult {
  BootstrapResult({required this.observer});
  final BlocObserver observer;
}

Future<BootstrapResult> bootstrap() async {
  ...
  return BootstrapResult(observer: getIt<CrashReportBlocObserver>());
}
```

---

## 一句話總結

> Bootstrap 做的是「副作用 setup」，不是「業務初始化」。
> 副作用 = 對全域單例 / SDK / observer 賦值；做完就忘記，不留回傳值、不下傳依賴。
> 業務初始化（讀使用者狀態、預載資料、決定首頁）由 `AppBloc` / router redirect / splash page 負責。

---

## Related

- [ADR-013: App Entry / Shell / Bootstrap Layering](../adr/013-app-entry-shell-bootstrap-layering.md) — 三層責任分離決策
- [`architecture/folder-structure.md`](../architecture/folder-structure.md) — 程式入口與 App 殼章節
- [`architecture/dependency-injection.md`](../architecture/dependency-injection.md) — DI 註冊規範（bootstrap 不做 DI 註冊）
