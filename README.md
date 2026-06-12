# BIZ

## 概要

このリポジトリはビジネス向けアプリケーションのソースコードを管理します。

---

## セットアップ手順

### 前提条件

- Git
- Node.js 18 以上（または使用言語のランタイム）
- npm / yarn / pnpm のいずれか

### 1. リポジトリのクローン

```bash
git clone https://github.com/kikuchi08022697209-glitch/BIZ.git
cd BIZ
```

### 2. 依存パッケージのインストール

```bash
npm install
```

### 3. 環境変数の設定

`.env.example` をコピーして `.env` を作成し、各値を設定してください。

```bash
cp .env.example .env
```

`.env` の主な設定項目:

| 変数名 | 説明 | 例 |
|--------|------|-----|
| `DATABASE_URL` | データベース接続文字列 | `postgresql://user:pass@localhost:5432/biz` |
| `PORT` | サーバーのポート番号 | `3000` |
| `NODE_ENV` | 実行環境 | `development` |

### 4. 開発サーバーの起動

```bash
npm run dev
```

### 5. ビルド（本番用）

```bash
npm run build
npm start
```

---

## テスト

```bash
# ユニットテスト
npm test

# ウォッチモード
npm run test:watch
```

---

## ブランチ戦略

| ブランチ | 用途 |
|----------|------|
| `main` | 本番リリース |
| `develop` | 開発統合ブランチ |
| `feature/*` | 機能開発 |
| `fix/*` | バグ修正 |

---

## ライセンス

[MIT](LICENSE)
