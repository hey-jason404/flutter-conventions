# Data Layer

> **TL;DR**：DataSource 接外部資料、用 DTO；Mapper 把 DTO 轉成 Entity；RepositoryImpl 統一回 `Either<AppFailure, T>`，是 exception → AppFailure 的轉換邊界。

---

## 不可妥協

| # | 規則 | ADR |
|---|---|---|
| 1 | DataSource 不回 `Either`，只 throw exception | — |
| 2 | RepositoryImpl 統一 try-catch，map 成 `Either<AppFailure, T>` | [ADR-007](../adr/007-either-pattern-for-domain-results.md) |
| 3 | DTO 用 `@JsonSerializable`，**禁** `@freezed` | — |
| 4 | DataSource 之外不能 access DataSource | — |
| 5 | 業務規則（排序、過濾）放 UseCase，不放 RepositoryImpl | — |
| 6 | 一個 feature 最多一個 RepositoryImpl、一個 remote DataSource、一個 local DataSource | — |

---

## 職責

**Data 層做：**

- DataSource 接外部資料（HTTP API、本地 DB、cache），回 raw DTO
- Mapper 轉 DTO ↔ Entity / Value Object
- RepositoryImpl 協調 DataSource 呼叫、套用 cache 策略、catch exception → 回 `Either`
- 修正 API 資料完整性問題（如去重）在 RepositoryImpl

**Data 層不做：**

- 業務規則、排序邏輯（屬於 UseCase）
- 讓 exception 漏到 Domain 層（必須在 RepositoryImpl catch）
- 在 RepositoryImpl 之外 access DataSource

---

## 目錄結構

```text
features/{feature}/data/
├── datasources/
│   ├── remote/
│   │   ├── {feature}_remote_datasource.dart
│   │   ├── {feature}_remote_datasource_impl.dart
│   │   └── {feature}_api_paths.dart
│   └── local/
│       ├── {feature}_local_datasource.dart
│       └── {feature}_local_datasource_impl.dart
├── models/
│   ├── {action}_response_dto/
│   │   ├── {action}_response_dto.dart
│   │   └── {action}_response_dto.g.dart            # generated
│   ├── {action}_request_dto/
│   └── {entity}_local_dto/
├── mappers/
│   └── {entity}_mapper.dart
└── repositories/
    └── {feature}_repository_impl.dart
```

---

## DataSource

### Interface

`abstract interface class`，方法回 `Future<Dto>` 或 `Future<void>`。**禁止回 `Either` 或 Entity。**

```dart
// ✅ remote datasource interface
abstract interface class AuthRemoteDataSource {
  Future<LoginResponseDto> login({required LoginRequestDto request});
  Future<GetUserProfileResponseDto> getProfile();
}

// ✅ local datasource interface
abstract interface class AuthLocalDataSource {
  Future<UserProfileLocalDto?> getProfile();
  Future<void> saveProfile({required UserProfileLocalDto dto});
  Future<void> clearProfile();
}
```

```dart
// ❌ DataSource 回 Either —— 違規
abstract interface class AuthRemoteDataSource {
  Future<Either<AppFailure, LoginResponseDto>> login({
    required LoginRequestDto request,
  });
}
```

**為什麼錯**：DataSource 只負責 IO；錯誤模型轉換是 RepositoryImpl 的事。混在 DataSource 會讓 exception → AppFailure 的邊界變模糊。

### Remote 實作

註冊 `@LazySingleton(as: Interface)`，注入抽象的 HTTP 客戶端（不要直接 `Dio()`）：

```dart
@LazySingleton(as: AuthRemoteDataSource)
class AuthRemoteDataSourceImpl implements AuthRemoteDataSource {
  const AuthRemoteDataSourceImpl({required Dio dio}) : _dio = dio;

  final Dio _dio;

  @override
  Future<LoginResponseDto> login({required LoginRequestDto request}) async {
    final response = await _dio.post<Map<String, dynamic>>(
      AuthApiPaths.login,
      data: request.toJson(),
    );
    return LoginResponseDto.fromJson(response.data!);
  }

  @override
  Future<GetUserProfileResponseDto> getProfile() async {
    final response = await _dio.get<Map<String, dynamic>>(
      AuthApiPaths.getProfile,
    );
    return GetUserProfileResponseDto.fromJson(response.data!);
  }
}
```

> **API path 集中於 `{feature}_api_paths.dart`**：
>
> ```dart
> abstract final class AuthApiPaths {
>   static const String login = '/auth/login';
>   static const String getProfile = '/auth/profile';
> }
> ```
>
> 禁止 hard-coded path 字面值散落 DataSource method 內。

### Local 實作

