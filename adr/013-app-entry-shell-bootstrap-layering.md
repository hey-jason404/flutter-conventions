# ADR-013: App Entry / Shell / Bootstrap Layering

## Status

Accepted | Date: 2026-04-30 | Supersedes: -

## Context

一個 Flutter app 啟動時要做的事大致分三類：

1. **拉起 Flutter runtime**（`WidgetsFlutterBinding.ensureInitialized()`、`runApp`）
2. **副作用 setup**（DI 初始化、外部 SDK 全域 callback、`Bloc.observer`、global error handler、長駐 service `start()`）
3. **描述 widget 樹結構**（Scopes、全域 `BlocProvider`、`MaterialApp.router`、Theme）

實務上常見三件事全擠在 `main.dart` 與 `app.dart`，導致：

- `main.dart` 變成「啟動腳本」：FCM background handler 註冊、`Bloc.observer` 賦值、從 DI 取 router 再當參數塞給 `App`、`GlobalErrorHandler.setup()` 等全擠進來
- `app.dart` 角色不一致：對外收 `router` / `pageRouteObserver` 等業務參數，對內又自行 `getIt<...>()` 抓其他依賴；「DI lookup 在哪做」沒有單一原則
- 副作用散落：`PushTokenRefreshService.start()` 之類「不依賴 BuildContext」的副作用，被掛在 widget 的 `didChangeDependencies` 中
- 新人讀 `main.dart` + `app.dart` 看不出「app 啟動到底發生什麼事」，要 grep 整個 repo 才能拼湊全貌

這些問題在中大型專案會持續累積：每加一個 SDK 就在 `main.dart` 多一塊 setup，每加一個全域 service 就在 widget 樹某處 `start()`，最終啟動流程變成考古題。

### 為什麼需要明文規範

既有規範（`architecture/folder-structure.md`、`architecture/dependency-injection.md`）只示意了 `main.dart` 與 `app/di/` 的存在，沒有定義 `main.dart` / `app.dart` 各自的責任邊界，也沒有「副作用該住哪」的概念。沒有規矩之下，每個工程師都覺得自己的加法「很合理」。

## Decision

**將 app 啟動拆成三層，責任彼此不重疊：**

| 層 | 責任 | 產出 |
|---|---|---|
| **入口（Entry）** | 拉起 Flutter runtime → 交棒 bootstrap → `runApp` | `lib/main.dart` |
| **啟動（Bootstrap）** | 所有「widget 樹掛起來之前」必須做完的副作用 | `lib/app/bootstrap/` |
| **殼（Shell）** | App widget 樹根：Scopes、全域 `BlocProvider`、`MaterialApp`、Theme、Router 接線 | `lib/app/app.dart` |

### 三條核心原則

**1. `main.dart` 是入口，不是腳本**

嚴格三行：binding → bootstrap → `runApp`。任何「啟動時要做的事」一律搬到 `bootstrap/`。

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

**2. `app.dart` 是純結構描述，不是接線中樞**

`App` 是 `const StatelessWidget`、不收業務參數。內部子 widget 可以從 `getIt` 取依賴（殼層接線需要），但**不做副作用**（不 `start()`、不註冊 callback、不賦值 observer）。

**3. Bootstrap 集中所有副作用**

凡是「對 SDK / 全域單例做設定」的動作（DI init、SDK callback 註冊、observer 賦值、長駐 service 啟動），全部進 `app/bootstrap/`，由單一入口 `Future<void> bootstrap()` 對外暴露。

### 配套：`getIt` 邊界（本層級內）

| 位置 | 允許？ | 說明 |
|---|---|---|
| `main.dart` | ❌ 禁用 | 入口三行守則已隱含；明文禁止避免日後鬆動 |
| `app/bootstrap/` | ✅ 允許 | 副作用 setup 需要從容器取出 service / observer 實例 |
| `app/app.dart` | ✅ 允許 | 殼層子 widget 需要接線全域依賴 |

> 📌 `features/` / `core/` / 其他層的 `getIt` 規則不在本 ADR 範圍。

