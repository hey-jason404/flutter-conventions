# 測試策略

> **TL;DR**：Unit test 涵蓋 BLoC + UseCase + Repository；Widget test 每個 page；Integration test 覆蓋跨層 happy path。Mock 用 `mocktail`，BLoC 測試用 `bloc_test`。

---

## 不可妥協

| # | 規則 | ADR |
|---|---|---|
| 1 | 每個 BLoC handler 至少要有 success + failure 測試 | — |
| 2 | 每個 UseCase 至少一個測試 | — |
| 3 | Mock 只用 `mocktail`，禁用 `mockito` | — |
| 4 | BLoC 測試用 `bloc_test` 不手動 await | — |
| 5 | 不測 implementation detail，測 observable 結果 | — |

---

## 三層測試策略

| 類型 | 測什麼 | 位置 |
|---|---|---|
| **Unit** | UseCase、BLoC、Repository 實作、Mapper、Value Object | `test/features/{feature}/...` |
| **Widget** | Page、共用 widget | `test/features/{feature}/presentation/...` |
| **Integration** | 跨層 flow、feature 主路徑 | `integration_test/` |

### 各類測什麼

**Unit**：純邏輯。Mock 所有外部依賴。執行最快，數量最多。

**Widget**：UI 渲染、互動、State → UI 對應。Mock BLoC，不 mock 整個 Flutter binding。

**Integration**：DI wiring 通、跨層 happy path、feature 主要使用情境（如登入完整流程）。執行慢但抓 wiring bug。

---

## Unit — BLoC

用 `blocTest` 測每個 handler，至少 success + failure 兩個 case。

```dart
class MockLoginUseCase extends Mock implements LoginUseCase {}

void main() {
  late MockLoginUseCase mockLoginUseCase;
  late LoginBloc loginBloc;

  setUp(() {
    mockLoginUseCase = MockLoginUseCase();
    loginBloc = LoginBloc(loginUseCase: mockLoginUseCase);
  });

  tearDown(() => loginBloc.close());

  group('LoginSubmitted', () {
    const tEmail = 'test@example.com';
    const tPassword = 'password123';
    final tUser = User(id: '1', email: tEmail);
    final tInput = LoginInput(email: tEmail, password: tPassword);

    blocTest<LoginBloc, LoginState>(
      'emits [LoginLoading, LoginSuccess] when login succeeds',
      build: () {
        when(() => mockLoginUseCase(input: tInput))
            .thenAnswer((_) async => Right(tUser));
        return loginBloc;
      },
      act: (bloc) => bloc.add(
        const LoginSubmitted(email: tEmail, password: tPassword),
      ),
      expect: () => [const LoginLoading(), LoginSuccess(user: tUser)],
    );

    blocTest<LoginBloc, LoginState>(
      'emits [LoginLoading, LoginFailure] when login fails',
      build: () {
        when(() => mockLoginUseCase(input: tInput))
            .thenAnswer((_) async => const Left(UnknownAppFailure()));
        return loginBloc;
      },
      act: (bloc) => bloc.add(
        const LoginSubmitted(email: tEmail, password: tPassword),
      ),
      expect: () => [
        const LoginLoading(),
        const LoginFailure(failure: UnknownAppFailure()),
      ],
    );
  });
}
```

```dart
// ❌ 手動驅動 stream + await
test('login success', () async {
  loginBloc.add(LoginSubmitted(...));
  await Future.delayed(Duration.zero);
  expect(loginBloc.state, isA<LoginSuccess>());
});
```

**為什麼錯**：`Future.delayed(Duration.zero)` 是脆弱的時序假設；`blocTest` 內建正確的 stream 訂閱與斷言時機。

---

## Unit — SideEffect

`blocTest` 只 assert state；SideEffect 走獨立 stream，要用 `expectLater`：

```dart
test('emits LoginNavigateToHome after successful login', () async {
  const tInput = LoginInput(email: 'test@example.com', password: 'pw');
  final tUser = User(id: '1', email: tInput.email);

  when(() => mockLoginUseCase(input: tInput))
      .thenAnswer((_) async => Right(tUser));

  final sideEffectFuture = expectLater(
    loginBloc.sideEffects,
    emitsInOrder(<LoginSideEffect>[const LoginNavigateToHome()]),
  );

  loginBloc.add(LoginSubmitted(email: tInput.email, password: tInput.password));

  await sideEffectFuture;
});
```

**重點**：`expectLater` 必須在 `add` **之前**設定，否則 broadcast stream 已經發完不會重發。
零參數變體用 `const LoginNavigateToHome()` 比 equality；帶參數用 `isA<T>().having(...)`。

