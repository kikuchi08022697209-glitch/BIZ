# 製造業ナレッジデータ管理 スキーマ仕様書

**スキーマID:** `https://example.org/mfg-knowledge/schema/1.1.0/mfg-knowledge-schema.json`
**バージョン:** 1.1.0
**策定日:** 2026-06-13
**準拠標準:** JSON Schema Draft 2020-12 / RO-Crate 1.1 / schema.org

---

## 目次

1. [スキーマ全体構造](#1-スキーマ全体構造)
2. [共通型定義（`$defs`）](#2-共通型定義defs)
3. [エンティティ型リファレンス](#3-エンティティ型リファレンス)
   - 3.1 [Standard（標準規格）](#31-standard標準規格)
   - 3.0 [DesignItem（設計事項）](#30-designitem設計事項)
   - 3.2 [ProductSpec（製品仕様書）](#32-productspec製品仕様書)
   - 3.3 [Requirement（要求事項）](#33-requirement要求事項)
   - 3.4 [BillOfMaterials（部品表）](#34-billofmaterials部品表)
   - 3.5 [BomEntry（部品エントリ）](#35-bomentry部品エントリ)
   - 3.6 [CadModel（3DCADモデル）](#36-cadmodel3dcadモデル)
   - 3.7 [TestReport（試験成績書）](#37-testreport試験成績書)
   - 3.8 [InspectionPlan（検査計画）](#38-inspectionplan検査計画)
4. [PMIアノテーション型リファレンス](#4-pmiアノテーション型リファレンス)
   - 4.1 [Datum（データム）](#41-datumデータム)
   - 4.2 [DimensionAnnotation（寸法値）](#42-dimensionannotation寸法値)
   - 4.3 [GeometricTolerance（幾何公差）](#43-geometrictolerance幾何公差)
   - 4.4 [SurfaceFinish（表面性状）](#44-surfacefinish表面性状)
   - 4.5 [GeneralNote（一般注記）](#45-generalnote一般注記)
   - 4.6 [ProcessNote（工程・検査指示注記）](#46-processnote工程検査指示注記)
   - 4.7 [WeldAnnotation（溶接記号）](#47-weldannotation溶接記号)
   - 4.8 [MaterialAnnotation（材料・熱処理指示）](#48-materialannotation材料熱処理指示)
5. [列挙値リファレンス](#5-列挙値リファレンス)
6. [バリデーションルール](#6-バリデーションルール)
7. [スキーマ拡張ガイド](#7-スキーマ拡張ガイド)

---

## 1. スキーマ全体構造

### 1.1 トップレベル

`ro-crate-metadata.json` はトップレベルに `@context` と `@graph` の2フィールドのみを持つ。

```
ro-crate-metadata.json
├── @context  [array, required]  ← JSON-LDコンテキスト（3要素固定）
└── @graph    [array, required]  ← 全エンティティの配列
      ├── [0] RoCrateDescriptor  ← RO-Crate記述子（固定）
      ├── [1] RootDataset        ← ルートデータセット（パッケージ）
      └── [2..] 各エンティティ   ← Standard / ProductSpec / BOM / CadModel / TestReport 等
```

### 1.2 `@context` の必須3要素

| インデックス | 値 | 役割 |
|---|---|---|
| 0 | `"https://w3id.org/ro/crate/1.1/context"` | RO-Crate 標準コンテキスト |
| 1 | `"https://schema.org"` | schema.org 汎用語彙 |
| 2 | `{ "mfg": "...", ... }` | 製造業拡張語彙マッピング |

### 1.3 `@graph` の先頭2要素

**RoCrateDescriptor（インデックス0、固定）**

```json
{
  "@type": "CreativeWork",
  "@id": "ro-crate-metadata.json",
  "conformsTo": {"@id": "https://w3id.org/ro/crate/1.1"},
  "about": {"@id": "./"}
}
```

**RootDataset（インデックス1）**

```json
{
  "@type": "Dataset",
  "@id": "./",
  "identifier": "PKG-{製品コード}-{版}",
  "name": "...",
  "dateCreated": "YYYY-MM-DD",
  "hasPart": [{"@id": "#std-xxx"}, ...]
}
```

---

## 2. 共通型定義（`$defs`）

| 型名 | 説明 | 使用箇所 |
|---|---|---|
| `IdRef` | `{"@id": "string"}` 形式の参照 | リンクフィールド全般 |
| `IsoDateTime` | ISO 8601 日時文字列（日付のみも可） | `dateCreated`, `dateModified` 等 |
| `Revision` | 版数文字列（1.0 / A / Rev.3 等） | 各エンティティの `mfg:revision` |
| `DocumentStatus` | ドキュメントステータス列挙 | 各エンティティの `mfg:status` |
| `DesignPhase` | 設計フェーズ列挙 | `DesignItem.mfg:designPhase` |
| `DesignItemStatus` | 設計事項ステータス列挙 | `DesignItem.mfg:status` |
| `Priority` | 要求優先度列挙 | `Requirement.mfg:priority` |
| `CtqLevel` | 品質特性重要度列挙 | PMI全般の `mfg:ctqLevel` |
| `ParametricRole` | パラメトリック役割列挙 | `DimensionAnnotation.mfg:parametricRole` |
| `Verdict` | 試験合否列挙 | `TestReport.mfg:verdict` |
| `MeasurementResult` | 試験測定値1件 | `TestReport.mfg:measurements[]` |
| `ChangeHistoryEntry` | 変更履歴1件 | `DimensionAnnotation.mfg:changeHistory[]` |
| `NoteItem` | 注記1項目 | `GeneralNote.mfg:noteItems[]` |
| `InspectionItem` | 検査計画1項目 | `InspectionPlan.mfg:inspectionItems[]` |
| `AssociatedMedia` | 関連メディア（Creoトレイル等） | `CadModel.schema:associatedMedia[]` |

---

## 3. エンティティ型リファレンス

### 3.0 DesignItem（設計事項）

**`@type`:** `"mfg:DesignItem"`
**`@id` パターン:** `#design-item-{連番}` （例: `#design-item-001`）

設計検討中の断片ナレッジを表す最小単位のエンティティ。
親文書へのリンク1本から始まり、詳細設計フェーズで CadModel や BomEntry に昇格する。

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `name` | string | ✓ | 設計事項名（例: `"フレーム材質選定"`） |
| `description` | string | — | 検討内容・根拠 |
| `mfg:designPhase` | DesignPhase | ✓ | 設計フェーズ（下記参照） |
| `mfg:status` | DesignItemStatus | ✓ | `under_study` / `decided` / `superseded` |
| `dcterms:isPartOf` | IdRef | ✓ | 所属する親文書（ProductSpec 等）`@id` |
| `prov:wasDerivedFrom` | IdRef | — | 派生元の要求事項・上位設計事項 `@id` |
| `mfg:decidedBy` | IdRef | — | 決定者（Person `@id`） |
| `mfg:decidedAt` | IsoDateTime | — | 確定日時 |
| `prov:wasRevisionOf` | IdRef | — | 旧版設計事項 `@id`（改訂時に設定） |
| `mfg:promotedTo` | IdRef | — | 昇格先エンティティ（CadModel・BomEntry 等）`@id` |

**`mfg:designPhase` 列挙値（DesignPhase）:**

| 値 | フェーズ |
|---|---|
| `requirements` | 要件定義 |
| `basic_design` | 基本設計 |
| `detail_design` | 詳細設計 |
| `cad_modeling` | CAD作成 |
| `verification` | 試験・評価 |
| `production` | 量産移行 |

**`mfg:status` 列挙値（DesignItemStatus）—一方向遷移のみ許可:**

```
under_study  →  decided  →  superseded
（検討中）      （確定）     （上位版に置換）
```

**AssociatedMedia（`CadModel.schema:associatedMedia[]` の要素）:**

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `@type` | const | ✓ | `"MediaObject"` 固定 |
| `name` | string | — | メディア名称 |
| `encodingFormat` | string | ✓ | MIMEタイプ（例: `"text/plain"`） |
| `contentUrl` | string | ✓ | ファイルパス |
| `mfg:mediaRole` | enum | — | `creo_trail` / `cad_native` / `fem_result` / `test_data` / `photo` |

---

### 3.1 Standard（標準規格）

**`@type`:** `"mfg:Standard"`
**`@id` パターン:** `#std-{規格番号を小文字ハイフン}` （例: `#std-jis-e4054`）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `identifier` | string | ✓ | 規格番号（例: `JIS E 4054`） |
| `name` | string | ✓ | 規格名称 |
| `version` | string | ✓ | 制定・改訂年（例: `"2016"`） |
| `url` | string（URI） | — | 規格の公式URL |
| `description` | string | — | 規格の概要説明 |
| `mfg:issuer` | string | ✓ | 発行機関名 |
| `mfg:status` | enum | ✓ | `active` / `withdrawn` / `under_revision` |
| `mfg:applicableTo` | IdRef[] | — | 適用先仕様書 `@id` のリスト |

---

### 3.2 ProductSpec（製品仕様書）

**`@type`:** `"mfg:ProductSpec"`
**`@id` パターン:** `#spec-{製品コード}-v{版}` （例: `#spec-rac35-v2`）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `identifier` | string | ✓ | 仕様書番号 |
| `name` | string | ✓ | 仕様書名称 |
| `mfg:revision` | Revision | ✓ | 版数 |
| `mfg:status` | DocumentStatus | ✓ | ステータス |
| `dateModified` | IsoDateTime | — | 最終更新日時 |
| `author` | IdRef | — | 作成者（Person `@id`） |
| `conformsTo` | IdRef[] | ✓ | 準拠する標準規格（最低1件） |
| `mfg:requirements` | Requirement[] | ✓ | 要求事項の配列（最低1件） |
| `mfg:linkedBom` | IdRef | — | 紐づく BOM |
| `mfg:linkedTestReport` | IdRef[] | — | 紐づく試験成績書 |
| `mfg:fileRef` | string | — | 仕様書 PDF パス |

---

### 3.3 Requirement（要求事項）

ProductSpec 内に埋め込む（独立エンティティではない）。

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `identifier` | string | ✓ | `REQ-NNN` 形式（例: `REQ-001`） |
| `name` | string | ✓ | 要求事項名 |
| `description` | string | ✓ | 要求内容（合否基準を含む） |
| `mfg:category` | enum | ✓ | 種別（下記列挙値参照） |
| `mfg:priority` | Priority | ✓ | `shall` / `should` / `may` |
| `mfg:standardRef` | IdRef | — | 根拠となる標準規格 |
| `mfg:standardClause` | string | — | 規格条項番号（例: `"6.3.1"`） |
| `mfg:verifiedBy` | IdRef | — | 検証に使用する試験成績書 |
| `mfg:acceptanceCriteria` | string | — | 合否基準の補足 |

**`mfg:category` 列挙値:**

| 値 | 意味 |
|---|---|
| `performance` | 性能・機能要求 |
| `safety` | 安全要求 |
| `reliability` | 信頼性・耐久性要求 |
| `comfort` | 快適性要求（騒音・振動など） |
| `mechanical` | 機械的要求（寸法・質量） |
| `electrical` | 電気的要求（絶縁・EMC） |
| `environmental` | 環境要求（RoHS・REACH等） |

---

### 3.4 BillOfMaterials（部品表）

**`@type`:** `"mfg:BillOfMaterials"`
**`@id` パターン:** `#bom-{製品コード}-v{版}` （例: `#bom-rac35-v2`）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `identifier` | string | ✓ | BOM番号 |
| `name` | string | ✓ | BOM名称 |
| `mfg:revision` | Revision | ✓ | 版数 |
| `mfg:status` | DocumentStatus | ✓ | ステータス |
| `dateModified` | IsoDateTime | — | 最終更新日時 |
| `derivedFrom` | IdRef | ✓ | 派生元 ProductSpec `@id` |
| `mfg:totalMassKg` | number（≥0） | — | 総質量（kg） |
| `mfg:entries` | BomEntry[] | ✓ | 部品エントリの配列（最低1件） |

---

### 3.5 BomEntry（部品エントリ）

BillOfMaterials 内の `mfg:entries` 配列に埋め込む。

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `identifier` | string | ✓ | 部品エントリID（例: `PRT-101`） |
| `name` | string | ✓ | 部品名称 |
| `mfg:partNumber` | string | ✓ | 部品番号 |
| `mfg:quantity` | number（≥0） | ✓ | 員数 |
| `mfg:unit` | string | ✓ | 単位（個 / 式 / kg 等） |
| `mfg:material` | string | — | 材質 |
| `mfg:massKg` | number（≥0） | — | 単品質量（kg） |
| `mfg:linkedCadModel` | IdRef \| null | — | 対応 CAD モデル（null = 調達品外形なし） |
| `mfg:procuredFrom` | string | — | 外部調達先（調達品の場合） |
| `mfg:governedBy` | IdRef | — | 部品が準拠する標準規格 |

---

### 3.6 CadModel（3DCADモデル）

**`@type`:** `"mfg:CadModel"`
**`@id` パターン:** `#cad-{製品コード}-{部品略称}` （例: `#cad-rac35-frame`）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `identifier` | string | ✓ | CADモデルID |
| `name` | string | ✓ | モデル名称 |
| `mfg:revision` | Revision | ✓ | 版数 |
| `mfg:status` | DocumentStatus | ✓ | ステータス |
| `mfg:format` | enum | ✓ | ファイルフォーマット（下記列挙値参照） |
| `mfg:fileRef` | string | ✓ | STEP等中間ファイルパス |
| `mfg:nativeFileRef` | string | — | ネイティブ形式ファイルパス |
| `dateModified` | IsoDateTime | — | 最終更新日時 |
| `mfg:mbdCompliant` | boolean | — | MBD準拠フラグ（default: false） |
| `mfg:mbdStandard` | string | — | 準拠 MBD 規格（例: `ISO 16792:2015`） |
| `mfg:derivedFrom` | IdRef | — | 派生元 ProductSpec |
| `mfg:referencedInBom` | IdRef \| IdRef[] | — | 参照する BomEntry |
| `mfg:validatedBy` | IdRef[] | — | 検証に使用した試験成績書 |
| `mfg:hasPmi` | PmiAnnotation[] | — | PMI アノテーションの配列 |
| `mfg:inspectionPlan` | IdRef | — | 生成済み検査計画 |
| `mfg:designNote` | string | — | 設計上の注意事項・補足 |

**`mfg:format` 列挙値:** `STEP AP203` / `STEP AP214` / `STEP AP242` / `JT` / `3MF` / `IGES`

---

### 3.7 TestReport（試験成績書）

**`@type`:** `"mfg:TestReport"`
**`@id` パターン:** `#test-{試験種別}-{年}` （例: `#test-vibration-2026`）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `identifier` | string | ✓ | 試験成績書番号 |
| `name` | string | ✓ | 試験成績書名称 |
| `mfg:revision` | Revision | ✓ | 版数 |
| `mfg:status` | DocumentStatus | ✓ | ステータス |
| `dateCreated` | IsoDateTime | — | 試験実施日 |
| `author` | IdRef | — | 試験担当者 |
| `mfg:verifies` | IdRef[] | ✓ | 検証対象の要求事項（最低1件） |
| `mfg:conformsTo` | IdRef[] | ✓ | 試験が準拠する標準規格（最低1件） |
| `mfg:testedObject` | IdRef[] | — | 試験対象の CAD / BOM |
| `mfg:derivedFrom` | IdRef | ✓ | 試験の根拠となる仕様書 |
| `mfg:testConditions` | object | — | 試験条件（フリーフォーム） |
| `mfg:verdict` | Verdict | ✓ | `PASS` / `FAIL` / `CONDITIONAL` |
| `mfg:conditionalNotes` | string | — | CONDITIONAL 時の条件詳細 |
| `mfg:fileRef` | string | ✓ | 試験成績書 PDF パス |
| `mfg:measurements` | MeasurementResult[] | — | 測定結果の配列 |

**MeasurementResult フィールド:**

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `parameter` | string | ✓ | 測定項目名 |
| `value` | any | ✓ | 測定値（数値または文字列） |
| `unit` | string | — | 単位 |
| `limit` | string | ✓ | 合否基準 |
| `result` | enum | ✓ | `PASS` / `FAIL` / `—` |

---

### 3.8 InspectionPlan（検査計画）

**`@type`:** `"mfg:InspectionPlan"`
**`@id` パターン:** `#insp-plan-{部品略称}` （例: `#insp-plan-frame`）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `identifier` | string | ✓ | 検査計画番号 |
| `name` | string | ✓ | 検査計画名称 |
| `mfg:derivedFromCad` | IdRef | ✓ | 元となる CAD モデル |
| `mfg:derivedFromSpec` | IdRef | — | 元となる ProductSpec |
| `mfg:revision` | Revision | — | 版数 |
| `dateModified` | IsoDateTime | — | 最終更新日時 |
| `mfg:inspectionItems` | InspectionItem[] | ✓ | 検査項目の配列（最低1件） |

**InspectionItem フィールド:**

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `sequence` | integer（≥1） | ✓ | 検査順序番号 |
| `pmiRef` | IdRef | ✓ | 対応する PMI エンティティ |
| `characteristic` | string | ✓ | 検査特性名 |
| `ctqLevel` | CtqLevel | ✓ | 品質特性重要度 |
| `method` | string | ✓ | 検査方法 |
| `frequency` | string | ✓ | 検査頻度（全数 / AQL 1.0 等） |
| `acceptanceCriteria` | string | ✓ | 合否基準 |
| `recordRequired` | boolean | — | 記録要否 |

---

## 4. PMIアノテーション型リファレンス

PMI は `CadModel.mfg:hasPmi` 配列に埋め込む。
各エンティティは `mfg:pmiType` フィールドで種別を宣言し、
`anyOf` ディスパッチにより該当サブスキーマで検証される。

### 4.1 Datum（データム）

**`@type`:** `["PmiAnnotation", "mfg:Datum"]`
**`@id` パターン:** `#pmi-datum-{ラベル}` （例: `#pmi-datum-A`）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `mfg:pmiType` | const | ✓ | `"datum"` 固定 |
| `mfg:datumLabel` | string `[A-Z]` | ✓ | データム識別ラベル（A / B / C） |
| `mfg:datumType` | enum | — | `plane` / `axis` / `point` |
| `mfg:description` | string | — | 基準面・軸・点の意味説明 |

---

### 4.2 DimensionAnnotation（寸法値）

**`@type`:** `["PmiAnnotation", "mfg:DimensionAnnotation"]`
**`@id` パターン:** `#pmi-dim-{部品略称}-{連番}` （例: `#pmi-dim-frame-w`）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `mfg:pmiType` | enum | ✓ | `linear_dimension` / `angular_dimension` / `reference_dimension` |
| `mfg:nominal` | number | ✓ | 名目値 |
| `mfg:upperTol` | number | — | 上偏差（正値または0） |
| `mfg:lowerTol` | number | — | 下偏差（負値または0） |
| `mfg:unit` | string | ✓ | 単位（mm / deg / inch 等） |
| `mfg:toleranceClass` | string | — | IT グレード（例: `IT6`） |
| `mfg:fitType` | string | — | 嵌め合い種別（例: `H7/h6`） |
| `mfg:ctqLevel` | CtqLevel | — | 品質特性重要度 |
| `mfg:inspectionMethod` | string | — | 検査方法 |
| `mfg:inspectionFrequency` | string | — | 検査頻度 |
| `mfg:parametricRole` | ParametricRole | — | パラメトリック役割 |
| `mfg:parametricName` | string | — | CAD パラメータ変数名 |
| `mfg:parametricFormula` | string | — | `driven` 寸法の計算式 |
| `mfg:drives` | IdRef[] | — | 駆動する子パラメータ `@id` のリスト |
| `mfg:drivenBy` | IdRef | — | 駆動する親パラメータ `@id` |
| `mfg:changeImpact` | string | — | 変更時影響の説明（承認プロセス等） |
| `mfg:linkedRequirement` | IdRef | — | 根拠となる要求事項 |
| `mfg:verifiedByPmi` | IdRef | — | 検証に使用した試験成績書 |
| `mfg:changeHistory` | ChangeHistoryEntry[] | — | 変更履歴 |

**ChangeHistoryEntry フィールド:**

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `revision` | Revision | ✓ | 変更時の版数 |
| `nominal` | number | — | 変更前の名目値 |
| `upperTol` | number | — | 変更前の上偏差 |
| `lowerTol` | number | — | 変更前の下偏差 |
| `changedAt` | IsoDateTime | ✓ | 変更日時 |
| `changedBy` | string | — | 変更者 |
| `reason` | string | ✓ | 変更理由 |

---

### 4.3 GeometricTolerance（幾何公差）

**`@type`:** `["PmiAnnotation", "mfg:GeometricTolerance"]`
**`@id` パターン:** `#pmi-gt-{連番}` （例: `#pmi-gt-001`）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `mfg:pmiType` | const | ✓ | `"geometric_tolerance"` 固定 |
| `mfg:characteristic` | enum | ✓ | 公差の種類（下記列挙値参照） |
| `mfg:symbol` | string | ✓ | 公差記号（⏥ ⊕ ⊥ 等） |
| `mfg:toleranceZone` | object | ✓ | 公差域（`value`, `unit`, `shape`） |
| `mfg:datumRef` | IdRef \| IdRef[] \| null | — | 参照データム |
| `mfg:ctqLevel` | CtqLevel | — | 品質特性重要度 |
| `mfg:inspectionMethod` | string | — | 検査方法 |
| `mfg:inspectionFrequency` | string | — | 検査頻度 |
| `mfg:linkedRequirement` | IdRef | — | 根拠となる要求事項 |
| `mfg:verifiedByPmi` | IdRef | — | 検証に使用した試験成績書 |
| `mfg:relatedBomEntry` | IdRef | — | 関連する BomEntry |

**`mfg:characteristic` 列挙値:**

| 記号 | 和名 | 英名 |
|---|---|---|
| ー | 真直度 | Straightness |
| ⏥ | 平面度 | Flatness |
| ○ | 真円度 | Circularity |
| ⌭ | 円筒度 | Cylindricity |
| ⌒ | 線の輪郭度 | Profile_Line |
| ⌓ | 面の輪郭度 | Profile_Surface |
| ∥ | 平行度 | Parallelism |
| ⊥ | 直角度 | Perpendicularity |
| ∠ | 傾斜度 | Angularity |
| ⊕ | 位置度 | Position |
| ◎ | 同軸度 | Concentricity |
| ═ | 対称度 | Symmetry |
| ↗ | 円周振れ | Circular_Runout |
| ⟿ | 全振れ | Total_Runout |

---

### 4.4 SurfaceFinish（表面性状）

**`@type`:** `["PmiAnnotation", "mfg:SurfaceFinish"]`
**`@id` パターン:** `#pmi-sf-{連番}` （例: `#pmi-sf-001`）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `mfg:pmiType` | const | ✓ | `"surface_finish"` 固定 |
| `mfg:parameter` | enum | ✓ | `Ra` / `Rz` / `Rmax` / `Wa` / `Wz` |
| `mfg:value` | number（≥0） | ✓ | パラメータ値 |
| `mfg:unit` | string | ✓ | 単位（通常 `μm`） |
| `mfg:standard` | string | — | 準拠規格（例: `JIS B 0601:2013`） |
| `mfg:machiningMethod` | string | — | 加工方法（旋削・研削 等） |
| `mfg:machiningAllowance` | string | — | 加工代指示 |
| `mfg:treatment` | string | — | 表面処理（メッキ・塗装 等） |
| `mfg:ctqLevel` | CtqLevel | — | 品質特性重要度 |
| `mfg:inspectionMethod` | string | — | 検査方法 |
| `mfg:inspectionFrequency` | string | — | 検査頻度 |
| `mfg:linkedRequirement` | IdRef | — | 根拠となる要求事項 |

---

### 4.5 GeneralNote（一般注記）

**`@type`:** `["PmiAnnotation", "mfg:GeneralNote"]`
**`@id` パターン:** `#pmi-note-{連番}` （例: `#pmi-note-001`）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `mfg:pmiType` | const | ✓ | `"general_note"` 固定 |
| `mfg:scope` | enum | — | `drawing_wide` / `local` / `bom_wide` |
| `mfg:noteItems` | NoteItem[] | ✓ | 注記項目の配列（最低1件） |

**NoteItem フィールド:**

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `seq` | integer（≥1） | ✓ | 注記連番 |
| `text` | string | ✓ | 注記本文 |
| `mfg:applicableTo` | string | — | 適用範囲の説明 |

---

### 4.6 ProcessNote（工程・検査指示注記）

**`@type`:** `["PmiAnnotation", "mfg:ProcessNote"]`
**`@id` パターン:** `#pmi-procnote-{連番}` （例: `#pmi-procnote-001`）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `mfg:pmiType` | const | ✓ | `"process_note"` 固定 |
| `mfg:processType` | enum | ✓ | 工程種別（下記列挙値参照） |
| `mfg:applicableTo` | IdRef[] | — | 適用対象の BomEntry 等 |
| `mfg:noteText` | string | ✓ | 指示の本文 |
| `mfg:processParameters` | object | — | 工程パラメータ（nominal / tolerance / min / max / unit） |
| `mfg:ctqLevel` | CtqLevel | — | 品質特性重要度 |
| `mfg:inspectionMethod` | string | — | 確認方法 |
| `mfg:linkedRequirement` | IdRef | — | 根拠となる要求事項 |

**`mfg:processType` 列挙値:**
`heat_treatment` / `surface_treatment` / `welding` / `inspection` / `assembly` / `cleaning` / `packaging`

---

### 4.7 WeldAnnotation（溶接記号）

**`@type`:** `["PmiAnnotation", "mfg:WeldAnnotation"]`
**`@id` パターン:** `#pmi-weld-{連番}` （例: `#pmi-weld-001`）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `mfg:pmiType` | const | ✓ | `"weld_symbol"` 固定 |
| `mfg:weldType` | string | ✓ | 溶接種別（完全溶込み / 隅肉 等） |
| `mfg:jointType` | string | — | 継手種別（T継手 / 突合せ 等） |
| `mfg:throatSize` | object | — | のど厚（`value`, `unit`） |
| `mfg:weldLength` | string | — | 溶接長さ（連続 / 断続 / 長さ指定） |
| `mfg:standard` | string | — | 準拠規格（例: `JIS Z 3001`） |
| `mfg:inspectionMethod` | string | — | 検査方法（UT / MT / PT 等） |
| `mfg:inspectionCoverage` | string | — | 検査率（100% / 抜取 等） |
| `mfg:ctqLevel` | CtqLevel | — | 品質特性重要度 |
| `mfg:linkedRequirement` | IdRef | — | 根拠となる要求事項 |

---

### 4.8 MaterialAnnotation（材料・熱処理指示）

**`@type`:** `["PmiAnnotation", "mfg:MaterialAnnotation"]`
**`@id` パターン:** `#pmi-mat-{連番}` （例: `#pmi-mat-001`）

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `mfg:pmiType` | const | ✓ | `"material_callout"` 固定 |
| `mfg:material` | string | ✓ | 材質記号（例: `SUS304`） |
| `mfg:materialStandard` | string | — | 材料規格（例: `JIS G 4305:2021`） |
| `mfg:heatTreatment` | string | — | 熱処理条件 |
| `mfg:hardness` | string \| null | — | 硬さ要求（例: `HRC 58〜62`） |
| `mfg:certification` | string | — | 証明書要求（ミルシート等） |

---

## 5. 列挙値リファレンス

### DocumentStatus

| 値 | 意味 |
|---|---|
| `draft` | 作成中 |
| `under_review` | 審査中 |
| `released` | 発行済（量産・調達に使用可） |
| `obsolete` | 廃止（参照のみ可） |

### DesignItemStatus

| 値 | 意味 | 遷移 |
|---|---|---|
| `under_study` | 検討中 | → `decided` |
| `decided` | 確定 | → `superseded` |
| `superseded` | 上位版に置換済み（終端状態） | — |

### Priority

| 値 | 意味 | ISO/IEC準拠 |
|---|---|---|
| `shall` | 必須 | 適合に必要な要求 |
| `should` | 推奨 | 従うことが望ましい要求 |
| `may` | 任意 | 許容される選択肢 |

### CtqLevel

| 値 | 意味 | 検査頻度 |
|---|---|---|
| `CTQ` | Critical To Quality | 全数・記録必須 |
| `major` | 重要特性 | 抜取 AQL 1.0 |
| `minor` | 一般特性 | 抜取 AQL 2.5 |

### ParametricRole

| 値 | 意味 |
|---|---|
| `driving` | 他の寸法を駆動する親パラメータ |
| `driven` | 親パラメータから計算される子パラメータ |
| `reference` | 参照のみ（変更不可） |
| `fixed` | 規格・標準部品により固定 |

### Verdict

| 値 | 意味 |
|---|---|
| `PASS` | 合格 |
| `FAIL` | 不合格 |
| `CONDITIONAL` | 条件付き合格（`mfg:conditionalNotes` に条件を記載） |

---

## 6. バリデーションルール

### 6.1 必須チェック

| ルール | 説明 |
|---|---|
| V-01 | `@graph[0]` は `RoCrateDescriptor` でなければならない |
| V-02 | `@graph[1]` は `RootDataset` でなければならない |
| V-03 | `ProductSpec.conformsTo` は最低1件の `Standard @id` を含むこと |
| V-04 | `ProductSpec.mfg:requirements` は最低1件の `Requirement` を含むこと |
| V-05 | `TestReport.mfg:verifies` は最低1件の `Requirement @id` を含むこと |
| V-06 | `TestReport.mfg:conformsTo` は最低1件の `Standard @id` を含むこと |
| V-07 | `InspectionPlan.mfg:inspectionItems` は最低1件を含むこと |

### 6.2 IDフォーマットチェック

| フィールド | パターン |
|---|---|
| `Standard.@id` | `^#std-` |
| `ProductSpec.@id` | `^#spec-` |
| `BillOfMaterials.@id` | `^#bom-` |
| `BomEntry.@id` | `^#bom-entry-` |
| `CadModel.@id` | `^#cad-` |
| `TestReport.@id` | `^#test-` |
| `InspectionPlan.@id` | `^#insp-plan-` |
| `Requirement.identifier` | `^REQ-\d{3}` |
| `RootDataset.identifier` | `^PKG-` |
| `Person.@id` | `^#emp-` |
| `Organization.@id` | `^#dept-` |
| `Datum.mfg:datumLabel` | `^[A-Z]$`（1文字大文字） |

### 6.3 参照整合性チェック（実装推奨）

| チェック | 説明 |
|---|---|
| RC-01 | `RootDataset.hasPart` に記載された `@id` がグラフ内に存在すること |
| RC-02 | `ProductSpec.mfg:linkedBom` の `@id` がグラフ内の `BillOfMaterials` を参照していること |
| RC-03 | `Requirement.mfg:verifiedBy` の `@id` がグラフ内の `TestReport` を参照していること |
| RC-04 | `TestReport.mfg:verifies` の `@id` がグラフ内の `Requirement` を参照していること |
| RC-05 | `BomEntry.mfg:linkedCadModel` の `@id` がグラフ内の `CadModel` を参照していること |
| RC-06 | `DimensionAnnotation.mfg:drives` の各 `@id` がグラフ内の `DimensionAnnotation` を参照していること |
| RC-07 | `DimensionAnnotation.mfg:drivenBy` の `@id` の `mfg:drives` に逆リンクが存在すること |
| RC-08 | `CadModel.mfg:inspectionPlan` の `@id` がグラフ内の `InspectionPlan` を参照していること |
| RC-09 | `InspectionItem.pmiRef` の `@id` が `CadModel.mfg:hasPmi` 内に存在すること |

---

## 7. スキーマ拡張ガイド

### 7.1 新規エンティティ型の追加手順

1. `$defs` に新規型を定義する
2. `@graph.items.anyOf` に追加する
3. `@context` の `mfg:` マッピングに新語彙を追加する
4. スキーマの `version` をバンプする（マイナー追加 → マイナー版上げ）

```json
"$defs": {
  "NewEntityType": {
    "type": "object",
    "required": ["@type", "@id", "identifier", "name"],
    "properties": {
      "@type":      { "const": "mfg:NewEntityType" },
      "@id":        { "type": "string", "pattern": "^#new-" },
      "identifier": { "type": "string" },
      "name":       { "type": "string" }
    },
    "additionalProperties": false
  }
}
```

### 7.2 非互換変更の禁止事項

| 禁止操作 | 代替手段 |
|---|---|
| 必須フィールドの追加 | 任意フィールドとして追加し `default` を設定 |
| 既存フィールドの型変更 | 新フィールドを追加し旧フィールドを deprecated マーク |
| 列挙値の削除 | `obsolete_` プレフィックスを付けて残存 |
| `@id` パターンの変更 | 新パターンを `oneOf` で追加 |

### 7.3 バージョン管理方針

| 変更種別 | バージョンバンプ | 例 |
|---|---|---|
| 新規任意フィールド追加 | マイナー（1.0 → 1.1） | PMIに新フィールド追加 |
| 新規エンティティ型追加 | マイナー（1.0 → 1.1） | KaizenRecord型追加 |
| 必須フィールド追加・型変更 | メジャー（1.0 → 2.0） | — |
| バグ修正・説明修正 | パッチ（1.0.0 → 1.0.1） | パターン正規表現修正 |
