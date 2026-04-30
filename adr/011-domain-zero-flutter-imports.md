# ADR-011: Domain Layer Zero Flutter Imports

## Status

Accepted | Date: 2024-06-01 | Supersedes: -

## Context

Clean Architecture 的核心原則：**inner layer 不依賴 outer layer**。Flutter 屬於 outer layer（UI framework），Domain 屬於 inner layer（業務邏輯）。Domain 引入 Flutter 直接違反此原則。

實務上 Domain 引入 Flutter 帶來四個具體問題：

### 1. 無法在 pure Dart 環境跑單元測試

```dart
// Domain 引入 Flutter
import 'package:flutter/material.dart';

class LoginUseCase {
  // ...
}
```

- 跑 `dart test` 時，Flutter binding 沒初始化會 crash
- 必須改跑 `flutter test`，啟動 Flutter binding 慢、佔資源
- pure Dart 測試 1 秒跑 100 個；Flutter binding 測試 5 秒跑 100 個

### 2. Flutter API 變動衝擊業務 code

- Flutter 升級偶有 API 改名 / 棄用
- 純 Dart Domain 不受影響；引入 Flutter 的 Domain 要跟 Flutter 升級綁
- 業務邏輯穩定 5 年，因為 import 一個 Flutter widget 而被迫每年重構

### 3. 違反依賴方向

```text
Presentation ────► Domain ◄──── Data
                      ▼
                   Flutter ?    ❌
```

Domain 同時被 Presentation 依賴、又依賴 Flutter（Presentation 也依賴 Flutter），出現循環依賴的概念設計（雖然編譯通過，但設計上錯了）。

### 4. 容易失控

- 一旦 Domain 容許 import Flutter，邊界判斷主觀
- 「我只是用 `Color` / `EdgeInsets` 一下」→ 漸漸全是 Flutter
- 嚴格規則簡單明確，鬆散規則無止境

### 容許的例外

```dart
// ✅ 允許的 Domain import
import 'dart:async';                                            // Dart 標準庫
import 'dart:convert';
import 'package:fpdart/fpdart.dart';                           // FP 工具，domain 的一部分
import 'package:freezed_annotation/freezed_annotation.dart';   // codegen annotation
```

`fpdart` 是例外因為它不是 framework，是 Dart 語言層的 FP 工具集，跟 Flutter 框架本質不同。

## Decision

**Domain 層所有檔案 zero Flutter import。允許的 import 限：**

- `dart:` 標準庫
- `package:fpdart/fpdart.dart`（`Either`、`Option`、`Unit`）
- `package:freezed_annotation/freezed_annotation.dart`（codegen）
- 同 feature 或其他 feature 的 Domain entity / interface

**禁止 import：**

- `package:flutter/...`
- `dart:ui`
- `package:dio/...`、`package:http/...`、其他 network 套件
- 任何 UI / persistence / platform 套件
- 同 feature 的 Data 層

```dart
// ✅
import 'dart:async';
import 'package:fpdart/fpdart.dart';
import 'package:freezed_annotation/freezed_annotation.dart';
import '../entities/user.dart';
import '../../../auth/domain/repositories/auth_repository.dart';

// ❌
import 'package:flutter/material.dart';
import 'dart:ui';
import 'package:dio/dio.dart';
import '../../data/datasources/remote/auth_remote_datasource.dart';
```

## Consequences

### Positive

- Domain 可在 pure Dart 環境跑單元測試（執行快、無 Flutter binding 啟動）
- Flutter API 變化不衝擊業務邏輯
- 依賴方向清晰、符合 Clean Architecture
- 邊界明確，不會慢慢失控

### Negative / Trade-offs

- 偶爾需要 abstract 一個介面（`Clock`、`Logger`、`PlatformService` 等）以避免在 Domain 直接用 SDK API
- 對熟悉 vanilla Flutter 寫法的 dev 需要 onboarding

## Alternatives Considered

### 容許 utility-level Flutter import（譬如 `Color`、`Locale`）

**為什麼不選**：邊界一旦鬆動會擴大；嚴格規則容易執行。

### Domain 容許 import 某些「核心 Flutter」（譬如 `dart:ui`）

**為什麼不選**：`dart:ui` 也是 Flutter 框架的一部分；同樣會在 pure Dart 環境跑不起來。

### 不規定（讓 dev 自主判斷）

**為什麼不選**：「業務邏輯」的判斷主觀；team 內不同 dev 可能不同邊界；統一規則 > 各自 judgment。

## Related

- 規範：[`architecture/overview.md`](../architecture/overview.md)、[`architecture/domain-layer.md`](../architecture/domain-layer.md)
- 相關 ADR：
  - [ADR-012: Features Never Cross-Import](./012-features-no-cross-import.md)（feature 邊界，跟 layer 邊界類似的紀律）
  - [ADR-013: App Entry / Shell / Bootstrap Layering](./013-app-entry-shell-bootstrap-layering.md)（app 啟動三層，責任分層的姊妹）
