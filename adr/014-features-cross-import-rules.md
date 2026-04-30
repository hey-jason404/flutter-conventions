# ADR-014: Features Cross-Import Rules

## Status

Accepted | Date: 2026-04-30 | Supersedes: [ADR-012](./012-features-no-cross-import.md)

## Context

[ADR-012](./012-features-no-cross-import.md) 採取嚴格立場：跨 feature 只允許 Domain entity 與 Domain Repository interface，禁止跨 feature 的 UseCase、Data、Presentation。實務上踩出三組摩擦：

### 1. 共用 UI 元件無自然歸屬

當一個 widget 被多個 page / feature 在 AppBar slot、leading、action 之類位置重複放置（譬如顯示登入用戶資訊、跳轉到設定的 icon button），ADR-012 下要嘛「全搬到 `core/shared` 或 `app/`，連同它的 BLoC 與 use case」，要嘛「在 `core/` 抽介面、各 feature 實作」。前者把 feature 業務邏輯拖出 feature；後者每個共用 widget 都建一套抽象，ceremony 過重。

### 2. Repository interface 跨 feature 開放、UseCase 跨 feature 禁止 —— 方向反了

ADR-012 容許 Repository interface 跨 feature，禁 UseCase 跨 feature。但 Repository 是寬契約（多方法、可能 leak DataSource 細節），UseCase 是窄契約（單一動詞、input → output 顯式）。**契約越窄越該開放才對**。

### 3. Service feature 的 page hand-off 沒有自然解法

當一個 page 被多個 flow 觸發、完成後回傳結果（驗證、選取、確認類流程），caller feature 既不能 import 對方 page、也不能 import 對方 UseCase。在 ADR-012 下只能透過 `core/` 建抽象介面，但這類 navigation hand-off 介面散在 `core/` 是錯位 —— `core/` 應是純基礎能力，不是 feature 對外契約的家。

### 結論

ADR-012 的禁令打到的範圍超過必要，且方向（禁 UseCase 開 Repository）與「窄契約 vs 寬契約」相反。實務上真正會踩雷的是跨 feature 牽動**狀態 / 實作**（Data / BLoC），而不是契約類構件（Page / Widget / UseCase / Entity）。

## Decision

**重新切跨 feature 邊界，依「契約 vs 實作」而非「層」分類。**

### 跨 feature 邊界表

| 對象 | 跨 feature import |
|---|---|
| Domain entity | ✅ |
| Domain enum | ✅ |
| Domain Value Object | ✅ |
| Domain UseCase | ✅ |
| Domain Repository interface | ❌ |
| Data 全部（DTO / Mapper / RepositoryImpl / DataSource） | ❌ |
| Presentation Page **+ 其對外 routing contract type**（params / result） | ✅ |
| Presentation Widget | ✅ |
| Presentation BLoC / state / event / sideEffect | ❌ |

### 配套規則

#### 1. 同層直接依賴禁止（不論是否跨 feature）

| 同層依賴 | 是否允許 |
|---|---|
| BLoC → 別的 BLoC | ❌ |
| UseCase → 別的 UseCase | ❌ |
| Repository → 別的 Repository | ❌ |
| DataSource → 別的 DataSource | ❌ |
| Mapper → 別的 Mapper | ❌ |

需要組合多個同類 unit 時，由**上層**協調（如 BLoC handler 內呼叫多個 UseCase）。

#### 2. 依賴鏈嚴格往內一層

```text
BLoC  →  UseCase  →  Repository interface  →  DataSource
```

- BLoC 不可跳過 UseCase 直接吃 Repository
- UseCase 不可跳過 Repository 直接吃 DataSource

#### 3. Page 跨 import 預設仍透過 router

技術上允許 caller 直接 `import` 別 feature 的 Page widget，但實務上應透過 router facade（在 `app/`）以 type-safe 方式 navigate；direct widget import 僅限「同一個導航 stack 中作為子畫面 embed」這類少見場景。

#### 4. Page 對外 routing contract type 與 page 同居

Page 的入口 params / 出口 result type（用於 router handoff）放在 page 同目錄，視為 page 的公開介面、可跨 feature import：

```text
features/{feature}/presentation/{page_name}/
├── {page_name}_page.dart
├── {page_name}_params.dart      # ← page 入口契約
├── {page_name}_result.dart      # ← page 出口契約
├── bloc/
└── widgets/
```

驅動業務邏輯的 type（如 enum 用於 UseCase 內分流邏輯、entity 用於業務資料）仍住 `domain/`。

