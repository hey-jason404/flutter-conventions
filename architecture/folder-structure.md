# 目錄結構

> **TL;DR**：feature-first 結構。`lib/{app, core, ui_kit, generated, features, main.dart}`，`features/` 下每個 feature 有 `data/ domain/ presentation/` 三層。Feature 一層深，禁止 nesting。

---

## 不可妥協

| # | 規則 | ADR |
|---|---|---|
| 1 | `features/` 一層深，禁止 sub-feature nesting | — |
| 2 | 一個 feature 一個 RepositoryImpl、最多一個 remote / 一個 local DataSource | — |
| 3 | 每個 page 一個 sub-folder，page-level BLoC 命名以 page 為主 | — |
| 4 | `presentation/` 下不放共用 `bloc/` / `pages/` / `widgets/` 目錄 | — |
| 5 | `generated/` 下檔案絕不手改 | — |

---

## 完整 `lib/` 結構

```text
lib/
├── app/
│   └── di/
│       ├── injection_container.dart        # GetIt instance + @InjectableInit
│       └── injection_container.config.dart # generated, do not edit
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
