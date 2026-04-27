# ADR-012: Features Never Cross-Import

## Status

Accepted | Date: 2024-06-05 | Supersedes: -

## Context

Feature-first 架構下，每個 `features/{name}/` 是獨立的業務單位。當 feature A 需要 feature B 的能力時，有兩種做法：

### 1. 直接 import feature B

```dart
// features/profile/domain/usecases/get_profile/get_profile_usecase.dart
import '../../../auth/domain/usecases/get_current_user_usecase.dart';     // ❌

@injectable
class GetProfileUseCase {
  const GetProfileUseCase(this._getCurrentUser, this._repository);
  final GetCurrentUserUseCase _getCurrentUser;
  final ProfileRepository _repository;
}
```

問題：

- Feature B 的 internal 改動可能波及 A
- Feature 越多耦合圖越亂（N 個 feature 可能 N×N 互相依賴）
- 無法獨立替換 / 移除 feature
- 測試 feature A 時要 mock feature B 的內部
- 違反 modular 原則

### 2. 透過 `core/` 抽象介面解耦

```dart
// core/session/session_context.dart
abstract interface class SessionContext {
  String? get currentUserId;
  bool get isAuthenticated;
}

// features/profile/domain/usecases/get_profile/get_profile_usecase.dart
import 'package:.../core/session/session_context.dart';

@injectable
class GetProfileUseCase {
  const GetProfileUseCase(this._sessionContext, this._repository);
  final SessionContext _sessionContext;     // 來自 core，不是 feature
  final ProfileRepository _repository;

  Future<Either<AppFailure, Profile>> call() async {
    final userId = _sessionContext.currentUserId;
    if (userId == null) return const Left(UnauthorizedAppFailure());
    return _repository.getProfile(userId: userId);
  }
}

// features/auth/.../session_manager_impl.dart
@LazySingleton(as: SessionContext)
class SessionManager implements SessionContext { ... }
```

優勢：

- profile feature 只知道 `SessionContext` 介面，不知道 auth feature 存在
- auth feature 改 internal 不波及 profile（介面契約穩定即可）
- 測試 profile 用 mock `SessionContext`，不涉及 auth
- 換掉 auth 實作（譬如改用第三方 SSO）只要新實作滿足 `SessionContext` 即可

### 為何 Domain entity / interface 跨 feature 容許

雖然原則是「features 不互相 import」，但**跨 feature 共用 Domain entity / interface 是允許的**：

- `User`、`Product` 這類核心 entity 跨 feature 是常見業務需求
- Domain entity 是純資料 + 介面，不含實作細節
- 引用 Domain interface 等於引用契約，不是依賴實作

```dart
// ✅ profile feature 引用 auth feature 的 User entity
import '../../../auth/domain/entities/user.dart';

@freezed
abstract class Profile with _$Profile {
  const factory Profile({
    required User user,
    required String displayName,
  }) = _Profile;
}
```

但 **跨 feature 的 Data / Presentation / DataSource / Repository impl / BLoC** 一律禁止：

- Data 層：DTO、Mapper、RepositoryImpl、DataSource
- Presentation：Page、Widget、BLoC
- 因為這些是實作細節，跨 feature 引用直接綁定耦合

### 同層直接依賴一併禁止

順帶規定：

- UseCase 不依賴 UseCase（同 feature 也不行）
- Repository impl 不依賴 Repository impl
- DataSource 不依賴 DataSource
- BLoC 不依賴 BLoC

需要組合多個同類 unit 時，由**上層**協調（如 BLoC handler 內呼叫多個 UseCase）。

## Decision

**Features 之間禁止互相 import 任何實作細節（Data / Presentation / 同 feature 的 DataSource / Repository impl / BLoC）。**

**容許**跨 feature 共用 Domain entity 與 Domain Repository interface（純契約）。

跨 feature 共用「能力」時，透過 `core/` 抽象介面解耦。

```text
| 跨 feature import 對象 | 容許 |
|---|---|
| Domain entity（純資料 class） | ✅ |
| Domain Repository interface | ✅ |
| Domain UseCase（其他 feature 的） | ❌ |
| Data DTO / Mapper / RepositoryImpl / DataSource | ❌ |
| Presentation Page / Widget / BLoC | ❌ |
```

## Consequences

### Positive

- Feature 邊界清晰、獨立可測試
- Feature 可獨立替換 / 移除（譬如 A/B test 切換 feature 實作）
- 耦合圖簡單（feature → core，而非 N×N）
- 大型團隊多 feature 並行開發摩擦小

### Negative / Trade-offs

- 共用功能要先抽到 `core/` 或建介面，比直接 import 多一步設計
- `core/` 會累積（Session、AppConfig、各種 abstraction）
- 對快速 prototype 階段稍嫌啰嗦

## Alternatives Considered

### 容許 feature 互 import

**為什麼不選**：詳見 Context #1。耦合圖快速失控，後期重構成本高。

### 單向依賴（A → B 但 B 不能 → A）

**為什麼不選**：

- 「哪個 feature 是 base」邊界主觀
- 隨產品演進，依賴方向會反轉
- 不如統一禁止

### 容許跨 feature import UseCase

**為什麼不選**：

- UseCase 是業務邏輯實作，跨 feature import 等於耦合業務
- 真要共用「能力」，抽 interface 到 `core/`，feature 各自實作

## Related

- 規範：[`architecture/overview.md`](../architecture/overview.md)、[`architecture/folder-structure.md`](../architecture/folder-structure.md)
- 相關 ADR：[ADR-011: Domain Zero Flutter Imports](./011-domain-zero-flutter-imports.md)（layer 邊界，feature 邊界的姊妹）
