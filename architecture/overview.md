# 架構總覽

> **TL;DR**：Feature-first Clean Architecture，三層分離（Presentation / Domain / Data）+ Core/Shared，依賴方向 Presentation → Domain ← Data，Domain 零 Flutter dependency。

---

## 不可妥協

| # | 規則 | ADR |
|---|---|---|
| 1 | Domain 層零 Flutter / 第三方 import | [ADR-011](../adr/011-domain-zero-flutter-imports.md) |
| 2 | 跨 feature 邊界依「契約 vs 實作」分類：Domain entity / enum / VO / UseCase 與 Page / Widget 允許跨 feature；Data / Repository interface / BLoC 禁止 | [ADR-014](../adr/014-features-cross-import-rules.md) |
| 3 | 依賴方向單向：Presentation → Domain ← Data | — |
| 4 | `features/` 一層深，禁止 sub-feature nesting | — |
| 5 | 一個 feature 一個 RepositoryImpl、最多一個 remote / 一個 local DataSource | — |

---

## 依賴方向

```text
   Presentation  ──────►  Domain  ◄──────  Data
                            ▲
                            │
                       Core / Shared
```

- **Presentation** 只能依賴 Domain（UseCase、Entity、Repository interface）
- **Domain** 不依賴任何外層（純 Dart，零 Flutter）
- **Data** 實作 Domain 的 Repository interface
- **Core / Shared** 不依賴任何 feature
- **Features 跨依賴受限**：契約類（entity / enum / VO / UseCase / Page / Widget）允許；實作 / 狀態類（Data / Repository interface / BLoC）禁止 —— 詳見 [ADR-014](../adr/014-features-cross-import-rules.md)

違反這些是 blocker，不可繞過。

---

## 各層職責

### Presentation 層

- BLoC、Page、Widget
- BLoC 只依賴 domain UseCase（**禁止**直接依賴 Repository / DataSource）
- Page 不寫業務邏輯（不 try-catch、不直接呼 UseCase 邏輯），只 wire BLoC 跟 UI

```dart
// ✅ BLoC 只依賴 UseCase
@injectable
class LoginBloc extends Bloc<LoginEvent, LoginState> {
  LoginBloc({required LoginUseCase loginUseCase})
      : _loginUseCase = loginUseCase,
        super(const LoginInitial());

  final LoginUseCase _loginUseCase;
}
```

```dart
// ❌ BLoC 直接依賴 Repository — 跨層違規
class LoginBloc extends Bloc<LoginEvent, LoginState> {
  LoginBloc({required AuthRepository repository})
      : _repository = repository,
        super(const LoginInitial());

  final AuthRepository _repository;
}
```

**為什麼錯**：BLoC 直連 Repository 等於繞過 UseCase 層；UseCase 是業務邏輯的封裝點（驗證、組合多個 repository、補資料），跳過去後業務邏輯散在 BLoC 各處。

→ 詳見 [`presentation-layer.md`](./presentation-layer.md)、[`patterns/state-management.md`](../patterns/state-management.md)

### Domain 層

- UseCase、Entity、Value Object、Repository interface、Enum
- **零 Flutter / 第三方套件 import**（純 Dart）
- 描述業務規則，不關心 UI、framework、資料來源

```dart
// ✅ Domain 純 Dart
import 'package:fpdart/fpdart.dart';        // OK，FP 工具是 domain 的一部分
import '../entities/user.dart';

abstract interface class AuthRepository {
  Future<Either<AppFailure, User>> login({
    required String email,
    required String password,
  });
}
```

```dart
// ❌ Domain import Flutter
import 'package:flutter/material.dart';     // 違規

abstract interface class AuthRepository { ... }
```

**為什麼錯**：Domain 引入 Flutter 後 (a) 無法在 pure Dart 環境跑單元測試；(b) Flutter API 變動會衝擊業務 code；(c) 違反 Clean Architecture 依賴方向。

→ 詳見 [`domain-layer.md`](./domain-layer.md)、[ADR-011](../adr/011-domain-zero-flutter-imports.md)

### Data 層

- DataSource、DTO、Mapper、RepositoryImpl
- 實作 Domain 的 Repository interface
- 唯一接觸外部資料來源（HTTP、本地 DB、cache）的層
- DataSource 拋 typed exception；RepositoryImpl 一律 catch、map 成 `Either<AppFailure, T>` 回傳

```dart
// ✅ RepositoryImpl 統一回 Either
@LazySingleton(as: AuthRepository)
class AuthRepositoryImpl implements AuthRepository {
  const AuthRepositoryImpl({required AuthRemoteDataSource remoteDataSource})
      : _remoteDataSource = remoteDataSource;

  final AuthRemoteDataSource _remoteDataSource;

  @override
  Future<Either<AppFailure, User>> login({
    required String email,
    required String password,
  }) async {
    try {
      final dto = await _remoteDataSource.login(
        request: LoginRequestDto(email: email, password: password),
      );
      return Right(UserMapper.fromResponseDto(dto));
    } on Object catch (error) {
      return Left(AppFailureMapper.toFailure(error));
    }
  }
}
```

→ 詳見 [`data-layer.md`](./data-layer.md)

### Core / Shared 層

- 跨 feature 共用基礎設施：error 型別（`AppFailure`、`FieldFailure`）、`Session`、`AppConfig`、network client、platform abstraction
- **不能依賴任何 feature**
- 純技術組件，不含業務邏輯

→ 詳見 [`folder-structure.md`](./folder-structure.md)、[`infrastructure/`](../infrastructure/)

---

## 依賴方向規則表

