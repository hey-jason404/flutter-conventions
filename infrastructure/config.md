# AppConfig

> **TL;DR**：`AppConfig` 是全域設定 SSOT。env 欄位 `final` 一次設定；runtime 欄位透過 `update()` 修改。Feature 注入 `AppConfig`，**不**注入 `EnvProvider`。

---

## 不可妥協

| # | 規則 |
|---|---|
| 1 | `AppConfig` 是全域設定 SSOT；feature / UseCase / core 一律注入 `AppConfig` |
| 2 | `EnvProvider` 是 `AppConfig` 的 internal detail，feature 不注入它 |
| 3 | env 欄位 `final` 一次設定；runtime 欄位透過 `update()` 改 |
| 4 | 直接 mutate `AppConfig.field = ...` 禁止 |
| 5 | Session 資料（access token、profile）**不屬於** `AppConfig` |

---

## `AppConfig` 結構

```dart
@LazySingleton()
class AppConfig {
  AppConfig({required EnvProvider envProvider})
      : apiBaseUrl = envProvider.apiBaseUrl,
        appVersion = envProvider.appVersion,
        isDebug = envProvider.isDebug,
        flavor = envProvider.flavor;

  // Env-source fields（建構時設一次，final）
  final String apiBaseUrl;
  final String appVersion;
  final bool isDebug;
  final AppFlavor flavor;

  // Runtime fields（可透過 update 修改）
  Map<String, bool> _featureFlags = const {};
  Map<String, bool> get featureFlags => Map.unmodifiable(_featureFlags);

  Locale _locale = const Locale('en');
  Locale get locale => _locale;

  void update({
    Map<String, bool>? featureFlags,
    Locale? locale,
  }) {
    _featureFlags = featureFlags ?? _featureFlags;
    _locale = locale ?? _locale;
  }
}
```

### Env vs runtime

| 類型 | 何時設 | 可變 | 範例 |
|---|---|---|---|
| **Env** | App 啟動時從 env / build flavor 讀 | `final`，不變 | `apiBaseUrl`、`appVersion`、`isDebug`、`flavor` |
| **Runtime** | App 運行時從 server / 使用者選擇來 | 透過 `update()` 改 | `featureFlags`、`locale`、`themeMode` |

---

## `EnvProvider`：env 來源

`EnvProvider` 抽象 build-time env 讀取邏輯（從 `--dart-define`、`.env` file、build flavor 等）：

```dart
abstract interface class EnvProvider {
  String get apiBaseUrl;
  String get appVersion;
  bool get isDebug;
  AppFlavor get flavor;
}

@LazySingleton(as: EnvProvider)
class EnvProviderImpl implements EnvProvider {
  @override
  String get apiBaseUrl => const String.fromEnvironment(
        'API_BASE_URL',
        defaultValue: 'https://api.dev.example.com',
      );

  @override
  String get appVersion => const String.fromEnvironment('APP_VERSION', defaultValue: '0.0.0');

  @override
  bool get isDebug => kDebugMode;

  @override
  AppFlavor get flavor => switch (const String.fromEnvironment('FLAVOR', defaultValue: 'dev')) {
    'prod' => AppFlavor.production,
    'staging' => AppFlavor.staging,
    _ => AppFlavor.development,
  };
}

enum AppFlavor { development, staging, production }
```

### 規則

- **`EnvProvider` 是 `AppConfig` 的 internal detail**
- 只有 `AppConfig` 可以注入 `EnvProvider`
- Feature / UseCase / core / repository **禁** 注入 `EnvProvider`

```dart
// ❌ feature 注入 EnvProvider
@injectable
class GetApiBaseUrlUseCase {
  const GetApiBaseUrlUseCase(this._envProvider);
  final EnvProvider _envProvider;                    // ❌

  String call() => _envProvider.apiBaseUrl;
}

// ✅ feature 注入 AppConfig
@injectable
class GetApiBaseUrlUseCase {
  const GetApiBaseUrlUseCase(this._appConfig);
  final AppConfig _appConfig;

  String call() => _appConfig.apiBaseUrl;
}
```

**為什麼錯**：`EnvProvider` 是底層機制（從哪讀 env）；feature 應該透過抽象層 `AppConfig` 取值，這樣將來 env 來源改變（譬如改用 remote config）feature 不受影響。

---

## Env file source（build-time）

`EnvProviderImpl` 的 `String.fromEnvironment(...)` 值，build 時由 `--dart-define-from-file=env/{flavor}.json` 餵入。File 放在 Flutter 專案根目錄下的 `env/`：

```text
env/
├── dev.json                # 開發環境
├── staging.json            # 測試環境
└── prod.json               # 正式環境
```

### File 格式

