---
name: sync-kintone-api
description: kintone APIアップデート情報からREST APIの変更を調査し、rest-api-clientへの反映を行うスキル
---

# kintone API更新追従ワークフロー

kintone REST APIの更新情報を調査し、`packages/rest-api-client` に反映する一連の作業を実行する。

**重要: このワークフローはステップ1〜6をすべて最後まで実行すること。途中で止まらず、コミット・PR作成まで完了させる。**

## 入力

ユーザーから以下のいずれかの形式で入力を受け取る：

- **期間指定**: 「2025年1月〜3月のAPI更新を反映して」→ ステップ1からWebFetchで取得
- **更新内容の直接指定**: 「以下のAPI変更を反映して: ...」→ ステップ1・2をスキップし、指定内容をもとにステップ3から開始

## 1. API更新情報の取得

- `https://cybozu.dev/ja/kintone/news/api-updates/` からユーザー指定期間のREST API更新を抽出する
- WebFetchで各更新の詳細ページを取得し、変更内容を把握する
- REST API以外の変更（JavaScript API等）はスコープ外のため除外する
- **WebFetchが失敗する場合**: ユーザーに更新内容のテキスト提供を依頼し、提供された内容をもとにステップ3へ進む

## 2. 公式APIドキュメントの参照

- `https://cybozu.dev/ja/kintone/docs/rest-api/` から該当APIの最新仕様を取得する
- リクエスト・レスポンスのプロパティ名・型・必須/任意を正確に把握する
- 既存コードのプロパティ名やコーディング規約と齟齬がないか確認する
- **WebFetchが失敗する場合**: ステップ1で取得済みの情報とコードベースの既存パターンをもとに進める

## 3. コードベースとの差分調査

変更の種類を以下のように分類し、差分を一覧化する：

| 変更種別 | 対象ファイルパス |
|---|---|
| フィールド型定義の追加・変更 | `packages/rest-api-client/src/KintoneFields/types/field.ts`, `property.ts`, `layout.ts`, `fieldLayout.ts` |
| フィールド型チェック | `packages/rest-api-client/src/KintoneFields/types/__checks__/` |
| クライアント型の追加・変更 | `packages/rest-api-client/src/client/types/` 配下（`app/`, `record/`, `space/`, `plugin/`） |
| 新規APIメソッドの追加 | `packages/rest-api-client/src/client/`（`AppClient.ts`, `RecordClient.ts`, `SpaceClient.ts`, `PluginClient.ts`, `FileClient.ts`） |
| テストの追加・更新 | `packages/rest-api-client/src/client/__tests__/` |

差分を確認する際は以下を実施：
- Grep/Globで既存の型定義やメソッドを検索
- 公式ドキュメントとの差分を特定
- **対話環境の場合**: 修正を始める前に、変更内容の一覧をユーザーに提示して確認を取る
- **非対話環境の場合（GitHub Actions等）**: 変更内容の一覧をログに出力し、そのまま修正に進む

## 4. 修正

- 型定義の追加・変更（プロパティの追加、optional/requiredの変更など）
- 必要に応じてクライアントメソッドの追加・変更
- テストフィクスチャの更新・追加
- 既存コードのパターンや命名規約に従う

## 5. 検証

`packages/rest-api-client` ディレクトリで以下を順番に実行する：

```bash
cd packages/rest-api-client && pnpm build
```

```bash
cd packages/rest-api-client && pnpm lint
# エラーがあれば:
cd packages/rest-api-client && pnpm fix
```

```bash
cd packages/rest-api-client && pnpm test
```

ルートディレクトリでフォーマットチェック：

```bash
pnpm lint:prettier
# エラーがあれば:
pnpm fix:prettier
```

## 6. コミットとPR作成

**検証が通ったら、ユーザーの確認を待たずにコミットとPR作成を実行する。**

### 6a. ブランチ作成とコミット

1. `feat/rest-api-client-YYYY-MM-api-updates` の形式でブランチを作成する（まだ作成していない場合）
2. 変更したファイルをステージングしてコミットする
   - コミットメッセージの例: `feat(rest-api-client): add missing properties for YYYY-MM-DD API update`
3. リモートにプッシュする

### 6b. PR作成

1. `.github/PULL_REQUEST_TEMPLATE.md` のフォーマット（Why / What / How to test / Checklist）に従ってPRを作成する
2. `gh pr create` を使用する。fork先リポジトリへのPR作成で認証エラーが出る場合は `gh api repos/{owner}/{repo}/pulls` を使用する（SAML認証回避）
3. **作成したPRのURLをユーザーに報告して完了とする**
