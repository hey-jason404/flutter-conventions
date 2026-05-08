# ADR-015: Cached Network Image with Wrapper Entry

## Status

Accepted | Date: 2026-05-07 | Supersedes: -

## Context

Mobile App 載入網路圖片有兩條路：

- Framework 內建 `Image.network(url)` —— 不做 disk cache，每次重新下載；無 placeholder / error builder 統一行為
- 第三方 `cached_network_image` —— 提供 disk + memory cache、placeholder / errorWidget hooks

實務上觀察到：

1. **散落的 `Image.network` 呼叫** —— 同一張圖在不同 page 重複下載，浪費頻寬、torch UX
2. **行為不一致** —— 各 feature 自己刻 placeholder / error fallback，視覺與互動行為各異
3. **無法集中升級** —— 想全 app 改 cache 策略 / 改 placeholder / 換 image library，找不到單一改動點
4. **Cache 一致性破口** —— 即使導入 `cached_network_image`，只要還有任何一處 `Image.network`，那張 URL 就繞過 cache，等於白做

## Decision

**1. 套件選擇：使用 `cached_network_image` 作為唯一網路圖片載入套件。**

**2. 統一入口：所有網路圖片必須走下游 repo 的 `lib/ui_kit/components/images/ui_network_image.dart` widget，**禁止任何 feature 直接 import `cached_network_image` 或 framework `Image.network`。**

包括：

- 新寫的 widget 一律用 `UiNetworkImage`
- placeholder / errorWidget / fit / cache 行為由 `UiNetworkImage` 內部統一收斂
- 要新增行為（例如 fade-in、blur hash placeholder）→ 改 `UiNetworkImage`，不在 caller 端疊

## Consequences

### Positive

- 全 app cache 命中率一致，省頻寬、加速二次載入
- Placeholder / error 視覺體驗統一
- 想換 image library / 加觀測（image load metric）只改一個檔
- Capabilities index 有單一條目可被搜尋與重用

### Negative / Trade-offs

- 多一層 widget 包裝成本（一次性，~30 行）
- 需要建立並維護 `UiNetworkImage` API（`url`、`fit`、`width`、`height`、`placeholder` override 等）
- 既有 `Image.network` 呼叫點需逐步遷移

## Alternatives Considered

### 直接讓各 feature 使用 `cached_network_image`，不做包裝

**為什麼不選**：失去單一改動點，未來想加 placeholder / 換 library / 加觀測都要全 app grep + 改。也無法強制風格一致。

### 繼續用 framework `Image.network`，僅用 HTTP cache header

**為什麼不選**：HTTP cache 行為依平台 / OS / dio 設定差異大；無 disk persistence 保證；無 placeholder / error 控制。

### 自己刻 cache layer（包 dio 下載 → file system → 餵給 `Image.memory`）

**為什麼不選**：重造輪子。`cached_network_image` 已成熟、社群活躍、與 Flutter widget tree lifecycle 整合良好。

### 用 `extended_image` / `octo_image` 等其他 image library

**為什麼不選**：`extended_image` 功能過廣（含 editor / crop），體積與依賴成本高；`octo_image` 偏 builder-only 工具，仍需另外搭 cache。`cached_network_image` 在「網路圖片 + cache」這個窄用例最直接。

## Related

- 規範：[`packages.md`](../packages.md#媒體--圖片)
- 下游實作位置：`lib/ui_kit/components/images/ui_network_image.dart`（capabilities index 條目）
