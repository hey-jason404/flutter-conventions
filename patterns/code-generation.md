# Code Generation

> **TL;DR**：Entity / State / Value Object 用 `freezed`；DTO 用 `json_serializable`；Event / SideEffect / UseCase Input 用 plain sealed class（不 codegen）；i18n 用 `slang`；asset 用 `flutter_gen`。Annotation 變更後跑 `build_runner`。

---

## 不可妥協

| # | 規則 | ADR |
|---|---|---|
| 1 | Entity / State / Value Object 用 `freezed` | [ADR-003](../adr/003-bloc-state-freezed-sealed-class.md) |
| 2 | DTO 用 `@JsonSerializable`，**禁** `@freezed` | — |
| 3 | Event / SideEffect / UseCase Input 用 plain sealed class | [ADR-002](../adr/002-bloc-event-plain-sealed-class.md)、[ADR-006](../adr/006-side-effect-plain-sealed-class.md) |
| 4 | 使用者可見字串一律 `context.t.*`（slang） | — |
| 5 | Asset 路徑一律 `Assets.*`（flutter_gen） | — |
| 6 | 生成檔絕不手改（`*.freezed.dart`、`*.g.dart`、`*.config.dart`、`translations.g.dart`） | — |
| 7 | Annotation 變更後一律跑 `build_runner build --delete-conflicting-outputs` | — |

---

## 何時用什麼 codegen

| 情境 | 用什麼 |
|---|---|
| Entity（一般） | `@freezed abstract class` |
| Entity（多型 / union） | `@freezed sealed class` |
| Value Object | `@freezed abstract class` + `create()` factory |
| BLoC State | `@freezed sealed class` |
| DTO（Request / Response / Local） | `@JsonSerializable` |
| BLoC Event | plain sealed class（**不 codegen**） |
| BLoC SideEffect | plain sealed class（**不 codegen**） |
| UseCase Input | plain class（**不 codegen**） |
| 使用者可見字串 | `slang` codegen |
| Asset 路徑 / 顏色 / 字型 | `flutter_gen` codegen |
| DI 註冊 | `injectable` codegen |

---

## `freezed`

### Entity（一般）

```dart
// ✅ @freezed abstract class
@freezed
abstract class User with _$User {
  const factory User({
    required String id,
    required String email,
    required String displayName,
  }) = _User;
}

// usage
final user = User(id: '1', email: 'a@b.com', displayName: 'A');
final user2 = user.copyWith(displayName: 'B');     // ✅ copyWith 自動生成
print(user == user2);                              // ✅ value-equality 自動正確
```

### Entity（多型 / union）

多型 entity 用 `sealed class` —— 配 Dart 3 的 pattern matching 處理 exhaustiveness：

```dart
@freezed
sealed class PaymentMethod with _$PaymentMethod {
  const factory PaymentMethod.creditCard({
    required String maskedNumber,
    required String holderName,
  }) = CreditCardPayment;

  const factory PaymentMethod.bankTransfer({
    required String accountNumber,
  }) = BankTransferPayment;
}

// usage with exhaustive switch
final desc = switch (method) {
  CreditCardPayment(:final maskedNumber) => 'Card $maskedNumber',
  BankTransferPayment(:final accountNumber) => 'Bank $accountNumber',
};
```

### Value Object

```dart
@freezed
abstract class EmailAddress with _$EmailAddress {
  const factory EmailAddress(String value) = _EmailAddress;

  const EmailAddress._();

  static Either<FieldFailure, EmailAddress> create(String raw) {
    final trimmed = raw.trim();
    if (trimmed.isEmpty) return const Left(RequiredFieldFailure());
    if (!trimmed.contains('@')) return const Left(InvalidEmailFormatFailure());
    return Right(EmailAddress(trimmed));
  }
}
```

### BLoC State

```dart
@freezed
sealed class LoginState with _$LoginState {
  const factory LoginState.initial() = LoginInitial;
  const factory LoginState.loading() = LoginLoading;
  const factory LoginState.success({required User user}) = LoginSuccess;
  const factory LoginState.failure({required AppFailure failure}) = LoginFailure;
}
```

---

## `@JsonSerializable`

### Response DTO（只解析）

```dart
@JsonSerializable(createToJson: false)
class GetUserProfileResponseDto {
  const GetUserProfileResponseDto({
    this.userId,
    this.email,
    this.displayName,
  });

  factory GetUserProfileResponseDto.fromJson(Map<String, dynamic> json) =>
      _$GetUserProfileResponseDtoFromJson(json);

  @JsonKey(name: 'user_id')
  final String? userId;
  final String? email;
  @JsonKey(name: 'display_name')
  final String? displayName;
}
```

### Request DTO（只序列化）

```dart
@JsonSerializable(createFactory: false)
class LoginRequestDto {
  const LoginRequestDto({
    required this.email,
    required this.password,
  });

  final String email;
  final String password;

  Map<String, dynamic> toJson() => _$LoginRequestDtoToJson(this);
}
```

### 規則

