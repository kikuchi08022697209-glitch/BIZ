# 製造業ナレッジデータ管理手法

## RO-Crate思想に基づくJSON-LD型ドキュメントリンク管理

---

## 目次

1. [概要](#1-概要)
2. [設計思想](#2-設計思想)
3. [ドキュメント種別とエンティティ定義](#3-ドキュメント種別とエンティティ定義)
4. [エンティティ間リンク述語](#4-エンティティ間リンク述語)
5. [フォーマット仕様](#5-フォーマット仕様)
6. [PMI連携仕様](#6-pmi連携仕様)
7. [パッケージ構造](#7-パッケージ構造)
8. [適用事例：電車用空調機 RAC-35型](#8-適用事例電車用空調機-rac-35型)
9. [ユースケース別クエリパターン](#9-ユースケース別クエリパターン)
10. [拡張ガイドライン](#10-拡張ガイドライン)
11. [データ構築プロセス：段階的ナレッジ蓄積アプローチ](#11-データ構築プロセス段階的ナレッジ蓄積アプローチ)

---

## 1. 概要

本手法は、製造業における以下の5大ドキュメント種別を相互リンクし、
設計変更・規格改訂・品質不具合発生時の影響波及をグラフ構造で即時に追跡できる
ナレッジデータ管理フォーマットを定義する。

| ドキュメント種別 | 略称 |
|---|---|
| 標準規格 | Standard |
| 製品仕様書 | ProductSpec |
| 部品表 | BOM |
| 3D CADモデル（PMI含む） | CadModel |
| 開発試験成績書 | TestReport |

### 解決する課題

| 従来の課題 | 本手法による解決 |
|---|---|
| ドキュメント間の依存関係が担当者の頭の中だけにある | `@id` による明示的リンクでグラフ化 |
| 規格改訂時に何が影響するか即座にわからない | `conformsTo` リンクを逆引きして影響先を列挙 |
| 試験成績書がどの要求事項を検証しているか不明 | `verifies` リンクで要求事項と試験を直結 |
| CAD変更時に再試験が必要か判断できない | `validatedBy` リンクで影響試験を自動抽出 |
| PMI（公差・表面粗さ）と検査計画が別管理 | PMIエンティティを CAD から `hasPmi` でリンク |

---

## 2. 設計思想

### 2.1 RO-Crate（Research Object Crate）の転用

本手法は学術研究データ管理標準である [RO-Crate](https://www.researchobject.org/ro-crate/) の以下の原則を製造業に適用する。

| RO-Crate原則 | 製造業への読み替え |
|---|---|
| Research Object（論文＋データ＋作者） | Knowledge Object（ナレッジ＋証跡＋担当者） |
| JSON-LD + schema.org | JSON-LD + schema.org + 製造業拡張語彙（`mfg:` namespace） |
| FAIR原則（Findable / Accessible / Interoperable / Reusable） | 見つけられる・使える・横展開できる・再利用できる |
| Provenance（来歴）の追跡 | 変更履歴・担当者・承認者の完全記録 |
| Digital Object化 | 証跡ファイルごとメタデータで包んでZip配布 |

### 2.2 FAIR原則との対応

```
Findable    … identifier による一意なID体系（例: SPEC-RAC35, CAD-RAC35-FRM）
Accessible  … fileRef による証跡ファイルの直接参照
Interoperable … JSON-LD + schema.org による標準語彙での記述
Reusable    … derivedFrom・conformsTo による再利用元の明示
```

### 2.3 エンティティグラフの全体像

```
標準規格 (Standard)
    │
    │ conformsTo ↓
    │
    ├──→ 製品仕様 (ProductSpec)
    │         │
    │         ├── requirements[] ──→ 要求事項 (Requirement)
    │         │                           │ verifiedBy ↓
    │         │                           └──────────────→ 試験成績書 (TestReport)
    │         │
    │         ├── linkedBom ──→ 部品表 (BOM)
    │         │                     │
    │         │                     └── entries[] ──→ 部品エントリ (BomEntry)
    │         │                                             │
    │         │                                             └── linkedCadModel ──→ CADモデル
    │         │
    │         └── linkedTestReport ──→ 試験成績書 (TestReport)
    │
    │                                  試験成績書 (TestReport)
    │                                      │
    │                                      ├── verifies ──→ 要求事項
    │                                      ├── conformsTo ──→ 標準規格
    │                                      └── testedObject ──→ CADモデル / BOM
    │
    └── CADモデル (CadModel / MBD)
              │
              ├── hasPmi[] ──→ PMIアノテーション
              │                   ├── GeometricTolerance（幾何公差）
              │                   ├── SurfaceFinish（表面性状）
              │                   ├── Datum（データム）
              │                   ├── WeldSymbol（溶接記号）
              │                   └── MaterialCallout（材料指示）
              │
              └── inspectionPlan ──→ 検査計画 (InspectionPlan)
```

---

## 3. ドキュメント種別とエンティティ定義

### 3.1 Standard（標準規格）

製品が準拠すべき国内外規格・社内規格。

| フィールド | 型 | 説明 |
|---|---|---|
| `@id` | string | `#std-{規格番号}` 形式 |
| `identifier` | string | 規格番号（例: `JIS E 4054`） |
| `name` | string | 規格名称 |
| `version` | string | 制定・改訂年 |
| `mfg:issuer` | string | 発行機関 |
| `mfg:status` | enum | `active` / `withdrawn` / `under_revision` |
| `mfg:applicableTo` | array | 適用先仕様書 `@id` のリスト |

### 3.2 ProductSpec（製品仕様書）

製品が満たすべき要求事項の集合。設計のルートエンティティ。

| フィールド | 型 | 説明 |
|---|---|---|
| `conformsTo` | array | 準拠する標準規格 `@id` のリスト |
| `mfg:requirements` | array | 要求事項（Requirement）の埋め込みリスト |
| `mfg:linkedBom` | object | 紐づく BOM の `@id` |
| `mfg:linkedTestReport` | array | 紐づく試験成績書 `@id` のリスト |
| `mfg:revision` | string | 版数 |
| `mfg:status` | enum | `draft` / `released` / `obsolete` |

#### Requirement（要求事項）サブエンティティ

| フィールド | 型 | 説明 |
|---|---|---|
| `identifier` | string | 要求事項番号（例: `REQ-001`） |
| `mfg:category` | enum | `performance` / `safety` / `reliability` / `comfort` / `mechanical` |
| `mfg:priority` | enum | `shall`（必須） / `should`（推奨） / `may`（任意） |
| `mfg:standardRef` | object | 根拠となる標準規格 `@id` |
| `mfg:standardClause` | string | 規格の条項番号 |
| `mfg:verifiedBy` | object | 検証する試験成績書 `@id` |

### 3.3 BillOfMaterials（部品表）

製品を構成する部品・材料の階層リスト。

| フィールド | 型 | 説明 |
|---|---|---|
| `derivedFrom` | object | 派生元仕様書 `@id` |
| `mfg:totalMassKg` | number | 総質量（質量要求の検証に使用） |
| `mfg:entries` | array | BomEntry の配列 |

#### BomEntry（部品エントリ）サブエンティティ

| フィールド | 型 | 説明 |
|---|---|---|
| `mfg:partNumber` | string | 部品番号 |
| `mfg:quantity` | number | 員数 |
| `mfg:material` | string | 材質 |
| `mfg:massKg` | number | 単品質量 |
| `mfg:linkedCadModel` | object | 対応 CAD モデル `@id`（null = 調達品外形なし） |
| `mfg:procuredFrom` | string | 外部調達先（調達品の場合） |
| `mfg:governedBy` | object | 部品が準拠する標準規格 `@id` |

### 3.4 CadModel（3D CADモデル）

MBD（Model-Based Definition）に基づく 3D モデル。PMI を内包する。

| フィールド | 型 | 説明 |
|---|---|---|
| `mfg:format` | string | 中間フォーマット（例: `STEP AP242`） |
| `mfg:fileRef` | string | STEP ファイルパス |
| `mfg:nativeFileRef` | string | ネイティブ形式ファイルパス（.catpart / .sldprt 等） |
| `mfg:mbdCompliant` | boolean | MBD準拠フラグ |
| `mfg:mbdStandard` | string | 準拠する MBD 規格（例: `ISO 16792:2015`） |
| `mfg:hasPmi` | array | PMI アノテーションエンティティの配列 |
| `mfg:inspectionPlan` | object | 検査計画エンティティ `@id` |
| `mfg:validatedBy` | array | 検証に使用した試験成績書 `@id` のリスト |
| `mfg:referencedInBom` | object | BomEntry `@id` |

### 3.5 TestReport（開発試験成績書）

要求事項の適合性を実証した試験の記録。

| フィールド | 型 | 説明 |
|---|---|---|
| `mfg:verifies` | array | 検証対象の要求事項 `@id` のリスト |
| `mfg:conformsTo` | array | 試験が準拠する標準規格 `@id` のリスト |
| `mfg:testedObject` | array | 試験対象の CAD / BOM `@id` のリスト |
| `mfg:derivedFrom` | object | 試験の根拠となる仕様書 `@id` |
| `mfg:testConditions` | object | 試験条件（温度・圧力・周波数等） |
| `mfg:verdict` | enum | `PASS` / `FAIL` / `CONDITIONAL` |
| `mfg:measurements` | array | 測定結果の配列（parameter / value / unit / limit / result） |
| `mfg:fileRef` | string | 試験成績書 PDF ファイルパス |

---

## 4. エンティティ間リンク述語

### 4.1 主要リンク一覧

| 述語 | 始点 → 終点 | 意味 |
|---|---|---|
| `conformsTo` | Spec → Standard | 仕様書が準拠する規格 |
| `mfg:applicableTo` | Standard → Spec | 規格の適用先仕様書 |
| `derivedFrom` | BOM / TestReport → Spec | どの仕様書から派生したか |
| `mfg:verifies` | TestReport → Requirement | どの要求事項を検証しているか |
| `mfg:verifiedBy` | Requirement → TestReport | 要求事項を検証する試験（逆向き） |
| `mfg:validatedBy` | CadModel → TestReport | CAD モデルを検証した試験 |
| `mfg:linkedCadModel` | BomEntry → CadModel | 部品エントリと 3D モデルの紐付け |
| `mfg:referencedInBom` | CadModel → BomEntry | CAD モデルが参照される BOM エントリ |
| `mfg:governedBy` | BomEntry → Standard | 部品が準拠すべき規格 |
| `mfg:testedObject` | TestReport → CadModel / BOM | 試験の対象物 |
| `mfg:hasPmi` | CadModel → PmiAnnotation | CAD に埋め込まれた PMI |
| `mfg:linkedRequirement` | PmiAnnotation → Requirement | PMI が根拠とする要求事項 |
| `mfg:verifiedByPmi` | PmiAnnotation → TestReport | PMI 寸法・公差を検証した試験 |
| `mfg:inspectionPlan` | CadModel → InspectionPlan | PMI から生成した検査計画 |
| `mfg:drives` | DimensionAnnotation → DimensionAnnotation | 親パラメータが駆動する子パラメータ |
| `mfg:drivenBy` | DimensionAnnotation → DimensionAnnotation | この寸法を駆動する親パラメータ |

### 4.2 リンク方向の原則

```
上流（規格） ─── conformsTo ───→ 下流（仕様・BOM・試験）
                            ←── derivedFrom ──

要求事項 ─── verifiedBy ───→ 試験成績書
         ←── verifies ────

CAD ─── validatedBy ───→ 試験成績書
    ─── hasPmi ────────→ PMI
    ─── inspectionPlan → 検査計画
```

---

## 5. フォーマット仕様

### 5.1 ファイル名・配置

```
{パッケージID}/
├── ro-crate-metadata.json    ← 必須：全エンティティのリンクグラフ
├── specs/                    ← 製品仕様書 PDF
├── bom/                      ← 部品表 Excel / CSV
├── cad/                      ← STEP / ネイティブ CAD ファイル
├── reports/                  ← 試験成績書 PDF
└── evidence/                 ← その他証跡（写真・ログ・解析結果）
```

### 5.2 ID体系

| エンティティ | IDパターン | 例 |
|---|---|---|
| パッケージ | `PKG-{製品コード}-{版}` | `PKG-RAC35-REL2` |
| 標準規格 | `#std-{規格番号を小文字ハイフン}` | `#std-jis-e4054` |
| 製品仕様 | `#spec-{製品コード}-v{版}` | `#spec-rac35-v2` |
| BOM | `#bom-{製品コード}-v{版}` | `#bom-rac35-v2` |
| BOM部品エントリ | `#bom-entry-{部品略称}` | `#bom-entry-compressor` |
| CADモデル | `#cad-{製品コード}-{部品略称}` | `#cad-rac35-frame` |
| 要求事項 | `#req-{キーワード}` | `#req-vibration-roof` |
| 試験成績書 | `#test-{試験種別}-{年}` | `#test-vibration-2026` |
| PMI（幾何公差） | `#pmi-gt-{連番}` | `#pmi-gt-001` |
| PMI（寸法値） | `#pmi-dim-{部品略称}-{連番}` | `#pmi-dim-frame-w` |
| PMI（表面性状） | `#pmi-sf-{連番}` | `#pmi-sf-001` |
| PMI（注記） | `#pmi-note-{連番}` / `#pmi-procnote-{連番}` | `#pmi-note-001` |
| PMI（溶接） | `#pmi-weld-{連番}` | `#pmi-weld-001` |
| PMI（材料指示） | `#pmi-mat-{連番}` | `#pmi-mat-001` |
| PMI（データム） | `#pmi-datum-{ラベル}` | `#pmi-datum-A` |
| 検査計画 | `#insp-plan-{部品略称}` | `#insp-plan-frame` |

### 5.3 `@context` テンプレート

```json
{
  "@context": [
    "https://w3id.org/ro/crate/1.1/context",
    "https://schema.org",
    {
      "mfg":                 "https://example.org/mfg-knowledge/vocab#",
      "Standard":            "mfg:Standard",
      "ProductSpec":         "mfg:ProductSpec",
      "BillOfMaterials":     "mfg:BillOfMaterials",
      "BomEntry":            "mfg:BomEntry",
      "CadModel":            "mfg:CadModel",
      "TestReport":          "mfg:TestReport",
      "Requirement":         "mfg:Requirement",
      "PmiAnnotation":       "mfg:PmiAnnotation",
      "GeometricTolerance":  "mfg:GeometricTolerance",
      "SurfaceFinish":       "mfg:SurfaceFinish",
      "Datum":               "mfg:Datum",
      "CtqCharacteristic":   "mfg:CtqCharacteristic",
      "InspectionPlan":      "mfg:InspectionPlan",
      "conformsTo":          "mfg:conformsTo",
      "verifies":            "mfg:verifies",
      "derivedFrom":         "mfg:derivedFrom",
      "governedBy":          "mfg:governedBy",
      "validatedBy":         "mfg:validatedBy",
      "linkedCadModel":      "mfg:linkedCadModel",
      "linkedBom":           "mfg:linkedBom",
      "linkedTestReport":    "mfg:linkedTestReport",
      "hasPmi":              "mfg:hasPmi",
      "pmiLinks":            "mfg:pmiLinks",
      "toleranceZone":       "mfg:toleranceZone",
      "datumRef":            "mfg:datumRef",
      "ctqLevel":            "mfg:ctqLevel",
      "inspectionMethod":    "mfg:inspectionMethod",
      "verifiedByPmi":       "mfg:verifiedByPmi",
      "linkedRequirement":   "mfg:linkedRequirement",
      "revision":            "mfg:revision",
      "status":              "mfg:status",
      "DimensionAnnotation": "mfg:DimensionAnnotation",
      "GeneralNote":         "mfg:GeneralNote",
      "ProcessNote":         "mfg:ProcessNote",
      "WeldAnnotation":      "mfg:WeldAnnotation",
      "MaterialAnnotation":  "mfg:MaterialAnnotation",
      "nominal":             "mfg:nominal",
      "upperTol":            "mfg:upperTol",
      "lowerTol":            "mfg:lowerTol",
      "parametricRole":      "mfg:parametricRole",
      "parametricName":      "mfg:parametricName",
      "parametricFormula":   "mfg:parametricFormula",
      "drives":              "mfg:drives",
      "drivenBy":            "mfg:drivenBy",
      "changeImpact":        "mfg:changeImpact",
      "changeHistory":       "mfg:changeHistory",
      "noteItems":           "mfg:noteItems",
      "noteText":            "mfg:noteText",
      "processParameters":   "mfg:processParameters"
    }
  ]
}
```

### 5.4 ルートデータセットエンティティ

```json
{
  "@type": "Dataset",
  "@id": "./",
  "name": "{製品名} 開発ナレッジパッケージ",
  "identifier": "{パッケージID}",
  "description": "標準規格・製品仕様・BOM・CADモデル・試験成績書のインデックスリンク集合",
  "dateCreated": "{YYYY-MM-DD}",
  "hasPart": [
    {"@id": "#std-xxx"},
    {"@id": "#spec-xxx"},
    {"@id": "#bom-xxx"},
    {"@id": "#cad-xxx"},
    {"@id": "#test-xxx"}
  ]
}
```

---

## 6. PMI連携仕様

### 6.1 PMIの定義

PMI（Product Manufacturing Information）は 3D CAD モデルに直接付与される
すべての製造指示情報を指す。本仕様では以下を PMI の範囲とする。

```
PMI（Product Manufacturing Information）
├── 【寸法値】      名目寸法・許容差・参照寸法
├── 【幾何公差】    GD&T / GPS（真円度・位置度・平面度 等）
├── 【表面性状】    粗さパラメータ（Ra / Rz / Rmax）・うねり・加工方法
├── 【注記事項】    一般注記・工程指示・検査指示・特殊要求
├── 【データム】    基準面・基準軸・基準点の定義
├── 【溶接記号】    溶接種別・サイズ・検査区分
└── 【材料・処理指示】材質・熱処理・表面処理・調達仕様
```

さらに本仕様では PMI をパラメトリック設計における **主要パラメータ**と位置づける。
CAD モデル内の各 PMI は他の PMI や要求事項・試験成績書と依存関係（親子関係）を持ち、
1つの寸法値変更が連鎖する影響範囲をグラフで追跡できる。

### 6.2 PMI種別と対応エンティティ型

| PMI種別 | `@type` | `mfg:pmiType` 値 | 主要フィールド |
|---|---|---|---|
| 寸法値（長さ・角度） | `DimensionAnnotation` | `linear_dimension` / `angular_dimension` | `nominal`, `upperTol`, `lowerTol`, `unit` |
| 参照寸法 | `DimensionAnnotation` | `reference_dimension` | `nominal`, `unit`（公差なし・情報のみ） |
| 幾何公差（GD&T/GPS） | `GeometricTolerance` | `geometric_tolerance` | `characteristic`, `symbol`, `toleranceZone`, `datumRef` |
| 表面性状 | `SurfaceFinish` | `surface_finish` | `parameter`（Ra/Rz等）, `value`, `machiningMethod` |
| 一般注記 | `GeneralNote` | `general_note` | `noteText`, `scope` |
| 工程・検査指示注記 | `ProcessNote` | `process_note` | `noteText`, `processType`, `applicableTo` |
| データム | `Datum` | `datum` | `datumLabel`, `datumType`, `description` |
| 溶接記号 | `WeldAnnotation` | `weld_symbol` | `weldType`, `throatSize`, `inspectionCoverage` |
| 材料・熱処理指示 | `MaterialAnnotation` | `material_callout` | `material`, `materialStandard`, `heatTreatment` |

### 6.3 寸法値エンティティ（DimensionAnnotation）の詳細仕様

```json
{
  "@type": ["PmiAnnotation", "DimensionAnnotation"],
  "@id": "#pmi-dim-{連番}",
  "identifier": "PMI-{部品略称}-DIM-{連番}",
  "name": "{寸法の意味・名称}",
  "mfg:pmiType": "linear_dimension",

  "mfg:nominal": 120.0,
  "mfg:upperTol": 0.05,
  "mfg:lowerTol": -0.05,
  "mfg:unit": "mm",

  "mfg:toleranceClass": "IT6",
  "mfg:fitType": "隙間嵌め H7/h6",

  "mfg:ctqLevel": "CTQ",
  "mfg:inspectionMethod": "内径マイクロメータ",
  "mfg:inspectionFrequency": "全数",

  "mfg:parametricRole": "driving",
  "mfg:parametricName": "compressor_bore_dia",
  "mfg:drivenBy": null,
  "mfg:drives": [
    {"@id": "#pmi-dim-{関連寸法}"}
  ],

  "mfg:linkedRequirement": {"@id": "#req-xxx"},
  "mfg:verifiedByPmi": {"@id": "#test-xxx"},
  "mfg:changeHistory": [
    {
      "revision": "A",
      "nominal": 118.0,
      "upperTol": 0.05,
      "lowerTol": -0.05,
      "changedAt": "2025-12-01",
      "reason": "圧縮機サプライヤ変更に伴う外形変更"
    }
  ]
}
```

### 6.4 注記事項エンティティの詳細仕様

注記（Note）は一般注記と工程・検査指示注記に分類する。

#### 一般注記（GeneralNote）

```json
{
  "@type": ["PmiAnnotation", "GeneralNote"],
  "@id": "#pmi-note-{連番}",
  "identifier": "PMI-{部品略称}-NOTE-{連番}",
  "name": "一般注記",
  "mfg:pmiType": "general_note",
  "mfg:scope": "drawing_wide",

  "mfg:noteItems": [
    {
      "seq": 1,
      "text": "図示なき角部は R0.5 以下に面取りすること",
      "mfg:applicableTo": "all_edges"
    },
    {
      "seq": 2,
      "text": "溶接後、変形修正は禁止。規定値を超えた場合は設計部門に連絡すること",
      "mfg:applicableTo": "weld_process"
    },
    {
      "seq": 3,
      "text": "ミルシートは製品に添付すること",
      "mfg:applicableTo": "material_traceability"
    }
  ]
}
```

#### 工程・検査指示注記（ProcessNote）

```json
{
  "@type": ["PmiAnnotation", "ProcessNote"],
  "@id": "#pmi-procnote-{連番}",
  "identifier": "PMI-{部品略称}-PN-{連番}",
  "name": "溶接後熱処理指示",
  "mfg:pmiType": "process_note",
  "mfg:processType": "heat_treatment",
  "mfg:applicableTo": ["#bom-entry-frame"],

  "mfg:noteText": "溶接完了後、応力除去焼鈍を実施すること。条件: 600±10℃ × 2h、炉冷（100℃/h以下）",

  "mfg:processParameters": {
    "temperatureC":    {"nominal": 600, "tolerance": 10},
    "holdTimeH":       {"nominal": 2,   "tolerance": 0},
    "coolingRateCph":  {"max": 100}
  },

  "mfg:ctqLevel": "CTQ",
  "mfg:inspectionMethod": "熱処理炉温度記録紙（チャート）確認",
  "mfg:linkedRequirement": {"@id": "#req-vibration-roof"}
}
```

### 6.5 パラメトリック依存関係の表現

パラメトリック設計では、ある寸法（親パラメータ）が変わると他の寸法（子パラメータ）が連動する。
この依存関係を `mfg:parametricRole` と `mfg:drives` / `mfg:drivenBy` で表現する。

```
mfg:parametricRole の値:
  "driving"  … 他の寸法を駆動する親パラメータ（スケルトンモデルの主要寸法等）
  "driven"   … 親パラメータから計算される子パラメータ
  "reference"… 参照のみ（変更不可の参照寸法）
  "fixed"    … 規格・標準部品により固定される寸法
```

#### 電車用空調機 筐体フレームの例

```json
[
  {
    "@type": ["PmiAnnotation", "DimensionAnnotation"],
    "@id": "#pmi-dim-frame-w",
    "identifier": "PMI-FRM-DIM-010",
    "name": "架台幅（車体取付ピッチ基準）",
    "mfg:pmiType": "linear_dimension",
    "mfg:nominal": 2100,
    "mfg:upperTol": 0,
    "mfg:lowerTol": -2,
    "mfg:unit": "mm",
    "mfg:parametricRole": "driving",
    "mfg:parametricName": "frame_width",
    "mfg:drives": [
      {"@id": "#pmi-dim-cond-w"},
      {"@id": "#pmi-dim-evap-w"},
      {"@id": "#pmi-dim-inv-pos"}
    ],
    "mfg:ctqLevel": "CTQ",
    "mfg:linkedRequirement": {"@id": "#req-weight"},
    "mfg:changeImpact": "車体取付穴ピッチに直結。変更時は車両メーカー承認必須"
  },

  {
    "@type": ["PmiAnnotation", "DimensionAnnotation"],
    "@id": "#pmi-dim-cond-w",
    "identifier": "PMI-COND-DIM-001",
    "name": "凝縮器ユニット幅",
    "mfg:pmiType": "linear_dimension",
    "mfg:nominal": 980,
    "mfg:upperTol": 0,
    "mfg:lowerTol": -1,
    "mfg:unit": "mm",
    "mfg:parametricRole": "driven",
    "mfg:parametricName": "condenser_width",
    "mfg:drivenBy": {"@id": "#pmi-dim-frame-w"},
    "mfg:parametricFormula": "frame_width / 2 - 70",
    "mfg:ctqLevel": "major"
  },

  {
    "@type": ["PmiAnnotation", "DimensionAnnotation"],
    "@id": "#pmi-dim-inv-pos",
    "identifier": "PMI-INV-DIM-001",
    "name": "インバータ盤 取付位置（フレーム端からの距離）",
    "mfg:pmiType": "linear_dimension",
    "mfg:nominal": 150,
    "mfg:upperTol": 2,
    "mfg:lowerTol": -2,
    "mfg:unit": "mm",
    "mfg:parametricRole": "driven",
    "mfg:parametricName": "inverter_pos_x",
    "mfg:drivenBy": {"@id": "#pmi-dim-frame-w"},
    "mfg:ctqLevel": "major",
    "mfg:linkedRequirement": {"@id": "#req-insulation"}
  }
]
```

#### 依存グラフ（architecture_width 変更時の影響波及）

```
#pmi-dim-frame-w（frame_width = 2100mm）  ← 親パラメータ
  │
  ├─ drives → #pmi-dim-cond-w   （condenser_width = frame_width/2 - 70）
  │               └─ linkedRequirement → #req-cooling-capacity
  │               └─ verifiedByPmi    → #test-cooling-2026  ← 再試験トリガ
  │
  ├─ drives → #pmi-dim-evap-w   （evaporator_width = frame_width/2 - 70）
  │               └─ verifiedByPmi → #test-cooling-2026
  │
  └─ drives → #pmi-dim-inv-pos  （inverter_pos = 150mm from edge）
                  └─ linkedRequirement → #req-insulation
                  └─ verifiedByPmi    → #test-insulation-2026 ← 再試験トリガ
```

**frame_width を変更した場合の自動影響抽出:**
1. `#pmi-dim-frame-w.drives` を走査 → 子パラメータ3件を取得
2. 各子パラメータの `verifiedByPmi` を集約 → `TEST-COOL`, `TEST-INS` を再試験リストに追加
3. `linkedRequirement` を集約 → `REQ-001`, `REQ-004` を影響要求事項リストに追加

### 6.6 CTQレベル定義

| `mfg:ctqLevel` | 意味 | 検査頻度の目安 |
|---|---|---|
| `CTQ` | Critical To Quality：安全・機能・規格適合に直結 | 全数検査・記録必須 |
| `major` | 重要特性：製品機能に影響 | 抜取（AQL 1.0） |
| `minor` | 一般特性 | 抜取（AQL 2.5） |

### 6.7 PMI → 検査計画の自動生成ルール

```
1. CadModel.hasPmi[] を走査（DimensionAnnotation・GeometricTolerance・SurfaceFinish・ProcessNote を対象）
2. ctqLevel == "CTQ"   → 全数検査・記録必須として InspectionPlan.inspectionItems[] に追加
3. ctqLevel == "major" → AQL 1.0 抜取として追加
4. ctqLevel == "minor" → AQL 2.5 抜取として追加
5. linkedRequirement が存在 → acceptanceCriteria に要求事項の description を転記
6. ProcessNote（process_note）→ 工程内検査・自主検査項目として別セクションに追加
7. 生成した InspectionPlan を CadModel.inspectionPlan にリンク
```

### 6.8 MBD準拠フラグ

`mfg:mbdCompliant: true` を設定した CAD モデルは、図面（2D図）を廃止し
3D モデルの PMI を唯一の製造指示源とする MBD 運用に対応していることを示す。

MBD運用時、PMI の `mfg:noteItems` および `mfg:noteText` が
従来の図面注記欄に相当する唯一の製造指示となる。

---

## 7. パッケージ構造

### 7.1 ディレクトリ構成

```
{PKG-ID}.zip
├── ro-crate-metadata.json        ← メインリンクグラフ（必須）
├── specs/
│   └── SPEC-{製品コード}-Rev{版}.pdf
├── bom/
│   ├── BOM-{製品コード}-Rev{版}.xlsx
│   └── BOM-{製品コード}-Rev{版}.csv   ← 機械処理用
├── cad/
│   ├── {CAD-ID}-Rev{版}.step          ← 中間フォーマット（STEP AP242）
│   └── {CAD-ID}-Rev{版}.{ext}         ← ネイティブ形式
├── reports/
│   └── {TEST-ID}.pdf
└── evidence/
    ├── *.jpg / *.png                  ← 写真・グラフ
    ├── *.csv                          ← 測定データ
    └── *.pdf                          ← 解析報告書・FEM結果
```

### 7.2 配布・保管方法

| 用途 | 推奨形式 |
|---|---|
| 工場間横展開 | Zip アーカイブ（上記構造） |
| 長期保管・アーカイブ | BagIt 形式（チェックサム付き） |
| バージョン管理 | Git（ro-crate-metadata.json + 証跡ファイル） |
| データリポジトリ登録 | OCFL（Oxford Common File Layout） |

---

## 8. 適用事例：電車用空調機 RAC-35型

### 8.1 製品概要

| 項目 | 内容 |
|---|---|
| 製品名 | 電車屋根上搭載型空調装置 RAC-35型 |
| 定格冷房能力 | 35 kW |
| 搭載対象 | 通勤形電車（ステンレス車体） |
| 準拠規格 | JIS E 4054 / JIS E 7106 / JIS E 4031 / JIS F 8501 |

### 8.2 要求事項と試験成績書の対応表

| 要求事項ID | 内容 | 合否基準 | 検証試験 | 結果 |
|---|---|---|---|---|
| REQ-001 | 定格冷房能力 | ≥ 35 kW | TEST-COOL-2026 | **PASS** 36.2 kW |
| REQ-002 | 成績係数（COP） | ≥ 2.8 | TEST-COOL-2026 | **PASS** 2.99 |
| REQ-003 | 振動耐久性（屋根上） | JIS E 7106 Cat.B 1000h | TEST-VIB-2026 | **PASS** |
| REQ-004 | 絶縁抵抗 | ≥ 10 MΩ（DC 500V） | TEST-INS-2026 | **PASS** 2800 MΩ |
| REQ-005 | 車内騒音寄与度 | ≤ 58 dB(A) | TEST-NSE-2026 | **PASS** 54.2 dB(A) |
| REQ-006 | 耐用年数 | 15年（6000h/年） | TEST-END-2026 | **PASS** |
| REQ-007 | 搭載質量 | ≤ 900 kg | BOM 総質量確認 | **PASS** 872 kg |

### 8.3 CADモデルとPMIの概要

| CADモデル | 主要PMI | CTQ特性 |
|---|---|---|
| 筐体フレーム | 架台取付面 平面度 ⏥0.5、圧縮機取付ボス位置度 ⊕φ0.3 ABC、主要溶接UT検査 | 3件 |
| スクロール圧縮機 | 外形参照モデル（調達品） | — |
| 熱交換器ユニット | フィンピッチ・管径寸法 | 1件 |
| インバータ制御盤 | 実装レイアウト・放熱面粗さ | 1件 |

### 8.4 パッケージ内リンクグラフ（要約）

```
JIS E 4054 ──┐
JIS E 7106 ──┤ conformsTo
JIS E 4031 ──┤            ↓
JIS F 8501 ──┘      SPEC-RAC35-v2
                          │
              ┌───────────┼───────────────┐
              ↓           ↓               ↓
          BOM-RAC35    REQ-001〜007    TEST-×5
              │           │               │
        ┌─────┴─────┐     │ verifiedBy    │
        ↓     ↓     ↓     └──────────────→┘
      CAD   CAD   CAD
     FRAME  COMP  HEX
        │
    hasPmi[]
        │
    ┌───┴──────────┐
    ↓              ↓
 PMI-GT-001    PMI-WELD-001
（平面度CTQ）  （溶接UT）
    │
 linkedRequirement
    ↓
 REQ-003（振動耐久）
```

---

## 9. ユースケース別クエリパターン

### 9.1 規格改訂時の影響調査

**問い：** `JIS E 4054` が改訂された場合、何を見直す必要があるか？

```
#std-jis-e4054
  └─ applicableTo → #spec-rac35-v2
       ├─ requirements[].standardRef == #std-jis-e4054 → REQ-001, REQ-002, REQ-005, REQ-006
       │     └─ verifiedBy → TEST-COOL, TEST-NSE, TEST-END（再試験候補）
       └─ linkedTestReport[].conformsTo == #std-jis-e4054 → 同上試験成績書
```

**出力：** 再試験が必要な可能性のある試験成績書 3件と対応要求事項

---

### 9.2 CAD設計変更時の再試験判断

**問い：** `#cad-rac35-frame`（筐体フレーム）を変更した場合、何の試験をやり直すか？

```
#cad-rac35-frame
  └─ validatedBy → [#test-vibration-2026]（再試験必須）
  └─ hasPmi[].verifiedByPmi → [#test-vibration-2026]（PMI検証試験）
```

**出力：** `TEST-VIB-RAC35-2026` の再実施が必要

---

### 9.3 要求事項の根拠規格の確認

**問い：** `REQ-003`（振動耐久）の根拠となる規格条項は？

```
#req-vibration-roof
  └─ standardRef → #std-jis-e7106
       └─ identifier: "JIS E 7106"
  └─ standardClause: "5.2"
```

**出力：** `JIS E 7106:2014 第5.2条`

---

### 9.4 CTQ特性の一覧抽出

**問い：** 筐体フレームの CTQ 特性（全数検査項目）は何か？

```
#cad-rac35-frame.hasPmi[]
  → filter(ctqLevel == "CTQ")
  → [#pmi-gt-001, #pmi-gt-002, #pmi-weld-001]
```

**出力：**
- 架台取付面 平面度 ⏥0.5（CMM全数）
- 圧縮機取付ボス 位置度 ⊕φ0.3 ABC（CMM全数）
- 主要溶接部 超音波探傷（UT全数）

---

### 9.5 BOM変更時の質量要求適合確認

**問い：** 部品を変更した場合、質量制限（REQ-007: ≤ 900 kg）を超えないか？

```
#bom-rac35-v2.mfg:entries[].massKg を合算
  → totalMassKg を更新
  → #req-weight の acceptanceCriteria（≤ 900 kg）と照合
```

**出力：** 再計算後の総質量と適合可否

---

### 9.6 試験未完了要求事項の検出

**問い：** まだ試験成績書が紐づいていない要求事項はあるか？

```
#spec-rac35-v2.requirements[]
  → filter(mfg:verifiedBy == null または mfg:verifiedBy.mfg:status != "approved")
```

**出力：** 未検証の要求事項リスト（リリース前チェックに活用）

---

## 10. 拡張ガイドライン

### 10.1 製品カテゴリ別プロファイル

本フォーマットは以下のプロファイル拡張を推奨する。

| プロファイル | 追加エンティティ | 適用領域 |
|---|---|---|
| `mfg-trouble` | `TroubleRecord`, `RootCauseAnalysis`, `Countermeasure`, `HorizontalDeployment` | 不具合対策ナレッジ |
| `mfg-process` | `ProcessStep`, `WorkInstruction`, `ProcessParameter` | 工程設計・作業標準 |
| `mfg-improvement` | `KaizenProposal`, `EffectMeasurement` | 改善ナレッジ |
| `mfg-skill` | `TacitKnowledge`, `TrainingRecord`, `SkillLevel` | 技能ナレッジ |

### 10.2 語彙の拡張方法

`mfg:` namespace への追加は以下の手順で管理する。

1. 追加語彙を `vocab/mfg-vocab.jsonld` に定義
2. 既存フィールドとの重複・競合を確認
3. バージョン管理（`schema_version` のマイナー番号を上げる）
4. 後方互換性を維持（既存フィールドの削除・型変更は禁止）

### 10.3 ナレッジの鮮度管理

```json
"mfg:lifecycle": {
  "reviewCycleMonths": 12,
  "nextReviewDate": "{YYYY-MM-DD}",
  "expiryDate": null,
  "archived": false
}
```

- 試験成績書：製品廃番後 10年間保管（規制要件に準拠）
- CADモデル：設計変更ごとに改版・旧版アーカイブ
- 標準規格：改訂追跡のため `mfg:status` を定期確認

### 10.4 関連標準との位置づけ

| 標準・仕様 | 本手法との関係 |
|---|---|
| RO-Crate 1.1 | メタデータパッケージング基盤として準拠 |
| schema.org | 汎用語彙（Person, Organization, Dataset 等）として利用 |
| ISO 16792:2015 | MBD（Model-Based Definition）のCAD運用標準 |
| STEP AP242 | CAD 中間フォーマット（PMI 情報を含む） |
| JIS B 0001:2019 | 機械製図（2D 図との併用時） |
| LOTAR（LOng Term Archiving and Retrieval） | 長期アーカイブ要件との整合 |

---

*本ドキュメントは RO-Crate 思想を製造業ナレッジ管理に適用した設計仕様書です。*
*語彙仕様（`mfg:` namespace）は各組織のシステム要件に応じて拡張してください。*

---

## 11. データ構築プロセス：段階的ナレッジ蓄積アプローチ

### 11.1 基本コンセプト：最小単位は「親との紐づけ」

設計担当者が設計事項を検討する際、**参照した親文書へのリンク1本**が最小のナレッジ単位となる。
この断片を積み重ねることで、開発プロセスの進行とともに JSON が有機的に成長する。

```json
{
  "@id": "#design-item-001",
  "@type": "mfg:DesignItem",
  "name": "フレーム材質選定",
  "mfg:designPhase": "基本設計",
  "dcterms:isPartOf":    {"@id": "#spec-rac35-v2"},
  "prov:wasDerivedFrom": {"@id": "#req-vibration-roof"}
}
```

設計事項の断片が蓄積されると、依存関係・検証関係がリンクされ、
最終的に製品全体のナレッジグラフ（大規模 JSON-LD）が構築される。

```
設計事項A（材質選定）─┐
設計事項B（板厚決定）─┼──→ CADモデル（PMI付き）─┐
設計事項C（形状決定）─┘                          ├──→ 試験成績書 ──→ 量産仕様
設計事項D（穴位置） ──→ BOM ────────────────────┘
```

### 11.2 追加される語彙プレフィックス

| プレフィックス | 用途 | 代表的な述語 |
|---|---|---|
| `mfg:` | 製造業固有の型・述語（独自オントロジー） | `mfg:drives`, `mfg:verifies`, `mfg:hasPmi` |
| `dcterms:` | 文書間の親子・参照関係 | `dcterms:isPartOf` |
| `prov:` | 派生・変更履歴（W3C PROV） | `prov:wasDerivedFrom`, `prov:wasRevisionOf` |
| `schema:` | 汎用メタデータ（名称・素材など） | `schema:associatedMedia` |

### 11.3 DesignItem（設計事項）エンティティ

**`@type`:** `"mfg:DesignItem"`
**`@id` パターン:** `#design-item-{連番}` （例: `#design-item-001`）

設計検討中の断片ナレッジを表す。詳細設計フェーズで CadModel や BomEntry に昇格する。

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `name` | string | ✓ | 設計事項名（例: 「フレーム材質選定」） |
| `description` | string | — | 検討内容・根拠 |
| `mfg:designPhase` | enum | ✓ | 設計フェーズ（下記参照） |
| `dcterms:isPartOf` | IdRef | ✓ | 所属する親文書（ProductSpec 等） `@id` |
| `prov:wasDerivedFrom` | IdRef | — | 派生元の要求事項・上位設計事項 `@id` |
| `mfg:decidedBy` | IdRef | — | 決定者（Person `@id`） |
| `mfg:decidedAt` | IsoDateTime | — | 決定日時 |
| `mfg:status` | enum | ✓ | `under_study` / `decided` / `superseded` |
| `prov:wasRevisionOf` | IdRef | — | 旧版設計事項 `@id`（改訂時） |
| `mfg:promotedTo` | IdRef | — | 昇格先のエンティティ（CadModel・BomEntry 等） |

**`mfg:designPhase` 列挙値:**

| 値 | フェーズ |
|---|---|
| `requirements` | 要件定義 |
| `basic_design` | 基本設計 |
| `detail_design` | 詳細設計 |
| `cad_modeling` | CAD作成 |
| `verification` | 試験・評価 |
| `production` | 量産移行 |

### 11.4 フェーズ別のナレッジ構築フロー

| フェーズ | 設計作業 | ナレッジ操作 | 生成される主なリンク |
|---|---|---|---|
| **要件定義** | 規格・仕様書の確定 | Standard / ProductSpec ノード作成 | `conformsTo` |
| **基本設計** | 設計事項の検討（親文書参照） | DesignItem ノードを逐次追加 | `dcterms:isPartOf` / `prov:wasDerivedFrom` |
| **詳細設計** | 設計事項間の依存関係確定 | 横断リンクを追加 | `mfg:drives` / `dependsOn` |
| **CAD作成** | 3DモデルへのPMI付与 | CadModel / PmiAnnotation ノード追加 | `mfg:hasPmi` / `mfg:linkedCadModel` |
| **試験・評価** | 試験成績書の発行 | TestReport ノード追加・検証リンク完結 | `mfg:verifies` / `mfg:verifiedByPmi` |
| **量産移行** | 量産仕様の確定 | DesignItem を `mfg:promotedTo` で昇格先に紐づけ | `mfg:promotedTo` |

#### フェーズ進行に伴うグラフの成長イメージ

```
【要件定義フェーズ】
  #std-jis-e4054 ──conformsTo──→ #spec-rac35-v2
                                       │
                                  requirements[]
                                       │
                                  REQ-001, REQ-003, ...

【基本設計フェーズ】（DesignItem が追加される）
  #design-item-001（材質選定）
    └─ dcterms:isPartOf    ──→ #spec-rac35-v2
    └─ prov:wasDerivedFrom ──→ #req-vibration-roof

  #design-item-002（板厚決定）
    └─ dcterms:isPartOf    ──→ #spec-rac35-v2
    └─ prov:wasDerivedFrom ──→ #design-item-001  ← 設計事項同士がリンク

【詳細設計フェーズ】（依存関係リンクが追加される）
  #design-item-003（フレーム幅決定）
    └─ mfg:drives ──→ #design-item-004（凝縮器幅）
    └─ mfg:drives ──→ #design-item-005（インバータ位置）

【CAD作成フェーズ】（CadModel・PMI に昇格）
  #cad-rac35-frame
    └─ mfg:promotedFrom ──→ #design-item-001, #design-item-002, #design-item-003
    └─ mfg:hasPmi ──→ [#pmi-dim-frame-w, #pmi-gt-001, ...]

【試験・評価フェーズ】（TestReport でグラフが閉じる）
  #test-vibration-2026
    └─ mfg:verifies ──→ REQ-003
    └─ mfg:testedObject ──→ #cad-rac35-frame
```

### 11.5 断片 JSON のマージ戦略

断片 JSON（1設計事項ごとの小さな JSON）を統合ナレッジファイルにマージする際のルール。

```
マージルール:
1. 同一 @id が存在しない場合  → @graph[] に新規追加
2. 同一 @id が存在する場合    → フィールドを上書きマージ
                               旧バージョンは prov:wasRevisionOf で参照保持
3. リンクフィールド（IdRef[]）→ 既存要素を保持したまま新要素を追加
4. mfg:status の遷移         → under_study → decided → superseded のみ許可
```

```json
{
  "@id": "#design-item-001",
  "@type": "mfg:DesignItem",
  "name": "フレーム材質選定",
  "mfg:status": "decided",
  "mfg:promotedTo": {"@id": "#cad-rac35-frame"},
  "prov:wasRevisionOf": {"@id": "#design-item-001-draft"},
  "prov:wasDerivedFrom": {"@id": "#req-vibration-roof"}
}
```

### 11.6 Creo トレイルファイルとの連携

PTC Creo のトレイルファイル（モデリング操作ログ）を CadModel ノードに紐づけることで、
CAD モデリング自動化システムとの連携が可能になる。

```json
{
  "@type": "mfg:CadModel",
  "@id": "#cad-rac35-frame",
  "name": "RAC-35型 筐体フレーム・架台 3Dモデル",
  "mfg:fileRef": "cad/RAC35-FRM-RevB.step",

  "schema:associatedMedia": [
    {
      "@type": "MediaObject",
      "name": "Creo トレイルファイル Rev.B",
      "encodingFormat": "text/plain",
      "contentUrl": "cad/trail/RAC35-FRM-RevB.trl",
      "mfg:mediaRole": "creo_trail"
    },
    {
      "@type": "MediaObject",
      "name": "Creo ネイティブモデル",
      "encodingFormat": "application/octet-stream",
      "contentUrl": "cad/RAC35-FRM-RevB.prt",
      "mfg:mediaRole": "cad_native"
    }
  ]
}
```

### 11.7 システム実装ファイル構成

```
knowledge-management/
├── ro-crate-metadata.json        ← 統合ナレッジデータ（メインファイル）
│
├── fragments/                    ← 断片ナレッジ（設計事項ごとの小JSON）
│   ├── design-item-001.json      ← フレーム材質選定
│   ├── design-item-002.json      ← 板厚決定
│   └── ...
│
├── scripts/
│   ├── merge.py                  ← 断片JSON を ro-crate-metadata.json にマージ
│   ├── validate.py               ← JSON-LD構文・リンク整合性チェック
│   └── trace.py                  ← トレーサビリティレポート生成
│
└── templates/
    ├── design-item.template.json ← 設計事項断片のテンプレート
    └── test-report.template.json ← 試験成績書ノードのテンプレート
```

### 11.8 断片 JSON テンプレート

#### design-item.template.json

```json
{
  "@context": [
    "https://w3id.org/ro/crate/1.1/context",
    "https://schema.org",
    {
      "mfg":     "https://example.org/mfg-knowledge/vocab#",
      "dcterms": "http://purl.org/dc/terms/",
      "prov":    "http://www.w3.org/ns/prov#"
    }
  ],
  "@graph": [
    {
      "@type": "mfg:DesignItem",
      "@id": "#design-item-{連番}",
      "name": "{設計事項名}",
      "description": "{検討内容・根拠}",
      "mfg:designPhase": "{フェーズ}",
      "mfg:status": "under_study",
      "dcterms:isPartOf":    {"@id": "{親文書 @id}"},
      "prov:wasDerivedFrom": {"@id": "{派生元 @id}"},
      "mfg:decidedBy": {"@id": "{担当者 @id}"},
      "mfg:decidedAt": "{YYYY-MM-DDTHH:MM:SS+09:00}"
    }
  ]
}
```

#### test-report.template.json

```json
{
  "@context": [ "..." ],
  "@graph": [
    {
      "@type": "mfg:TestReport",
      "@id": "#test-{種別}-{年}",
      "name": "{試験成績書名称}",
      "mfg:revision": "1.0",
      "mfg:status": "draft",
      "dateCreated": "{YYYY-MM-DD}",
      "mfg:verifies": [{"@id": "{要求事項 @id}"}],
      "mfg:conformsTo": [{"@id": "{規格 @id}"}],
      "mfg:testedObject": [{"@id": "{CADモデル @id}"}],
      "mfg:derivedFrom": {"@id": "{仕様書 @id}"},
      "mfg:testConditions": {},
      "mfg:verdict": "PASS",
      "mfg:fileRef": "reports/{ファイル名}.pdf",
      "mfg:measurements": []
    }
  ]
}
```

### 11.9 実装上の注意事項

| 事項 | 内容 |
|---|---|
| **`@id` の一意性** | プロジェクト全体で衝突しないよう命名規則を厳守する（`#[type]-[product]-[seq]` 形式） |
| **マージ戦略** | 同一 `@id` が存在する場合は上書きとし、旧バージョンは `prov:wasRevisionOf` で参照保持 |
| **フェーズメタデータ** | 各ノードに `mfg:designPhase` を付与し、フェーズ別のナレッジフィルタリングを可能にする |
| **Creo 連携** | CADモデルノードにトレイルファイルのパスを `schema:associatedMedia` で紐づける |
| **ステータス遷移の制約** | `mfg:status` は `under_study → decided → superseded` の一方向遷移のみ許可 |
| **参照整合性** | `fragments/` の各断片にある `@id` 参照が `ro-crate-metadata.json` に存在することをマージ時に検証 |