| 層 | 可以 import | 不可以 import |
|---|---|---|
| Presentation | Domain、Core、ui_kit、其他 feature 的 Domain（entity / enum / VO / UseCase）、其他 feature 的 Page / Widget | Data 任何東西、其他 feature 的 BLoC / state / event、其他 feature 的 Repository interface |
| Domain | 純 Dart 標準庫、`fpdart`、其他 feature 的 Domain entity / enum / VO / UseCase | Flutter、Data、第三方框架（除 fpdart）、其他 feature 的 Repository interface |
| Data | 自家 Domain（實作 interface 用）、Core | 其他 feature 的 Data、其他 feature 的 Repository interface |
| Core / Shared | Dart 標準庫、Flutter、第三方 | 任何 feature |
| App（composer） | 任何 feature 的 Domain、任何 feature 的 Page、core、ui_kit | 任何 feature 的 Data、任何 feature 的 BLoC 細節 |

---

## Feature 邊界

### 邊界以**業務能力**劃分，不依後端 API 名

```text
// ✅ 業務能力命名，flat
features/
├── auth/
├── profile/
└── product_catalog/

// ❌ 後端 API 命名
features/
├── auth_api/
├── user_api/
└── product_api/
```

### Features 跨 import 邊界

**契約 vs 實作** 兩刀切：契約類允許跨 feature，實作 / 狀態類禁止。

| 對象 | 跨 feature import |
|---|---|
| Domain entity / enum / VO | ✅ |
| Domain UseCase | ✅ |
| Domain Repository interface | ❌ |
| Data 全部（DTO / Mapper / RepositoryImpl / DataSource） | ❌ |
| Presentation Page + 對外 routing contract type | ✅ |
| Presentation Widget | ✅ |
| Presentation BLoC / state / event / sideEffect | ❌ |

```dart
// ✅ profile feature 的 BLoC 透過跨 feature UseCase 取登入用戶
import '../../../auth/domain/usecases/get_current_user/get_current_user_usecase.dart';

@injectable
class ProfileBloc extends Bloc<ProfileEvent, ProfileState> {
  ProfileBloc({
    required GetProfileUseCase getProfile,
    required GetCurrentUserUseCase getCurrentUser,
  });
}
```

```dart
// ❌ profile feature 直接吃 auth feature 的 Repository interface
import '../../../auth/domain/repositories/auth_repository.dart';

@injectable
class GetProfileUseCase {
  const GetProfileUseCase({
    required ProfileRepository profileRepo,
    required AuthRepository authRepo,    // ❌ 跨 feature Repository interface
  });
}
```

**為什麼錯**：跨 feature 取資料應該走對方的 UseCase（單一動詞、明確 input / output），不可繞過 UseCase 紀律直接吃 Repository。

```dart
// ❌ cart feature 直接 import auth feature 的 BLoC
import '../../../auth/presentation/login_page/bloc/login_bloc.dart';

class CartPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final user = context.watch<LoginBloc>().state;    // ❌ 跨 feature BLoC
    // ...
  }
}
```

**為什麼錯**：BLoC 有 lifetime + `BlocProvider` scope，跨 feature 取對方 BLoC 帶 runtime crash 風險。需要對方資料就走對方 UseCase。

→ 詳見 [ADR-014](../adr/014-features-cross-import-rules.md)

### `features/` 一層深

```text
// ✅ flat
features/
├── auth/
├── product_catalog/
└── settings/

// ❌ sub-feature nesting
features/
└── shop/
    ├── shop_home/
    └── shop_detail/
```

如果一個 feature 太大要拆，拆成 sibling features，不要 nest。

---

## 同層直接依賴禁止

任何同類型的兩個 unit 不能互相依賴：

- UseCase 不依賴 UseCase
- Repository impl 不依賴 Repository impl
- DataSource 不依賴 DataSource
- BLoC 不依賴 BLoC

需要組合多個同類 unit 時，由**上層**協調（如 BLoC handler 內呼叫多個 UseCase）。

---

## 常見錯誤

```dart
// ❌ Presentation 直接 import Data
import 'package:.../features/auth/data/datasources/auth_remote_datasource.dart';

class LoginBloc { ... }
```
**為什麼錯**：跨層違規；Presentation 應該透過 UseCase 才能間接 reach Repository / DataSource。

```dart
// ❌ Domain import Dio / Flutter / 任何外部框架
import 'package:dio/dio.dart';

abstract interface class AuthRepository { ... }
```
**為什麼錯**：Domain 必須是 pure Dart；引入 Dio 違反 Domain 框架無關原則。

```dart
// ❌ Feature A 的 BLoC import Feature B 的 BLoC
import '../../../auth/presentation/login_page/bloc/login_bloc.dart';

class CartBloc extends Bloc<CartEvent, CartState> {
  CartBloc({required LoginBloc loginBloc});    // ❌ 跨 feature BLoC
}
```
**為什麼錯**：跨 feature import BLoC 會綁 lifetime / `BlocProvider` scope，帶 runtime crash 風險。需要對方資料走對方的 **UseCase**（跨 feature UseCase ✅）；需要 session 狀態仍走 `SessionContext`（在 `core/`）。

---

## Related

- [`folder-structure.md`](./folder-structure.md) — 完整目錄結構模板
- [`data-layer.md`](./data-layer.md) — Data 層詳細規範
- [`domain-layer.md`](./domain-layer.md) — Domain 層詳細規範
- [`presentation-layer.md`](./presentation-layer.md) — Presentation 層詳細規範
- [`dependency-injection.md`](./dependency-injection.md) — DI 規範
- [ADR-011](../adr/011-domain-zero-flutter-imports.md)、[ADR-014](../adr/014-features-cross-import-rules.md)（supersedes ADR-012）