- **所有 field nullable**（容錯：API 改結構不會炸）
- **不混用 `@freezed`**（freezed 也能做 fromJson，但官方明確不推薦混用）
- 命名：`{Entity}ResponseDto` / `{Action}RequestDto` / `{Entity}LocalDto`

```dart
// ❌ DTO 用 @freezed
@freezed
class LoginRequestDto with _$LoginRequestDto {
  const factory LoginRequestDto({
    required String email,
    required String password,
  }) = _LoginRequestDto;

  factory LoginRequestDto.fromJson(Map<String, dynamic> json) =>
      _$LoginRequestDtoFromJson(json);
}
```

**為什麼錯**：DTO 不需要 `copyWith` / value-equality；freezed 是 overhead；混用 freezed + json_serializable 在 codegen 上偶有 friction。

---

## Plain sealed class（不 codegen）

### BLoC Event

```dart
sealed class LoginEvent {}

final class LoginSubmitted extends LoginEvent {
  const LoginSubmitted({required this.email, required this.password});
  final String email;
  final String password;
}
```

### BLoC SideEffect

```dart
sealed class LoginSideEffect {}

final class LoginNavigateToHome extends LoginSideEffect {
  const LoginNavigateToHome();
}

final class LoginShowError extends LoginSideEffect {
  const LoginShowError({required this.message});
  final String message;
}
```

### UseCase Input（plain class，不需要 sealed）

```dart
class LoginInput {
  const LoginInput({required this.email, required this.password});
  final String email;
  final String password;
}
```

→ 詳見 [`state-management.md`](./state-management.md)、[`architecture/domain-layer.md`](../architecture/domain-layer.md)

---

## `slang`：i18n

使用者可見字串**全用 slang**，禁止字面值：

```dart
// ✅
Text(context.t.login.title);
Text(context.t.login.error.invalidCredentials);

// ❌ hardcoded
Text('Login');
Text('Invalid credentials');
```

**為什麼錯**：

1. 無法在地化（即使現在只支援一個語言）
2. 字串散落 widget code，QA / PM 改文案要 grep 全 repo
3. 編譯期不擋拼錯（slang typed access 會擋）

### File 位置

```text
i18n/                         # source 文案，與 lib/ 同層（Flutter 專案根目錄下）
├── en.i18n.json              # base locale
└── {locale}.i18n.json        # 其他語系（zh / th / vi 等）

lib/generated/localization/
└── translations.g.dart       # slang 產出，**禁止手改**
```

File 命名一律 `{locale}.i18n.json`（locale 用 Flutter `Locale` 對應的 short code，如 `en`、`zh`、`th`、`vi`）。

### File 結構

JSON 用**巢狀結構**按 domain 組織 key，避免單檔平面化造成命名衝突與難 grep：

```json
// i18n/en.i18n.json
{
  "popup": {
    "title": {
      "loggedOut": "Logged out",
      "error": "Error"
    },
    "message": {
      "autoLogout": "You have been inactive for ${minutes} minutes."
    },
    "button": {
      "ok": "OK",
      "cancel": "Cancel"
    }
  },
  "login": {
    "title": "Login",
    "error": {
      "invalidCredentials": "Invalid username or password"
    }
  },
  "statusCode": {
    "GNR00001": "Success",
    "GNR00003": "General Error (${errorCode}). Please reload page and try again."
  }
}
```

對應 slang 產出 typed access：

```dart
context.t.popup.title.loggedOut
context.t.popup.message.autoLogout(minutes: '5')
context.t.statusCode.GNR00003(errorCode: 'X')
```

巢狀分組原則：依**畫面 / 功能 domain** 分（`popup`、`login`、`statusCode`、`tabBar`...），不要把所有 key 拉到 root level。

### `slang.yaml` config

slang 行為由 Flutter 專案根目錄的 `slang.yaml` 控制（最小範例）：

```yaml
input_directory: i18n
input_file_pattern: .i18n.json
output_directory: lib/generated/localization
output_file_name: translations.g.dart
flutter_integration: true
locale_handling: true
translation_class_visibility: public
```

