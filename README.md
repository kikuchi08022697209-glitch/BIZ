# BIZ — 製造業ナレッジデータ管理手法

設計ドキュメントを **グラフ（ノードとエッジのネットワーク）** として管理し、
設計変更・規格改訂・品質不具合の発生時に、その影響波及範囲を即座に追跡できる
ナレッジデータ管理フォーマットの **仕様書・JSONスキーマ・解説ガイド** を収録したリポジトリです。

学術研究データ管理標準である [RO-Crate](https://www.researchobject.org/ro-crate/) の思想を製造業に転用し、
JSON-LD 型のドキュメントリンク管理を実現します。

> **注意:** 本リポジトリは仕様・スキーマ・ドキュメントを管理するものであり、
> 実行可能なアプリケーションのソースコードは含みません。

---

## 解決する課題

製造現場でよく起きる、ドキュメント間のつながりが見えないことに起因する困りごとを解決します。

| 従来の課題 | 本手法による解決 |
|---|---|
| ドキュメント間の依存関係が担当者の頭の中だけにある | `@id` による明示的リンクでグラフ化 |
| 規格改訂時に何が影響するか即座にわからない | `conformsTo` リンクを逆引きして影響先を列挙 |
| 試験成績書がどの要求事項を検証しているか不明 | `verifies` リンクで要求事項と試験を直結 |
| CAD変更時に再試験が必要か判断できない | `validatedBy` リンクで影響試験を自動抽出 |
| PMI（公差・表面粗さ）と検査計画が別管理 | PMIエンティティを CAD から `hasPmi` でリンク |

---

## 対象ドキュメント種別

製造業における5大ドキュメント種別を相互リンクで管理します。

| ドキュメント種別 | 略称 |
|---|---|
| 標準規格 | Standard |
| 製品仕様書 | ProductSpec |
| 部品表 | BOM |
| 3D CADモデル（PMI含む） | CadModel |
| 開発試験成績書 | TestReport |

---

## リポジトリ構成

| パス | 内容 |
|------|------|
| [`manufacturing-knowledge-management.md`](manufacturing-knowledge-management.md) | 手法の本仕様書。設計思想・エンティティ定義・リンク述語・PMI連携・適用事例・クエリパターンを解説 |
| [`schema/mfg-knowledge-schema.json`](schema/mfg-knowledge-schema.json) | フォーマットを検証するための JSON スキーマ（JSON Schema draft 2020-12, v1.0.0） |
| [`schema/schema-specification.md`](schema/schema-specification.md) | スキーマの逐条解説。各エンティティ型・PMIアノテーション型・列挙値・バリデーションルールのリファレンス |
| [`docs/guide.md`](docs/guide.md) | 設計者・品質管理者・製造技術者・IT担当者向けの解説ガイド。具体的な書き方と運用フロー |

---

## はじめに読むもの

目的に応じて、以下の順に参照することを推奨します。

1. **全体像を知りたい** → [`docs/guide.md`](docs/guide.md)（やさしい解説ガイド）
2. **仕様を正確に把握したい** → [`manufacturing-knowledge-management.md`](manufacturing-knowledge-management.md)（本仕様書）
3. **データを書く・検証する** → [`schema/schema-specification.md`](schema/schema-specification.md) と [`schema/mfg-knowledge-schema.json`](schema/mfg-knowledge-schema.json)

---

## データの検証方法

作成した JSON-LD データは、付属の JSON スキーマで検証できます。
（[Ajv](https://ajv.js.org/) などの JSON Schema draft 2020-12 対応バリデータを利用してください。）

```bash
# 例: ajv-cli を使った検証
npx ajv-cli validate \
  --spec=draft2020 \
  -s schema/mfg-knowledge-schema.json \
  -d your-data.jsonld
```

バリデーションルールの詳細（必須チェック・IDフォーマット・参照整合性）は
[`schema/schema-specification.md`](schema/schema-specification.md) の「6. バリデーションルール」を参照してください。