```dart
@LazySingleton(as: AuthLocalDataSource)
class AuthLocalDataSourceImpl implements AuthLocalDataSource {
  const AuthLocalDataSourceImpl({required SharedPreferences prefs})
      : _prefs = prefs;

  final SharedPreferences _prefs;

  static const _profileKey = 'auth_user_profile';

  @override
  Future<UserProfileLocalDto?> getProfile() async {
    final raw = _prefs.getString(_profileKey);
    if (raw == null) return null;
    return UserProfileLocalDto.fromJson(
      jsonDecode(raw) as Map<String, dynamic>,
    );
  }

  @override
  Future<void> saveProfile({required UserProfileLocalDto dto}) {
    return _prefs.setString(_profileKey, jsonEncode(dto.toJson()));
  }

  @override
  Future<void> clearProfile() {
    return _prefs.remove(_profileKey);
  }
}
```

### In-memory DataSource

只需要 session 期間存活的資料，可用 in-memory DataSource，**直接存 Entity，不需要 LocalDto**：

```dart
// ✅ in-memory — 存 Entity 不存 LocalDto
abstract interface class FeatureFlagsLocalDataSource {
  FeatureFlags? get cached;
  void cache(FeatureFlags flags);
}

@LazySingleton(as: FeatureFlagsLocalDataSource)
class FeatureFlagsLocalDataSourceImpl implements FeatureFlagsLocalDataSource {
  FeatureFlags? _cached;

  @override
  FeatureFlags? get cached => _cached;

  @override
  void cache(FeatureFlags flags) => _cached = flags;
}
```

LocalDto 只在資料**真的持久化**（SharedPreferences、Hive、SQLite）時才需要。

---

## DTO

三種 DTO，各有命名規則：

| 類型 | 命名 | 用途 |
|---|---|---|
| Response DTO | `{Entity}ResponseDto` 或 `{Action}ResponseDto` | API 回傳資料 |
| Request DTO | `{Action}RequestDto` | API 請求資料 |
| Local DTO | `{Entity}LocalDto` | 本地持久化資料 |

### 規則

- **用 `@JsonSerializable`，禁 `@freezed`**
- 所有 field **nullable**（容錯：API 改資料結構不會炸）
- 一個 DTO 一個目錄（含 `.g.dart` codegen 檔）

```dart
// ✅ Response DTO
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

// ✅ Request DTO
@JsonSerializable(createFactory: false)
class LoginRequestDto {
  const LoginRequestDto({required this.email, required this.password});

  final String email;
  final String password;

  Map<String, dynamic> toJson() => _$LoginRequestDtoToJson(this);
}
```

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

**為什麼錯**：`@freezed` 對 DTO 是 overhead；DTO 是純資料容器，不需要 `copyWith` / value-equality；混用 freezed + json_serializable 在 codegen 上偶有 friction。

---

## Mapper

`abstract final class`，static-only，**不註冊 DI**。

### 規則

- 一個 Entity 一個 Mapper：`{Entity}Mapper`
- 用 `?? defaultValue` 處理 nullable DTO field（**禁拋例外**）
- 雙向轉換時提供 `fromDto` / `toDto` 兩個 method

```dart
// ✅ Mapper — abstract final class，static-only
abstract final class UserMapper {
  static User fromResponseDto(GetUserProfileResponseDto dto) {
    return User(
      id: dto.userId ?? '',
      email: dto.email ?? '',
      displayName: dto.displayName ?? '',
    );
  }

  static User fromLocalDto(UserProfileLocalDto dto) {
    return User(
      id: dto.userId,
      email: dto.email,
      displayName: dto.displayName,
    );
  }

  static UserProfileLocalDto toLocalDto(User entity) {
    return UserProfileLocalDto(
      userId: entity.id,
      email: entity.email,
      displayName: entity.displayName,
    );
  }
}

// ✅ usage
final user = UserMapper.fromResponseDto(dto);
```

```dart
// ❌ Mapper 拋 exception
abstract final class UserMapper {
  static User fromResponseDto(GetUserProfileResponseDto dto) {
    if (dto.userId == null) throw FormatException('missing userId');
    return User(id: dto.userId!, email: dto.email!, displayName: dto.displayName!);
  }
}
```

**為什麼錯**：Mapper 拋 exception 後，RepositoryImpl 必須再 catch；Mapper 用 `?? default` 容錯更穩，業務驗證留給 Value Object / UseCase。

---

## RepositoryImpl

### 結構

- 註冊 `@LazySingleton(as: <Interface>)`
- 注入 DataSource（remote / local）
- 統一 try-catch，包成 `Either<AppFailure, T>` 回傳

