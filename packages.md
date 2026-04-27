# 套件白名單

> **TL;DR**：本表列出所有獲准使用的 Flutter / Dart 套件。**不在表內的套件**需走「套件評估流程」取得批准才能加入專案。

---

## 不可妥協

- ✅ 只用本表內的套件
- ❌ 不引入未經評估的新套件 —— 即使「只是個小工具」也要走流程
- ❌ 不為了單一場景引入 heavy 套件，先看是否能用 Dart 標準庫或現有白名單套件實作

---

## 套件白名單

### 狀態管理

| 套件 | 用途 | 備註 |
|---|---|---|
| `flutter_bloc` | BLoC 狀態管理 | 唯一允許的狀態管理套件（→ [ADR-001](./adr/001-bloc-not-cubit.md)） |
| `bloc_test` | BLoC 測試 | 配對 `flutter_bloc` 使用 |

### 函數式 / 錯誤處理

| 套件 | 用途 | 備註 |
|---|---|---|
| `fpdart` | `Either`、`Option`、`Unit` 等 FP 工具 | 唯一允許的 FP 套件（→ [ADR-007](./adr/007-either-pattern-for-domain-results.md)） |

### Dependency Injection

| 套件 | 用途 | 備註 |
|---|---|---|
| `get_it` | Service locator | DI runtime |
| `injectable` | DI annotation + codegen | 配對 `get_it` 使用 |
| `injectable_generator` | injectable 的 codegen runner | dev_dependency |

### 網路

| 套件 | 用途 | 備註 |
|---|---|---|
| `dio` | HTTP client | 唯一允許的 HTTP 套件 |

### 導航

| 套件 | 用途 | 備註 |
|---|---|---|
| `go_router` | 路由 | 唯一允許的導航套件 |

### Code Generation

| 套件 | 用途 | 備註 |
|---|---|---|
| `freezed` | Immutable class / sealed class codegen | Entity、State 必用 |
| `freezed_annotation` | freezed annotation | runtime dependency |
| `json_serializable` | JSON 序列化 codegen | DTO 必用 |
| `json_annotation` | json_serializable annotation | runtime dependency |
| `slang` | i18n 字串 codegen | 用於使用者可見字串 |
| `slang_flutter` | slang Flutter integration | 配對 `slang` |
| `flutter_gen_runner` | Asset 路徑 codegen | dev_dependency |
| `build_runner` | Codegen orchestrator | dev_dependency |

### 本地儲存

| 套件 | 用途 | 備註 |
|---|---|---|
| `shared_preferences` | 簡單 key-value 儲存 | 不敏感資料 |
| `flutter_secure_storage` | 加密 key-value 儲存 | token、敏感資料 |
| `hive` | 結構化本地儲存 | 大量結構化資料才用 |

### 測試

| 套件 | 用途 | 備註 |
|---|---|---|
| `mocktail` | Mock library | 唯一允許的 mock 套件（不用 `mockito`） |
| `flutter_test` | Flutter 測試框架 | SDK 內建 |

### Logging / Telemetry

| 套件 | 用途 | 備註 |
|---|---|---|
| `logger` | App-level logger | 取代 `print()` / `debugPrint()` |

---

## 加新套件的流程

1. 在 GitHub Issue 開「Package proposal: `<package_name>`」
2. 填寫評估資訊：
   - **解決什麼問題**：當前用既有套件 / 標準庫做這件事為什麼不夠
   - **替代方案**：考慮過哪些其他套件、為什麼不選
   - **維護健康度**：套件最近一次更新、open issue 數、star、author
   - **License**：是否與本 repo 相容
   - **加入後對 bundle size 影響**
3. 團隊 review
4. 通過後加入本檔案 + 開 ADR（如果是有立場性的選擇）

---

## 排除清單（明確不用）

| 套件 | 不用的原因 |
|---|---|
| `provider` | 跟 `flutter_bloc` 重複，造成多套狀態管理混用 |
| `riverpod` | 同上；本規範以 BLoC 為唯一狀態管理 |
| `getx` | Magic 過多、testability 差、社群評價不一 |
| `mockito` | 用 `mocktail`；mockito 需要 codegen，相對麻煩 |
| `dartz` | 用 `fpdart`；fpdart 是 dartz 的後續維護版本 |

---

## Related

- [`adr/`](./adr/) — 套件選擇的 rationale
- [`patterns/code-generation.md`](./patterns/code-generation.md) — `freezed` / `injectable` / `slang` / `flutter_gen` 用法規範