判斷準則：「**換 router 框架，這個 type 還在嗎？**」
- 只為 routing handoff 存在 → 跟 page 同居
- 驅動業務邏輯 → domain 層

#### 5. Shell-level 共用 UI 屬於 `app/`

跨 page / 跨 feature 用的 shell-level UI（譬如所有 tab 共用的 AppBar slot widget）住 `app/`，不住 `features/`。它們不是 feature widget 放錯地方，是 app shell 的合法子模組。具體子目錄結構由各專案自訂。

#### 6. 三類重用模式判斷

| 重用形態 | 特徵 | 住哪 | 怎麼被使用 |
|---|---|---|---|
| Shell-level UI | 組裝到多個 page 的 AppBar / 導航相關 widget | `app/` | feature page 直接 `import` |
| Service feature | 被多個 flow hand-off、完成後回傳結果的獨立 page（驗證 / 選取 / 確認類） | `features/<name>/` | 走 `app/` 內的 router facade + `import` 對方 domain entity / page contract type |
| 純 UI primitive | 無業務邏輯、無 BLoC | `ui_kit/` | 直接 `import` widget |

#### 7. `app/` 是 composer，不受跨 feature 規則限制

`app/` 不是 feature。可以 import 任何 feature 的 domain（含 Repository interface）、presentation Page；可以容納跨 feature 流程協調器、router facade、shell UI 與其 BLoC。

#### 8. `core/` 仍不可依賴任何 feature

`core/` 是被所有人依賴的基礎層，**永遠不可** import `features/*`。這條從 ADR-012 沿用，不變。

## Consequences

### Positive

- 共用 UI 元件不必為了規矩搬家或建抽象介面
- Service feature 的 caller 只要 import domain entity + page contract type + 走 `app/` router facade，最少 ceremony
- UseCase 成為跨 feature 的合法業務操作 contract，紀律從「跨 feature ❌」改成「呼叫 UseCase 而非 Repository」
- 同層禁互依 + 依賴鏈往內一層的規則清楚可機械檢查

### Negative / Trade-offs

- 跨 feature import 比 ADR-012 多，刪除 / 重命名 feature 時需要 grep 範圍比較大
- Page 跨 import 規範允許但實務上仍應走 router，需要 reviewer 把關不要被誤用
- `app/` 會多出 shell UI 子目錄，需要明確界定「是 shell 還是 feature」

## Alternatives Considered

### 1. 維持 ADR-012 嚴格規則

**為什麼不選**：解共用 UI / service feature 真實 case 時，被迫做出「全搬到 `app/`」、「在 `core/` 建抽象介面」這類動作 —— 實際是把耦合換個位置，並未消減耦合本身。Reviewer 的注意力被花在「東西該不該搬」，而非「依賴方向是否正確」。

### 2. 只禁 Data layer，其他全開放

**為什麼不選**：放掉 BLoC 跨 import 會帶 lifetime / `BlocProvider` runtime crash 風險；放掉 Repository interface 跨 import 會繞過 UseCase 紀律，BLoC → Repository 變成可能。這兩條真的會踩雷。

### 3. 只開放 Widget，其他禁

**為什麼不選**：解掉共用 widget case 但沒解掉 Service feature handoff。Service feature 需要跨 feature 取 entity / page contract，封死 Page / UseCase / entity 跨 import 仍然走不通。

### 4. 用 import lint 強制檢查

**為什麼選 + 不選**：規範本身仍以這份 ADR 為準；lint 自動化是後續工具化選項，不影響本次規範決策。

## Related

- 規範：[`architecture/overview.md`](../architecture/overview.md)、[`architecture/domain-layer.md`](../architecture/domain-layer.md)、[`architecture/presentation-layer.md`](../architecture/presentation-layer.md)、[`patterns/state-management.md`](../patterns/state-management.md)、[`patterns/navigation.md`](../patterns/navigation.md)
- 相關 ADR：
  - [ADR-011: Domain Zero Flutter Imports](./011-domain-zero-flutter-imports.md)（layer 邊界，feature 邊界的姊妹）
  - [ADR-012: Features Never Cross-Import](./012-features-no-cross-import.md)（被本 ADR supersede）
  - [ADR-013: App Entry / Shell / Bootstrap Layering](./013-app-entry-shell-bootstrap-layering.md)（`app/` 三層分離；本 ADR 沿用 `app/` 為 composer）