```dart
@LazySingleton(as: AuthRepository)
class AuthRepositoryImpl implements AuthRepository {
  const AuthRepositoryImpl({
    required AuthRemoteDataSource remoteDataSource,
    required AuthLocalDataSource localDataSource,
  })  : _remoteDataSource = remoteDataSource,
        _localDataSource = localDataSource;

  final AuthRemoteDataSource _remoteDataSource;
  final AuthLocalDataSource _localDataSource;

  @override
  Future<Either<AppFailure, User>> login({
    required String email,
    required String password,
  }) async {
    try {
      final dto = await _remoteDataSource.login(
        request: LoginRequestDto(email: email, password: password),
      );
      final user = UserMapper.fromResponseDto(dto);
      await _localDataSource.saveProfile(dto: UserMapper.toLocalDto(user));
      return Right(user);
    } on Object catch (error) {
      return Left(AppFailureMapper.toFailure(error));
    }
  }

  @override
  Future<Either<AppFailure, User>> getCachedUser() async {
    try {
      final dto = await _localDataSource.getProfile();
      if (dto == null) return const Left(NotFoundAppFailure());
      return Right(UserMapper.fromLocalDto(dto));
    } on Object catch (error) {
      return Left(AppFailureMapper.toFailure(error));
    }
  }
}
```

### 規則

- **單一 catch block via AppFailureMapper** —— 不要分多個 catch 不同 exception
- **禁用 specific exception catch**（如 `on DioException catch (e)`）—— mapping 邏輯集中在 `AppFailureMapper`

```dart
// ❌ 在 RepositoryImpl catch 特定 exception
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
  } on DioException catch (e) {                          // ❌
    return Left(NetworkAppFailure(message: e.message));
  } on FormatException catch (e) {                       // ❌
    return Left(ValidationAppFailure(message: e.message));
  } catch (e) {
    return Left(UnknownAppFailure(message: e.toString()));
  }
}
```

**為什麼錯**：每個 RepositoryImpl 都要重複 catch 邏輯，難維護；應該在 `AppFailureMapper` 集中處理。

---

## Cache 策略

需要 cache 時，由 RepositoryImpl 協調：

```dart
// ✅ read-through cache：先 local、沒有 fallback remote、寫回 local
@override
Future<Either<AppFailure, User>> getProfile() async {
  try {
    final localDto = await _localDataSource.getProfile();
    if (localDto != null) {
      return Right(UserMapper.fromLocalDto(localDto));
    }

    final remoteDto = await _remoteDataSource.getProfile();
    final user = UserMapper.fromResponseDto(remoteDto);
    await _localDataSource.saveProfile(dto: UserMapper.toLocalDto(user));
    return Right(user);
  } on Object catch (error) {
    return Left(AppFailureMapper.toFailure(error));
  }
}
```

策略選擇（read-through / write-through / cache-aside / TTL）依業務需求，但**控制邏輯永遠在 RepositoryImpl**，不滲到 Domain。

---

## 同層直接依賴禁止

```dart
// ❌ RepositoryImpl 依賴另一個 RepositoryImpl
@LazySingleton(as: ProductRepository)
class ProductRepositoryImpl implements ProductRepository {
  const ProductRepositoryImpl({
    required ProductRemoteDataSource ds,
    required AuthRepository authRepository,    // ❌
  });
}
```

**為什麼錯**：跨 feature 耦合，feature 互相影響；應該在 UseCase 層協調多個 Repository。

```dart
// ❌ DataSource 依賴另一個 DataSource
@LazySingleton(as: ProductRemoteDataSource)
class ProductRemoteDataSourceImpl implements ProductRemoteDataSource {
  const ProductRemoteDataSourceImpl({
    required Dio dio,
    required AuthRemoteDataSource authDs,      // ❌
  });
}
```

**為什麼錯**：DataSource 應該是單純 IO；組合多個來源是 RepositoryImpl 或更上層的事。

---

## 新增 API endpoint 流程

1. 加 path 到 `{feature}_api_paths.dart`
2. 建 `{action}_request_dto/`（如有 request body）
3. 建 `{action}_response_dto/`
4. 跑 `dart run build_runner build`
5. 在 DataSource interface 加 method
6. 在 DataSource impl 實作（call dio + return DTO）
7. 在 Mapper 加 DTO ↔ Entity 轉換（如需要）
8. 在 RepositoryImpl 加 method（call DS + map + try-catch + return Either）
9. 在 Repository interface（domain 層）加對應 method 簽章
10. 寫測試（DataSource impl、RepositoryImpl）

---

## Related

- [`domain-layer.md`](./domain-layer.md) — Repository interface 定義在 domain 層
- [`patterns/error-handling.md`](../patterns/error-handling.md) — `AppFailure` / `AppFailureMapper`
- [`patterns/code-generation.md`](../patterns/code-generation.md) — `@JsonSerializable` 規範
- [`infrastructure/network.md`](../infrastructure/network.md) — Dio / API path 慣例
- [ADR-007](../adr/007-either-pattern-for-domain-results.md)
