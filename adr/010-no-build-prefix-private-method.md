# ADR-010: No `_build` Prefix Private Methods for Widgets

## Status

Accepted | Date: 2024-05-25 | Supersedes: -

## Context

Flutter widget 變大時要拆分。常見拆分方式有兩種：

### 方式 A：用 `_buildXxx()` private method 拆

```dart
class LoginPage extends StatelessWidget {
  const LoginPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: _buildAppBar(),
      body: Column(
        children: [
          _buildEmailField(),
          _buildPasswordField(),
          _buildSubmitButton(),
        ],
      ),
    );
  }

  AppBar _buildAppBar() => AppBar(title: const Text('Login'));
  Widget _buildEmailField() => const TextField(...);
  Widget _buildPasswordField() => const TextField(..., obscureText: true);
  Widget _buildSubmitButton() => ElevatedButton(...);
}
```

### 方式 B：抽獨立 widget class

```dart
class LoginPage extends StatelessWidget {
  const LoginPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Login')),
      body: const _LoginForm(),
    );
  }
}

class _LoginForm extends StatelessWidget {
  const _LoginForm();

  @override
  Widget build(BuildContext context) => const Column(
    children: [_EmailField(), _PasswordField(), _SubmitButton()],
  );
}

class _EmailField extends StatelessWidget {
  const _EmailField();
  @override
  Widget build(BuildContext context) => const TextField(...);
}

// ... etc
```

### 為何方式 A 是反 pattern

1. **失去 const 優化**
   - `_buildEmailField()` 回傳的 widget **無法 const 化**（function call 不是 const expression）
   - 每次 build 都重新建立 widget instance
   - 即使 `TextField` 本身 const、整個樹也不是 const

2. **DevTools widget tree 看不到子 widget**
   - 用 `_buildXxx()` 拆，DevTools 顯示「一個大 LoginPage」
   - 用獨立 class 拆，DevTools 顯示 `LoginPage > _LoginForm > _EmailField` 階層
   - debug / inspect 時前者完全失明

3. **Rebuild 範圍失控**
   - parent rebuild 時，`_buildXxx()` 全部跟著重跑（function 不是 widget tree node）
   - 獨立 class 是 widget，framework 看 `==` 跳過未變的子樹（const 化的子 widget 永遠跳過）
   - 結果：方式 A 在大 page 上**所有子部分每次 build 都重建**

4. **無法獨立測試**
   - `_buildEmailField()` 是 private method，widget test 無法直接測
   - 獨立 class（即使 private `_EmailField`）可以透過測試 parent 間接覆蓋

5. **命名 `_build` 暗示「建構動作」，但 widget 本來就是 declarative**
   - widget 是「描述」不是「動作」
   - `_buildXxx()` 把 widget 當 imperative function 用，違反 Flutter declarative model

### 為何方式 B 是正解

- 保留 const 優化 → rebuild skip
- DevTools 看得到階層
- 可獨立測試
- 命名清楚（`_LoginForm` vs `_buildLoginForm()` —— 前者是名詞，符合 widget 本質）

### 拆分位置慣例

- 只在這個 page 用 → 同檔尾用 private class（`class _LoginForm`）
- feature 內多 page 共用 → 主 page 的 `widgets/`
- 跨 feature 共用 → `ui_kit/`

## Decision

**Widget 拆分一律用獨立 class（`StatelessWidget` / `StatefulWidget`）。禁用 `_buildXxx()` 私有 method 拆 widget。**

不限於 `_build` prefix —— 任何回傳 `Widget` 的 private method **拆 widget 用** 都禁止。

```dart
// ❌ 任何回傳 Widget 的 private method
Widget _someWidgetMethod() { ... }
Widget _renderHeader() { ... }
```

## Consequences

### Positive

- 保留 const 優化 → rebuild 範圍精準
- DevTools widget tree 顯示完整階層
- 子 widget 可獨立測試
- code 結構符合 declarative model

### Negative / Trade-offs

- 程式碼略多（class 比 method 啰嗦）
- 命名負擔（每個子 widget 要起 class 名）
- 對「funnctional 風格」愛好者初期感覺繁瑣

## Alternatives Considered

### `_buildXxx()` 私有 method

**為什麼不選**：詳見 Context 五個問題。const 優化、DevTools、rebuild 範圍、testability 全是真實成本。

### 單一巨大 `build()`（不拆）

**為什麼不選**：

- 可讀性差
- rebuild 範圍最大
- code review 焦點散落

### Helper class（混合 widget 跟 method）

**為什麼不選**：跟方式 A 相同的 const / rebuild 問題。

## Related

- 規範：[`architecture/presentation-layer.md`](../architecture/presentation-layer.md)