---

## Unit — UseCase

```dart
class MockAuthRepository extends Mock implements AuthRepository {}

void main() {
  late MockAuthRepository mockRepository;
  late LoginUseCase useCase;

  setUp(() {
    mockRepository = MockAuthRepository();
    useCase = LoginUseCase(mockRepository);
  });

  final tUser = User(id: '1', email: 'test@example.com');
  const tInput = LoginInput(email: 'test@example.com', password: 'pw');

  test('returns User on success', () async {
    when(() => mockRepository.login(
          email: tInput.email,
          password: tInput.password,
        )).thenAnswer((_) async => Right(tUser));

    final result = await useCase(input: tInput);

    expect(result, Right(tUser));
    verify(() => mockRepository.login(
          email: tInput.email,
          password: tInput.password,
        )).called(1);
    verifyNoMoreInteractions(mockRepository);
  });
}
```

---

## Unit — Repository

測 exception → AppFailure 的映射：

```dart
test('returns NetworkAppFailure when data source throws NetworkException', () async {
  when(() => mockRemoteDataSource.login(request: any(named: 'request')))
      .thenThrow(const NetworkException('no network'));

  final result = await repository.login(email: 'email', password: 'password');

  expect(result.isLeft(), true);
  expect(result.fold((f) => f, (_) => null), isA<NetworkAppFailure>());
});
```

---

## Widget 測試

每個 page 一份；共用 widget 一份。Mock BLoC。

```dart
class MockLoginBloc extends MockBloc<LoginEvent, LoginState> implements LoginBloc {}

void main() {
  testWidgets('shows loading indicator when LoginLoading state', (tester) async {
    final mockBloc = MockLoginBloc();
    when(() => mockBloc.state).thenReturn(const LoginLoading());
    whenListen(mockBloc, Stream.fromIterable([const LoginLoading()]));

    await tester.pumpWidget(
      MaterialApp(
        home: BlocProvider<LoginBloc>.value(
          value: mockBloc,
          child: const LoginPage(),
        ),
      ),
    );

    expect(find.byType(CircularProgressIndicator), findsOneWidget);
  });
}
```

```dart
// ❌ 在 widget test 啟動真的 DI graph、用真的 BLoC
testWidgets('login flow', (tester) async {
  await configureDependencies(); // 啟整個 app DI
  await tester.pumpWidget(MyApp());
  // ...
});
```

**為什麼錯**：widget test 應該只測 UI 行為。啟整個 DI 是 integration test 的工作；混在一起會 widget test 變慢、failure 難 debug。

---

## Integration 測試

何時寫：

- 跨層 happy path（Presentation → BLoC → UseCase → Repository → DataSource），單層 unit test 抓不到 wiring bug
- App DI wiring 驗證
- Feature 主使用路徑（登入、結帳、onboarding 等核心流程）

何時不寫：

- 單層 case
- 已被 unit test 完整覆蓋且無真實 wiring 風險

位置：

```text
integration_test/
└── features/
    └── auth/
        └── login_flow_test.dart
```

---

## 何時補哪種測試

| 變更類型 | 必補測試 |
|---|---|
| 業務邏輯（UseCase、Repository、BLoC） | **Unit test** |
| UI / 互動（Page、Widget） | **Widget test** |
| 跨層 flow / feature 主路徑 | **Integration test** |

---

## 檔案命名與位置

Mirror `lib/` 結構到 `test/`：

```text
lib/features/auth/domain/usecases/login_usecase.dart
→ test/features/auth/domain/usecases/login_usecase_test.dart

lib/features/auth/presentation/login_page/bloc/login_bloc.dart
→ test/features/auth/presentation/login_page/bloc/login_bloc_test.dart

lib/features/auth/presentation/login_page/login_page.dart
→ test/features/auth/presentation/login_page/login_page_test.dart
```

---

## 不寫測試的後果

- 新增 BLoC / UseCase 沒測試 → PR 阻擋 merge
- 修改 BLoC handler 沒對應補 / 改測試 → PR 阻擋
- 修改 Page UI 沒對應 widget test → PR 視情況阻擋（小 cosmetic 改動可豁免，互動行為改動不可）

---

## Related

- [`patterns/state-management.md`](./patterns/state-management.md) — BLoC / Event / State / SideEffect 結構
- [`patterns/error-handling.md`](./patterns/error-handling.md) — Either / AppFailure / Failure 映射
- [`packages.md`](./packages.md) — 測試套件白名單