### 判斷某段 code 屬於哪一層

- 不需要 `BuildContext`？→ 屬 bootstrap，不准放 widget。
- SDK 全域 callback 註冊 / observer 賦值？→ 屬 bootstrap，不准放 main。
- 描述 widget 樹結構？→ 屬 shell。

## Consequences

### Positive

- **可讀性**：新人讀 `main.dart` + `app.dart` + `bootstrap/` 三處就能完整描述啟動流程
- **可測試性**：`App` 是 `const` widget，widget test 不需 mock 啟動邏輯
- **可演化性**：加新 SDK 只在 `bootstrap/` 加檔，不污染 main 也不污染 widget 樹
- **可審查性**：`main.dart` 被改是 red flag；`bootstrap/` 加檔是預期路徑。規矩讓壞改動顯眼。

### Negative / Trade-offs

- 多一個 `app/bootstrap/` 目錄（中型專案以上分檔有價值，純 PoC 略嫌多餘）
- `App` 不收參數改從內部 `getIt` 取依賴，對「widget 樹零 DI lookup」純粹派不友善（取捨見 Alternatives）
- 既有專案要重構 `main.dart` / `app.dart` / 移動 `start()` 呼叫位置

## Alternatives Considered

### A. Typed Dependencies — Widget 樹零 `getIt`

`main.dart` 從 DI 取出一個 `AppDependencies` typed bundle，逐層下傳到 `App` 與所有子 widget；widget 樹完全不出現 `getIt`。

**為什麼不選**：

- 殼層本來就是 DI 接線層，強迫它走 typed deps 等於把 DI 容器再做一次
- 中大型專案的真痛點在 features 之間不該互抓 `getIt`，那條規矩屬於 features / presentation 規範，不屬於 app shell 層
- 把嚴格度用在不該用的地方會稀釋規矩權威

`features/` 層是否禁用 `getIt` 是另一場討論，本 ADR 不裁定。

### B. Bootstrap 收進 DI（`configureDependencies()` 包副作用）

不另開 `app/bootstrap/`，把 FCM、`Bloc.observer`、error handler 等副作用塞進 `configureDependencies()`。

**為什麼不選**：

- `configureDependencies()` 名字會騙人——它不再是「純 DI」，但既有 [`architecture/dependency-injection.md`](../architecture/dependency-injection.md) 寫的就是純 DI 入口
- 名稱語意混淆是中大型專案最大長期成本之一
- 多一個 `bootstrap/` 目錄的成本，遠低於語意混淆的後果
- 還會連帶要修改 `dependency-injection.md` 的語意，規範增量從「加」變「改」

### C. 不訂規範，依個案判斷

**為什麼不選**：詳見 Context。沒有明文邊界時，散落會持續累積；半年後啟動流程要靠考古才看得懂。

### D. Bootstrap 單一檔不分子模組

`app/bootstrap/` 只放一個 `app_bootstrap.dart`，所有副作用全寫在裡面。

**為什麼不選**：每加一個 SDK 就要動同一檔，merge conflict 機率高；且讀一個 200 行混合 FCM / crash / push / observer 的檔，比讀 5 個 30 行的子檔難。

> 📌 但「每個 SDK / 副作用種類一個子檔」的具體切分原則屬於規範細節，由 [`infrastructure/bootstrap.md`](../infrastructure/bootstrap.md) 規定，不在本 ADR 裁定。

## Related

- 規範：[`architecture/folder-structure.md`](../architecture/folder-structure.md)（程式入口與 App 殼章節）
- 規範：[`infrastructure/bootstrap.md`](../infrastructure/bootstrap.md)（bootstrap 細則：可做 / 禁止 / 順序）
- 既有 ADR：[ADR-011: Domain Zero Flutter Imports](./011-domain-zero-flutter-imports.md)（layer 邊界，責任分層的姊妹）
- 既有 ADR：[ADR-012: Features Never Cross-Import](./012-features-no-cross-import.md)（feature 邊界，責任分層的姊妹）