完整 option 見 [slang 官方文件](https://pub.dev/packages/slang)。

### 文案參數

字串內插用 slang `string_interpolation` 規範。slang 預設 `dart` 模式 → `${name}`：

```json
{
  "greeting": "Hello ${name}",
  "remaining": "${count} attempts left",
  "withFallback": "Code (${errorCode})"
}
```

對應 slang 產出 typed callable：

```dart
context.t.greeting(name: 'Alice')
context.t.remaining(count: '3')
```

> 📌 三種 interpolation 模式可在 `slang.yaml` 切換：`dart`（預設，`${name}`）、`braces`（`{name}`）、`double_braces`（`{{name}}`）。**整個專案必須選一種、寫進 `slang.yaml`、所有 i18n file 統一**。混用會 codegen 失敗。

參數命名：

- snake_case 或 camelCase 皆可，**整 repo 一致**
- 不用 Dart keyword（`for`、`is`、`null`、`new`、`class` 等）當參數名

### 加新語系流程

| 步驟 | 動作 |
|---|---|
| 1 | `i18n/{new_locale}.i18n.json` 加新檔，key 結構與 base locale 對齊 |
| 2 | 跑 `dart run build_runner build --delete-conflicting-outputs` |
| 3 | `MaterialApp` 用 slang 產出的 `AppLocaleUtils.supportedLocales` 與對應 delegates |
| 4 | 視需要在 `AppConfig` runtime field 開放 `locale` 切換 UI |

### 規則

- **新增的 locale file 必須與 base locale key 結構對齊**；slang 會檢查 missing key
- **巢狀分組依 domain**，避免單檔平面化（不要把 `login.title` / `popup.title` 都拉到 root level）
- **文案參數遵守 slang `string_interpolation` 模式**（見「文案參數」）；不在 widget 用 Dart 字串拼接（如 `'${context.t.x} world'`）
- **參數命名整 repo 一致**（snake_case 或 camelCase 擇一）
- **不要手改 `translations.g.dart`**（一般 `*.g.dart` 規則同樣適用）

---

## `flutter_gen`：Asset / 顏色 / 字型

```dart
// ✅
Image.asset(Assets.images.logo.path);
Image.asset(Assets.images.products.placeholder.path);

// ❌ hardcoded path
Image.asset('assets/images/logo.png');
Image.asset('assets/images/products/placeholder.png');
```

**為什麼錯**：

1. typo 不會編譯期擋（runtime 才炸）
2. rename / 移檔不會 propagate 到 reference
3. asset 是否真的在 `pubspec.yaml` 註冊也不會擋（漏註冊 runtime 才炸）

flutter_gen 處理：asset、font、color（`Assets.*`、`FontFamily.*`、`ColorName.*`）

---

## `injectable`：DI codegen

詳見 [`architecture/dependency-injection.md`](../architecture/dependency-injection.md)。簡述：

- 每個 class annotate `@injectable` / `@LazySingleton(as: ...)` / `@Singleton`
- `@module` 用於第三方 class
- `@InjectableInit` 在 `injection_container.dart` 標 entry point
- 跑 `build_runner` 產出 `injection_container.config.dart`

---

## `build_runner`

### 何時跑

任何 annotation 變更後：

```bash
dart run build_runner build --delete-conflicting-outputs
```

### `--delete-conflicting-outputs`

預設 build_runner 遇到衝突會 abort；加這個 flag 直接覆寫舊產出。**團隊建議始終加這個 flag**。

### 觸發條件

- 加 / 改 / 刪 任何 `@freezed`、`@JsonSerializable`、`@injectable`、`@module`、`@LazySingleton`、`@Singleton`
- 改 freezed class 的 factory constructor
- 改 `@injectable` class 的 constructor 簽章
- 改 i18n 字串檔
- 加 / 改 asset

### CI 行為

CI build 流程 **應該** 自動跑 build_runner，並驗證沒有 uncommitted diff（避免有人忘記跑）：

```bash
dart run build_runner build --delete-conflicting-outputs
git diff --exit-code     # 有差異就 fail
```

---

## 生成檔規則

```text
{file}.dart                # source（手寫）
{file}.freezed.dart        # generated by freezed
{file}.g.dart              # generated by json_serializable / others
injection_container.config.dart  # generated by injectable
translations.g.dart        # generated by slang
flutter_gen/...            # generated by flutter_gen
```

### 規則

- **絕不手改**任何生成檔
- 是否 commit 生成檔：團隊一致即可（規範不強制）
  - **Commit**：clone 即可 build；CI 不必跑 build_runner；diff 看得到產出變化
  - **Gitignore**：repo 乾淨；但 clone 後第一次 build 必跑

---

## 常見錯誤

```dart
// ❌ DTO 用 freezed
@freezed
class LoginRequestDto with _$LoginRequestDto { ... }

// ❌ Event 用 freezed
@freezed
sealed class LoginEvent with _$LoginEvent { ... }

// ❌ State 用 plain sealed class（無 copyWith / equality）
sealed class LoginState {}
final class LoginSuccess extends LoginState { ... }

// ❌ 手改生成檔
// _$UserFromJson() {  // 編輯 user.g.dart 會被下次 build_runner 蓋掉
//   ...
// }

// ❌ hardcoded 字串
Text('Login');

// ❌ hardcoded asset path
Image.asset('assets/images/logo.png');
```

---

## Related

- [`state-management.md`](./state-management.md) — Event / State / SideEffect 用什麼 codegen
- [`architecture/data-layer.md`](../architecture/data-layer.md) — DTO 規範
- [`architecture/domain-layer.md`](../architecture/domain-layer.md) — Entity / Value Object 用 freezed
- [`architecture/dependency-injection.md`](../architecture/dependency-injection.md) — `injectable` 用法
- [`packages.md`](../packages.md) — codegen 套件白名單
- [ADR-002](../adr/002-bloc-event-plain-sealed-class.md)、[ADR-003](../adr/003-bloc-state-freezed-sealed-class.md)、[ADR-006](../adr/006-side-effect-plain-sealed-class.md)
