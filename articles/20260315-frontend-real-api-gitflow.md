---
title: "モックデータを卒業してリアルAPIへ〜IssueDock フロント実装とGit Flow API〜"
emoji: "🔌"
type: "tech"
topics: ["nuxtjs", "vue", "go", "github", "個人開発"]
published: false
---

今日は2日分の勢いで実装を進めました。
Vue（Nuxt.js）フロントのモックデータをすべて実APIに差し替え、
Go バックエンド側では新たに**リポジトリ管理API**と**Git Flow データAPI**を実装しました。

---

## 作っているシステム

**IssueDock** — GitHub の Projects / Issue / Branch / PR を一元管理するツールです。

- **サーバー：** Go（自作 REST API + GitHub GraphQL/REST）
- **フロントエンド：** Nuxt.js（Vue 3）
- **DB：** リポジトリ管理用テーブルを本日追加

昨日までに GitHub API 連携のバックエンド全体（Projects / Issue CRUD / Branch&PR 紐付け）が完成していたので、今日はいよいよフロントとの繋ぎ込みが中心でした。

---

## 今日やったこと

### 1. モックデータ → 実 API への全面切り替え（フロント）

`useProjects` コンポーザブルのモックデータを `useAsyncData` による実 API 呼び出しに置き換えました。
合わせて `api/github.ts` を新規作成し、API クライアントを一箇所に集約しています。

```ts
// app/api/github.ts
export async function fetchProjects(token: string): Promise<GithubProject[]> {
  const res = await $fetch<{ projects: GithubProject[] }>(
    `${apiUrl}/github/projects`,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return res.projects;
}
```

プロジェクト一覧・Issue 一覧・サマリー統計（Total Open / Closed This Week）すべてが実データで表示されるようになりました。

---

### 2. サマリー統計の N+1 問題を解消（フロント＋バックエンド連携）

当初の実装では「プロジェクト一覧取得 → 各プロジェクトの Issue 一覧を個別取得」という N+1 リクエストが発生していました。

**解決策：** バックエンド側（Go）の GraphQL クエリを修正し、`GET /github/projects` のレスポンスに `open_issues` / `closed_issues` カウントを含めるようにしました（Issue #25）。

```go
// GraphQL で items を取得し Go 側で open/closed を集計
for _, item := range project.Items.Nodes {
    if item.Content.State == "OPEN" {
        openCount++
    } else if item.Content.State == "CLOSED" {
        closedCount++
    }
}
```

これにより、フロントは Issue 一覧を個別に取得せず、プロジェクト一覧の1リクエストだけでサマリーを表示できるようになりました。

---

### 3. ページリロード後の GitHub 連携状態を維持（フロント）

`GET /auth/me` エンドポイントを新設し（Issue #24）、フロントではページロード時に `useAsyncData('auth-me')` で GitHub の連携状態とユーザー名を自動復元するようにしました。

```ts
// useGitHub.ts
const { data } = await useAsyncData("auth-me", () => fetchMe(token.value));
if (data.value?.github_connected) {
  connect(data.value.username);
}
```

以前はページリロードすると `isConnected` が `false` にリセットされていたのが、これで解消されました。

---

### 4. リポジトリ管理 API の実装（バックエンド）

プロジェクトに GitHub リポジトリを紐付ける機能を実装しました（Issue #29）。

新設テーブル：`project_repositories`

| エンドポイント | 概要 |
|---|---|
| `GET /projects/:id/repositories` | 紐付き済みリポジトリ一覧 |
| `POST /projects/:id/repositories` | リポジトリを紐付け |
| `DELETE /projects/:id/repositories/:repoId` | 紐付けを解除 |

POST 時は GitHub API でリポジトリの存在確認と重複チェックを行い、DB のユニーク制約違反は `409 Conflict` に変換しています。

---

### 5. Git Flow データ API の実装（バックエンド）

紐付けたリポジトリのブランチ・コミット履歴を Git Flow 形式で返す API を実装しました（Issue #30）。

```
GET /projects/:id/repositories/:repoId/git-flow
```

ブランチには以下の優先度を設定し、**複数ブランチに同じコミットが含まれる場合は優先度の高いブランチに割り当て**ます。

```
feature > hotfix > release > develop > main
```

ブランチ名のパターンマッチで自動分類し、Git Flow のビジュアライズに必要なデータを返します。

---

## 本日気づいたこと

### Nuxt の `useState` はコンポーネント間の状態共有に便利

サイドバーにプロジェクト名を表示する際、`useProject(projectId)` を AppSidebar 側でも呼ぶと初期値が `undefined` になってリアクティブに追従できない問題がありました。

`useCurrentProjectTitle` という composable を作り `useState` で状態を共有することで解決。
詳細ページで `project` が取得されたタイミングで `watch` で書き込む形にしました。

```ts
// useCurrentProjectTitle.ts
export const useCurrentProjectTitle = () =>
  useState<string>("current-project-title", () => "");
```

Nuxt の `useState` は SSR 対応のグローバルステートとして使えるので、
Vuex/Pinia を使わずにコンポーネント間で状態を渡せて便利です。

### N+1 はフロントではなくバックエンド側で解消する

プロジェクトの Issue カウントを「フロントで並列 fetch して集計」という実装にしていましたが、
当然ながらプロジェクト数が増えるほど重くなります。

「サーバー側の GraphQL クエリで一緒に取得してしまう」のが正解でした。
データ集計は API を呼ぶ側ではなく、**データを持っている側でやる**という基本を再確認しました。

### ブランチ名の URL エンコードは忘れがち

Git Flow API 実装中、スラッシュを含むブランチ名（`feature/xxx`）を GitHub API に渡すと 404 になるバグを踏みました。
`url.QueryEscape` でエンコードするだけなのですが、見落としがちなポイントでした。

---

## まとめ

| 項目 | 内容 |
|---|---|
| フロント | モックデータを実API全面切り替え、状態管理の改善 |
| バックエンド | `/auth/me`、Issue カウント集計、リポジトリ管理・Git Flow API |
| 気づき | N+1 はサーバー側で解消、Nuxt の useState 活用、ブランチ名エンコード |

モックを卒業して画面に実データが流れるようになると、開発のテンションが上がりますね。
次はリポジトリ紐付けとGit Flow のビジュアライズ画面をフロントに実装していく予定です。
