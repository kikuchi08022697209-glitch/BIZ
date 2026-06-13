# 製造業ナレッジデータ管理 解説ガイド

**対象読者:** 設計者・品質管理者・製造技術者・IT担当者
**前提知識:** JSON の基本的な読み書き・製造業の一般的な設計プロセス

---

## 目次

1. [このデータ管理の「なぜ」](#1-このデータ管理のなぜ)
2. [全体のイメージ：5枚のカードをひもで結ぶ](#2-全体のイメージ5枚のカードをひもで結ぶ)
3. [基本的な書き方](#3-基本的な書き方)
4. [ドキュメント種別ごとの書き方](#4-ドキュメント種別ごとの書き方)
5. [PMI（製品製造情報）の書き方](#5-pmi製品製造情報の書き方)
6. [よくある操作パターン](#6-よくある操作パターン)
7. [実務での運用フロー](#7-実務での運用フロー)
8. [よくある質問（FAQ）](#8-よくある質問faq)

---

## 1. このデータ管理の「なぜ」

### 1.1 現場でよく起きる困りごと

```
「この部品の公差を変えたいんだけど、どの試験成績書に影響するの？」
「JIS規格が改訂されたけど、どの仕様書を見直せばいい？」
「この試験成績書、どの要求事項を検証したものだっけ？」
「CAD Rev.B にしたとき、BOMと試験成績書も連動してる？」
```

これらはすべて**「ドキュメント間のつながりが見えない」**ことで生じます。

### 1.2 このフォーマットが解決すること

本フォーマットは、設計ドキュメントを**グラフ（ノードとエッジのネットワーク）**として管理します。
各ドキュメントが `@id` という住所を持ち、互いを指し示す `@id` の参照でつながります。

```
つながりが見える → 変更の影響範囲が即座にわかる
                → 要求事項と試験成績書の対応が自動で追跡できる
                → 規格改訂時に「何を見直すべきか」が一覧で出る
```

### 1.3 RO-Crate という土台

このフォーマットは **RO-Crate**（Research Object Crate）という国際標準の構造を
製造業向けに拡張したものです。RO-Crate はもともと研究データの管理に使われており、
「データ本体とメタデータを一体でパッケージ化する」という思想を持っています。

製造業に当てはめると：

| 研究データ管理（RO-Crate） | 製造業ナレッジ管理 |
|---|---|
| 論文 | 製品仕様書 |
| 実験データ | 試験成績書 |
| 試薬・装置情報 | BOM・CADモデル |
| 研究者プロフィール | 設計者・試験担当者 |
| 引用文献 | 準拠する標準規格 |

---

## 2. 全体のイメージ：5枚のカードをひもで結ぶ

### 2.1 5種類のドキュメント（カード）

```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│  標準規格    │   │  製品仕様書  │   │   BOM       │
│  Standard   │   │ ProductSpec │   │（部品表）    │
│             │   │             │   │             │
│ JIS E 4054  │   │ 要求事項を  │   │ 部品リスト  │
│ JIS E 7106  │   │ 定義する    │   │ 質量管理    │
└─────────────┘   └─────────────┘   └─────────────┘

┌─────────────┐   ┌─────────────┐
│  CADモデル  │   │ 試験成績書  │
│  CadModel   │   │ TestReport  │
│             │   │             │
│ 3D形状＋PMI │   │ 試験結果    │
│ 寸法・公差  │   │ 合否記録    │
└─────────────┘   └─────────────┘
```

### 2.2 ひも（リンク）の種類と方向

```
標準規格
  │
  │ conformsTo（〜に準拠する）
  ↓
製品仕様書 ─── linkedBom ──────────────→ BOM
  │                                        │
  │ requirements                           │ entries
  ↓                                        ↓
要求事項 ──── verifiedBy ─→ 試験成績書   部品エントリ
                                           │
                                    linkedCadModel
                                           ↓
                                       CADモデル
                                           │
                                         hasPmi
                                           ↓
                                       PMIアノテーション
                                    （寸法値・公差・注記）
```

### 2.3 パッケージとして一体管理

ドキュメントとそのつながり情報は、1つのフォルダにまとめて管理します。

```
PKG-RAC35-REL2/                     ← パッケージフォルダ
│
├── ro-crate-metadata.json           ← 【核心】全ドキュメントのつながりを記述
│
├── specs/SPEC-RAC35-Rev2.pdf        ← 製品仕様書 PDF
├── bom/BOM-RAC35-Rev2.xlsx          ← BOM Excel
├── cad/RAC35-FRM-RevB.step          ← CAD STEP ファイル
├── reports/TEST-COOL-RAC35-2026.pdf ← 試験成績書 PDF
└── evidence/xray_defect.jpg         ← 証跡ファイル
```

**`ro-crate-metadata.json` が「目次兼インデックス」であり、
このファイルを読めば全ドキュメントのつながりが把握できます。**

---

## 3. 基本的な書き方

### 3.1 ファイルの骨格

```json
{
  "@context": [ ... ],    ← 「この言葉はこういう意味です」という辞書
  "@graph":   [ ... ]     ← 全エンティティ（ドキュメント）の配列
}
```

### 3.2 `@context`（辞書）は3層構造

```json
"@context": [
  "https://w3id.org/ro/crate/1.1/context",    ← RO-Crateの標準語彙
  "https://schema.org",                        ← schema.orgの汎用語彙
  {
    "mfg": "https://example.org/mfg-knowledge/vocab#",  ← 製造業拡張語彙の名前空間
    "Standard":     "mfg:Standard",
    "ProductSpec":  "mfg:ProductSpec",
    "conformsTo":   "mfg:conformsTo",
    ...
  }
]
```

`mfg:` で始まるフィールド・型はすべて製造業拡張の語彙です。

### 3.3 `@graph` の先頭2要素（定型文）

```json
"@graph": [
  {
    "@type": "CreativeWork",
    "@id": "ro-crate-metadata.json",
    "conformsTo": {"@id": "https://w3id.org/ro/crate/1.1"},
    "about": {"@id": "./"}
  },
  {
    "@type": "Dataset",
    "@id": "./",
    "identifier": "PKG-RAC35-REL2",
    "name": "電車屋根上搭載型空調装置 RAC-35型 開発ナレッジパッケージ",
    "dateCreated": "2026-06-13",
    "hasPart": [
      {"@id": "#std-jis-e4054"},
      {"@id": "#spec-rac35-v2"},
      {"@id": "#bom-rac35-v2"},
      {"@id": "#cad-rac35-frame"},
      {"@id": "#test-cooling-2026"}
    ]
  },
  ...
]
```

`hasPart` にはこのパッケージに含まれる全エンティティの `@id` を列挙します。

### 3.4 `@id`（住所）の命名ルール

| エンティティ | パターン | 例 |
|---|---|---|
| 標準規格 | `#std-{番号}` | `#std-jis-e4054` |
| 製品仕様書 | `#spec-{製品}-v{版}` | `#spec-rac35-v2` |
| BOM | `#bom-{製品}-v{版}` | `#bom-rac35-v2` |
| 部品エントリ | `#bom-entry-{略称}` | `#bom-entry-compressor` |
| CADモデル | `#cad-{製品}-{略称}` | `#cad-rac35-frame` |
| 要求事項 | `#req-{キーワード}` | `#req-vibration-roof` |
| 試験成績書 | `#test-{種別}-{年}` | `#test-vibration-2026` |
| 担当者 | `#emp-{番号}` | `#emp-0042` |
| 部署 | `#dept-{略称}` | `#dept-mfg2` |

---

## 4. ドキュメント種別ごとの書き方

### 4.1 Standard（標準規格）の書き方

```json
{
  "@type": "mfg:Standard",
  "@id": "#std-jis-e4054",
  "identifier": "JIS E 4054",
  "name": "鉄道車両用空気調和装置",
  "version": "2016",
  "mfg:issuer": "日本産業標準調査会（JISC）",
  "mfg:status": "active",
  "mfg:applicableTo": [{"@id": "#spec-rac35-v2"}]
}
```

**ポイント：**
- `mfg:applicableTo` で「この規格を使っている仕様書」へのリンクを張る
- `mfg:status` は規格の現在の有効性を示す（`active` / `withdrawn` / `under_revision`）

---

### 4.2 ProductSpec（製品仕様書）の書き方

製品仕様書の核心は **`mfg:requirements`（要求事項の配列）** です。

```json
{
  "@type": "mfg:ProductSpec",
  "@id": "#spec-rac35-v2",
  "identifier": "SPEC-RAC35",
  "name": "RAC-35型 電車屋根上搭載空調装置 製品仕様書",
  "mfg:revision": "2.0",
  "mfg:status": "released",

  "conformsTo": [
    {"@id": "#std-jis-e4054"},
    {"@id": "#std-jis-e7106"}
  ],

  "mfg:requirements": [
    {
      "@type": "mfg:Requirement",
      "@id": "#req-cooling-capacity",
      "identifier": "REQ-001",
      "name": "定格冷房能力",
      "description": "外気温度 35℃、車内温度 27℃ 条件下で冷房能力 35 kW 以上",
      "mfg:category": "performance",
      "mfg:priority": "shall",
      "mfg:standardRef": {"@id": "#std-jis-e4054"},
      "mfg:standardClause": "6.3.1",
      "mfg:verifiedBy": {"@id": "#test-cooling-2026"}
    }
  ],

  "mfg:linkedBom": {"@id": "#bom-rac35-v2"},
  "mfg:linkedTestReport": [{"@id": "#test-cooling-2026"}]
}
```

**ポイント：**
- `conformsTo` → 準拠する規格（複数可）
- `mfg:requirements[].mfg:verifiedBy` → この要求事項を検証する試験成績書
- `mfg:linkedBom` / `mfg:linkedTestReport` → 下流ドキュメントへのリンク

---

### 4.3 BillOfMaterials（BOM）の書き方

```json
{
  "@type": "mfg:BillOfMaterials",
  "@id": "#bom-rac35-v2",
  "identifier": "BOM-RAC35",
  "name": "RAC-35型 部品表",
  "mfg:revision": "2.0",
  "mfg:status": "released",
  "derivedFrom": {"@id": "#spec-rac35-v2"},
  "mfg:totalMassKg": 872,

  "mfg:entries": [
    {
      "@type": "mfg:BomEntry",
      "@id": "#bom-entry-frame",
      "identifier": "PRT-101",
      "name": "筐体フレーム",
      "mfg:partNumber": "RAC35-FRM-001",
      "mfg:quantity": 1,
      "mfg:unit": "式",
      "mfg:material": "SUS304",
      "mfg:massKg": 210,
      "mfg:linkedCadModel": {"@id": "#cad-rac35-frame"},
      "mfg:governedBy": {"@id": "#std-jis-e7106"}
    },
    {
      "@type": "mfg:BomEntry",
      "@id": "#bom-entry-compressor",
      "identifier": "PRT-102",
      "name": "スクロール圧縮機",
      "mfg:partNumber": "RAC35-COMP-001",
      "mfg:quantity": 2,
      "mfg:unit": "台",
      "mfg:massKg": 85,
      "mfg:procuredFrom": "XXX株式会社 型式: SCC-18",
      "mfg:linkedCadModel": {"@id": "#cad-rac35-compressor"}
    }
  ]
}
```

**ポイント：**
- `derivedFrom` → どの仕様書から作られたBOMか
- `mfg:linkedCadModel` → 部品と3Dモデルをつなぐ（調達品で外形モデルなしの場合は `null`）
- `mfg:totalMassKg` → 質量要求（REQ-007）の検証に使用

---

### 4.4 CadModel（3DCADモデル）の書き方

```json
{
  "@type": "mfg:CadModel",
  "@id": "#cad-rac35-frame",
  "identifier": "CAD-RAC35-FRM",
  "name": "RAC-35型 筐体フレーム・架台 3Dモデル（MBD）",
  "mfg:revision": "B",
  "mfg:status": "released",
  "mfg:format": "STEP AP242",
  "mfg:fileRef": "cad/RAC35-FRM-RevB.step",
  "mfg:nativeFileRef": "cad/RAC35-FRM-RevB.catpart",

  "mfg:mbdCompliant": true,
  "mfg:mbdStandard": "ISO 16792:2015",

  "mfg:derivedFrom": {"@id": "#spec-rac35-v2"},
  "mfg:referencedInBom": {"@id": "#bom-entry-frame"},
  "mfg:validatedBy": [{"@id": "#test-vibration-2026"}],

  "mfg:hasPmi": [ ... ],

  "mfg:inspectionPlan": {"@id": "#insp-plan-frame"}
}
```

**ポイント：**
- `mfg:mbdCompliant: true` → 2D図廃止・3Dモデルが唯一の製造指示源
- `mfg:validatedBy` → このモデルを検証した試験成績書（複数可）
- `mfg:hasPmi` → PMIアノテーションの配列（次章参照）

---

### 4.5 TestReport（試験成績書）の書き方

```json
{
  "@type": "mfg:TestReport",
  "@id": "#test-cooling-2026",
  "identifier": "TEST-COOL-RAC35-2026",
  "name": "RAC-35型 冷房性能試験成績書",
  "mfg:revision": "1.0",
  "mfg:status": "approved",
  "dateCreated": "2026-05-15",

  "mfg:verifies": [
    {"@id": "#req-cooling-capacity"},
    {"@id": "#req-cop"}
  ],
  "mfg:conformsTo": [{"@id": "#std-jis-e4054"}],
  "mfg:testedObject": [{"@id": "#cad-rac35-compressor"}],
  "mfg:derivedFrom": {"@id": "#spec-rac35-v2"},

  "mfg:testConditions": {
    "outdoorTempC": 35,
    "indoorTempC": 27,
    "supplyVoltageV": 600
  },

  "mfg:verdict": "PASS",
  "mfg:fileRef": "reports/TEST-COOL-RAC35-2026.pdf",

  "mfg:measurements": [
    {"parameter": "冷房能力", "value": 36.2, "unit": "kW", "limit": "≥35.0", "result": "PASS"},
    {"parameter": "COP",     "value": 2.99,  "unit": "—",  "limit": "≥2.8",  "result": "PASS"}
  ]
}
```

**ポイント：**
- `mfg:verifies` → どの要求事項を検証したか（複数可）
- `mfg:conformsTo` → 試験が準拠する規格
- `mfg:verdict` → `PASS` / `FAIL` / `CONDITIONAL`

---

## 5. PMI（製品製造情報）の書き方

PMI は `CadModel.mfg:hasPmi` 配列に埋め込みます。
PMI には8種類あり、`mfg:pmiType` フィールドで種別を宣言します。

### 5.1 PMIの8種類まとめ

| 種別 | `mfg:pmiType` | 用途 | `@id` パターン |
|---|---|---|---|
| 寸法値 | `linear_dimension` | 長さ・径・距離 | `#pmi-dim-` |
| 角度寸法 | `angular_dimension` | 角度 | `#pmi-dim-` |
| 参照寸法 | `reference_dimension` | 参考値（公差なし） | `#pmi-dim-` |
| 幾何公差 | `geometric_tolerance` | 真円度・位置度 等 | `#pmi-gt-` |
| 表面性状 | `surface_finish` | 粗さ Ra・Rz 等 | `#pmi-sf-` |
| 一般注記 | `general_note` | 全体に適用される指示 | `#pmi-note-` |
| 工程指示注記 | `process_note` | 熱処理・溶接条件 等 | `#pmi-procnote-` |
| データム | `datum` | 基準面・基準軸 | `#pmi-datum-` |
| 溶接記号 | `weld_symbol` | 溶接指示 | `#pmi-weld-` |
| 材料指示 | `material_callout` | 材質・熱処理 | `#pmi-mat-` |

### 5.2 寸法値（DimensionAnnotation）の書き方

```json
{
  "@type": ["mfg:PmiAnnotation", "mfg:DimensionAnnotation"],
  "@id": "#pmi-dim-frame-w",
  "identifier": "PMI-FRM-DIM-010",
  "name": "架台幅（車体取付ピッチ基準）",
  "mfg:pmiType": "linear_dimension",

  "mfg:nominal": 2100,
  "mfg:upperTol": 0,
  "mfg:lowerTol": -2,
  "mfg:unit": "mm",

  "mfg:ctqLevel": "CTQ",
  "mfg:inspectionMethod": "3次元測定機（CMM）",
  "mfg:inspectionFrequency": "全数",

  "mfg:parametricRole": "driving",
  "mfg:parametricName": "frame_width",
  "mfg:drives": [
    {"@id": "#pmi-dim-cond-w"},
    {"@id": "#pmi-dim-evap-w"}
  ],
  "mfg:changeImpact": "車体取付穴ピッチに直結。変更時は車両メーカー承認必須",

  "mfg:linkedRequirement": {"@id": "#req-weight"}
}
```

**寸法値の公差表現：**

| 表記例 | `nominal` | `upperTol` | `lowerTol` |
|---|---|---|---|
| `120 ±0.05` | `120` | `0.05` | `-0.05` |
| `120 +0/-0.1` | `120` | `0` | `-0.1` |
| `120 h6`（IT6） | `120` | `0` | `-0.022` |

### 5.3 パラメトリック依存関係の書き方

パラメトリック設計では「親寸法が変わると子寸法が連動」します。
この関係を `mfg:parametricRole` / `mfg:drives` / `mfg:drivenBy` で表現します。

```
役割（mfg:parametricRole）:
  "driving"   … 他の寸法を計算式で駆動する【親】
  "driven"    … 親寸法の計算式から決まる【子】
  "reference" … 参照のみ（変更不可）
  "fixed"     … 規格・標準部品により固定
```

**親パラメータの書き方：**

```json
{
  "mfg:parametricRole": "driving",
  "mfg:parametricName": "frame_width",
  "mfg:drives": [
    {"@id": "#pmi-dim-cond-w"},   ← この @id が駆動される子
    {"@id": "#pmi-dim-evap-w"}
  ]
}
```

**子パラメータの書き方：**

```json
{
  "mfg:parametricRole": "driven",
  "mfg:parametricName": "condenser_width",
  "mfg:drivenBy": {"@id": "#pmi-dim-frame-w"},   ← 親の @id
  "mfg:parametricFormula": "frame_width / 2 - 70"
}
```

### 5.4 幾何公差（GeometricTolerance）の書き方

```json
{
  "@type": ["mfg:PmiAnnotation", "mfg:GeometricTolerance"],
  "@id": "#pmi-gt-001",
  "identifier": "PMI-FRM-GT-001",
  "name": "架台取付面 平面度",
  "mfg:pmiType": "geometric_tolerance",

  "mfg:characteristic": "平面度",
  "mfg:symbol": "⏥",
  "mfg:toleranceZone": {"value": 0.5, "unit": "mm"},
  "mfg:datumRef": null,

  "mfg:ctqLevel": "CTQ",
  "mfg:inspectionMethod": "3次元測定機（CMM）",
  "mfg:inspectionFrequency": "全数",

  "mfg:linkedRequirement": {"@id": "#req-vibration-roof"},
  "mfg:verifiedByPmi": {"@id": "#test-vibration-2026"}
}
```

```json
{
  "@type": ["mfg:PmiAnnotation", "mfg:GeometricTolerance"],
  "@id": "#pmi-gt-002",
  "name": "圧縮機取付ボス 位置度",
  "mfg:pmiType": "geometric_tolerance",
  "mfg:characteristic": "位置度",
  "mfg:symbol": "⊕",
  "mfg:toleranceZone": {"value": 0.3, "unit": "mm", "shape": "円"},
  "mfg:datumRef": [
    {"@id": "#pmi-datum-A"},
    {"@id": "#pmi-datum-B"},
    {"@id": "#pmi-datum-C"}
  ]
}
```

### 5.5 一般注記（GeneralNote）の書き方

```json
{
  "@type": ["mfg:PmiAnnotation", "mfg:GeneralNote"],
  "@id": "#pmi-note-001",
  "identifier": "PMI-FRM-NOTE-001",
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

### 5.6 工程指示注記（ProcessNote）の書き方

工程指示注記では `mfg:processParameters` に数値条件を構造化します。

```json
{
  "@type": ["mfg:PmiAnnotation", "mfg:ProcessNote"],
  "@id": "#pmi-procnote-001",
  "identifier": "PMI-FRM-PN-001",
  "name": "溶接後熱処理指示",
  "mfg:pmiType": "process_note",
  "mfg:processType": "heat_treatment",

  "mfg:noteText": "溶接完了後、応力除去焼鈍を実施すること。条件: 600±10℃ × 2h、炉冷（100℃/h以下）",

  "mfg:processParameters": {
    "temperatureC":   {"nominal": 600, "tolerance": 10,  "unit": "℃"},
    "holdTimeH":      {"nominal": 2,   "tolerance": 0,   "unit": "h"},
    "coolingRateCph": {"max": 100,                       "unit": "℃/h"}
  },

  "mfg:ctqLevel": "CTQ",
  "mfg:inspectionMethod": "熱処理炉温度記録紙（チャート）確認",
  "mfg:linkedRequirement": {"@id": "#req-vibration-roof"}
}
```

---

## 6. よくある操作パターン

### 6.1 「JIS規格が改訂されたら何を見直すか」を調べる

**手順:**

```
1. 改訂された規格の @id を確認（例: #std-jis-e4054）

2. @graph を検索し、conformsTo に #std-jis-e4054 を含む ProductSpec を探す
   → #spec-rac35-v2 が該当

3. #spec-rac35-v2 の mfg:requirements[] のうち
   mfg:standardRef == #std-jis-e4054 のものを抽出
   → REQ-001（冷房能力）, REQ-002（COP）, REQ-005（騒音）, REQ-006（耐用年数）

4. 各 REQ の mfg:verifiedBy から影響する試験成績書を取得
   → TEST-COOL, TEST-NSE, TEST-END が再試験候補
```

### 6.2 「CADを改版したら再試験が必要か」を調べる

```
1. 改版する CADモデルの @id を確認（例: #cad-rac35-frame）

2. mfg:validatedBy を参照
   → [#test-vibration-2026]  ← 再試験必須

3. mfg:hasPmi[] の各 PMI の mfg:verifiedByPmi を収集
   → #test-vibration-2026  ← 同上

4. PMIの mfg:linkedRequirement を収集
   → REQ-003（振動耐久）  ← 影響する要求事項
```

### 6.3 「CTQ（全数検査）特性の一覧を出す」

```
1. 対象 CadModel（例: #cad-rac35-frame）を取得

2. mfg:hasPmi[] を走査し、mfg:ctqLevel == "CTQ" のものを抽出
   → #pmi-gt-001（平面度）
   → #pmi-gt-002（位置度）
   → #pmi-weld-001（溶接UT）

3. 各PMIの name と mfg:inspectionMethod を表として出力
```

### 6.4 「親寸法を変更したとき影響する試験は何か」を調べる

```
1. 変更する親寸法の @id を確認（例: #pmi-dim-frame-w）

2. mfg:drives を取得
   → [#pmi-dim-cond-w, #pmi-dim-evap-w, #pmi-dim-inv-pos]

3. 各子寸法の mfg:verifiedByPmi を収集
   → TEST-COOL（冷房性能）, TEST-INS（絶縁）

4. 影響する要求事項も mfg:linkedRequirement から収集
   → REQ-001（冷房能力）, REQ-004（絶縁抵抗）
```

### 6.5 「試験成績書が未対応の要求事項はあるか」を確認する

```
1. ProductSpec の mfg:requirements[] を走査

2. mfg:verifiedBy が null または
   参照先 TestReport の mfg:status != "approved" のものを抽出

3. 一覧表として出力 → リリース前の未了チェックに活用
```

---

## 7. 実務での運用フロー

### 7.1 新製品開発時の作成順序

```
Step 1: 標準規格エンティティを登録
        → 製品が準拠すべき JIS・ISO を @id で登録

Step 2: 製品仕様書エンティティを作成
        → conformsTo で Step 1 の規格を参照
        → mfg:requirements[] に要求事項を記述

Step 3: BOMエンティティを作成
        → derivedFrom で Step 2 の仕様書を参照
        → 各部品エントリを記述（CADリンクは後で追加可）

Step 4: CADモデルエンティティを作成
        → mfg:hasPmi[] に PMI を追加
        → BomEntry の mfg:linkedCadModel に @id を登録

Step 5: 試験成績書エンティティを作成
        → mfg:verifies で Step 2 の要求事項を参照
        → mfg:testConditions と mfg:measurements を記録

Step 6: 検査計画を生成
        → mfg:hasPmi[] の CTQ/major/minor を走査して自動生成
```

### 7.2 設計変更時の更新手順

```
Step 1: 変更対象エンティティの mfg:revision を上げる

Step 2: 変更内容に応じたリンクを更新・追加
        → CAD変更 → mfg:validatedBy を更新
        → 寸法変更 → mfg:changeHistory にエントリを追加
        → 要求変更 → mfg:verifiedBy をリセット（再試験必要フラグ）

Step 3: 影響波及チェック（6章の操作パターンを実行）

Step 4: 再試験が必要な TestReport を特定し
        mfg:status を "draft" に戻す（再試験後に "approved" に更新）

Step 5: RootDataset.hasPart に新エンティティが増えた場合は追記
```

### 7.3 バージョン管理の考え方

```
ドキュメント改版 → mfg:revision を上げる
                  → mfg:changeHistory に理由を記録

旧版の扱い:
  → mfg:status を "obsolete" に変更（削除しない）
  → 旧版 @id は残し、新版に別の @id を割り当てる

例: #spec-rac35-v2（最新版released） と
    #spec-rac35-v1（旧版obsolete）を共存させる
```

---

## 8. よくある質問（FAQ）

### Q1. JSON が書けなくても使えますか？

**A.** はい。`ro-crate-metadata.json` の編集には JSON 入力補助ツールや
スプレッドシート → JSON 変換ツールが使えます。
将来的には GUI 編集ツールの提供も検討してください。

### Q2. 既存の BOM（Excel）はどう扱いますか？

**A.** Excel ファイルを `bom/` フォルダに置き、`mfg:fileRef` で参照します。
同時に `mfg:entries[]` にも主要部品を JSON で記述することで、
グラフ検索（どの部品がどの CAD に対応するか）が可能になります。
最低限 `identifier` / `partNumber` / `linkedCadModel` だけでも有効です。

### Q3. CAD のネイティブ形式は CATIA / SolidWorks / NX のどれでも使えますか？

**A.** はい。`mfg:nativeFileRef` にネイティブファイルパスを記録し、
`mfg:fileRef` には共通フォーマットの STEP AP242 を指定してください。
STEP は PMI 情報（寸法・公差）も保持できます。

### Q4. PMI はすべての寸法を記述しなければいけませんか？

**A.** いいえ。すべての寸法を記述するのは現実的ではありません。
**CTQ（安全・機能に直結する特性）と親パラメータ（他の寸法を駆動するもの）**
を優先して記述し、残りは CAD ファイル本体を参照することを推奨します。

### Q5. 複数の製品間で共通部品を共有できますか？

**A.** はい。共通部品の BomEntry や CadModel に固有の `@id` を付与しておけば、
複数の BOM から同じ `@id` を参照することで共有を表現できます。

```json
{"@id": "#bom-entry-common-seal"}  ← 複数の BOM から参照可能
```

### Q6. 規格への適合証拠として法的に使えますか？

**A.** `ro-crate-metadata.json` はインデックスであり、証拠の実体は
`mfg:fileRef` が指す PDF・STEP・CSV 等のファイルです。
法的な証拠能力はこれらのファイルが持ちます。
BagIt 形式でパッケージ化することでチェックサムによる改ざん検知が可能です。

### Q7. スキーマのバリデーションはどうやって行いますか？

**A.** `schema/mfg-knowledge-schema.json` を使って
JSON Schema バリデーターで検証できます。

```bash
# ajv-cli を使う場合
ajv validate -s schema/mfg-knowledge-schema.json -d ro-crate-metadata.json
```

---

*製造業ナレッジデータ管理フォーマット v1.0.0*
*詳細仕様は `schema/schema-specification.md` を参照してください。*