```json
// env/dev.json
{
  "API_BASE_URL": "https://api.dev.example.com",
  "APP_VERSION": "1.0.0",
  "FLAVOR": "dev"
}
```

JSON key 一律大寫 `SCREAMING_SNAKE_CASE`，與 `String.fromEnvironment(KEY)` 取的 key 同名。

> 📌 為避免 key 字串散落各處，可在 `core/env/env_keys.dart` 集中常數：
>
> ```dart
> abstract final class EnvKeys {
>   static const String apiBaseUrl = 'API_BASE_URL';
>   static const String appVersion = 'APP_VERSION';
>   static const String flavor = 'FLAVOR';
> }
> ```

### Build 指令

```bash
flutter run --dart-define-from-file=env/dev.json
flutter build apk --dart-define-from-file=env/prod.json
flutter build ipa --dart-define-from-file=env/prod.json
```

### 加新 env key 流程

| 步驟 | 動作 |
|---|---|
| 1 | `env/*.json` 各 flavor file 都加 key（值依環境填）|
| 2 | （可選）`core/env/env_keys.dart` 加常數 |
| 3 | `EnvProvider` interface 加 getter |
| 4 | `EnvProviderImpl` 加 `String.fromEnvironment(...)` 對應行 |
| 5 | `AppConfig` 加 `final` 欄位 + 在建構式從 `envProvider` 取值 |
| 6 | 跑 `build_runner` |

### 規則

- **`env/*.json` 各 flavor file 必須 key set 一致** — 避免 prod build 漏 key 才 runtime 才炸
- **`env/*.json` 若含敏感值（API key、secret）** → git ignore，提供 `env/example.json` 作為新人 onboard 範本
- 不要把 build-time env 跟 runtime config 混淆 — runtime config 透過 `AppConfig.update()` 改，不從 env file 來

---

## 修改 runtime config

```dart
// ✅ 透過 update() 改
@injectable
class UpdateLocaleUseCase {
  const UpdateLocaleUseCase(this._appConfig);
  final AppConfig _appConfig;

  void call({required Locale locale}) {
    _appConfig.update(locale: locale);
  }
}

// ❌ 直接 mutate
class UpdateLocaleUseCase {
  void call({required Locale locale}) {
    _appConfig.locale = locale;                     // ❌
  }
}
```

**為什麼錯**：`update()` 集中所有可變欄位的修改邏輯，便於日後加 validation、log、stream notify；直接賦值繞過這層。

---

## 加新 config 欄位流程

| 步驟 | 動作 |
|---|---|
| 1 | 判斷是 env-source 還是 runtime |
| 2 | 如果是 env：加到 `EnvProvider` interface + impl |
| 3 | 加到 `AppConfig`（env 用 `final`，runtime 用 `private + getter + update`） |
| 4 | 如果是 runtime：擴充 `update()` 方法簽章 |
| 5 | 跑 `build_runner`（如果有 annotation 變動） |
| 6 | Feature inject `AppConfig` 取值 |

---

## Session vs AppConfig 分工

| 屬性 | 屬於 |
|---|---|
| `apiBaseUrl`、`appVersion`、`isDebug` | `AppConfig` |
| `featureFlags`、`locale`、`themeMode` | `AppConfig` |
| `accessToken`、`userProfile` | `Session` |
| 登入狀態 | `SessionContext` |

```dart
// ❌ access token 塞進 AppConfig
class AppConfig {
  String accessToken = '';                          // ❌ 屬於 Session
}

// ❌ apiBaseUrl 塞進 Session
@freezed
abstract class Session with _$Session {
  const factory Session({
    required String accessToken,
    required UserProfile profile,
    required String apiBaseUrl,                     // ❌ 屬於 AppConfig
  }) = _Session;
}
```

判斷依據：

- 跟**使用者身份**綁定 → `Session`
- 跟**環境 / 設定**綁定 → `AppConfig`

---

## 常見錯誤

```dart
// ❌ feature 注入 EnvProvider
@injectable
class GetCdnUrlUseCase {
  const GetCdnUrlUseCase(this._envProvider);
  final EnvProvider _envProvider;
}
```

```dart
// ❌ 直接 mutate AppConfig field
appConfig.locale = const Locale('zh');             // ❌
```

```dart
// ❌ AppConfig 塞 cache 資料
class AppConfig {
  Map<String, Product> _productCache = {};         // ❌ 屬於 Repository cache
}
```

---

## Related

- [`architecture/dependency-injection.md`](../architecture/dependency-injection.md) — `EnvProvider` 與 `AppConfig` 註冊
- [`infrastructure/session.md`](./session.md) — Session vs AppConfig 分工
- [`infrastructure/network.md`](./network.md) — Dio 從 `AppConfig.apiBaseUrl` 取 base URL
