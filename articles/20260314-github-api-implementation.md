---
title: "GitHub API統合バックエンドを一気に実装した話 〜Projects・Issue CRUD・ブランチ連携〜"
emoji: "🐙"
type: "tech"
topics: ["go", "github", "graphql", "個人開発", "api"]
published: true
---

今日はGoで開発しているバックエンドサーバーに、GitHub API関連の機能をまとめて実装しました。
実装量が多かったので、何を作ったか・どんな点に詰まったかを振り返ります。

---

## 今日実装した機能

### 1. GitHub Projects一覧取得 API

**エンドポイント：** `GET /github/projects`

GitHub GraphQL API（Projects v2）を使って、ユーザーのプロジェクト一覧を取得する機能です。

ポイントは、DBに暗号化して保存しているアクセストークンを復号してGitHub APIを呼び出す部分。
`GithubAccountRepository` に `GetGitHubAccount` メソッドを追加し、トークンの復号処理もここに集約しました。

OAuthスコープには `read:project` を追加しています。

```go
// GitHub GraphQL でプロジェクト一覧を取得
query := `
  query {
    viewer {
      projectsV2(first: 20) {
        nodes {
          id
          title
          number
          url
        }
      }
    }
  }
`
```

---

### 2. プロジェクトのIssue一覧取得 API

**エンドポイント：** `GET /github/projects/{projectId}/issues`

Projects v2のプロジェクトに紐づくIssue一覧を取得します。
プロジェクトのアイテムにはPRも含まれるため、**Issueのみをフィルタリング**してレスポンスに返すようにしました。

`GitHubIssue` モデルを新設し、ID・番号・タイトル・本文・状態・URL・担当者・ラベルを保持します。

**詰まった点：** プライベートリポジトリのIssueを取得したとき、`content` フィールドが `null` で返ってくる問題が発生。
原因は OAuthスコープが `read:project` のみだったこと。`repo` スコープを追加することで解決しました。

---

### 3. Issue CRUD API

**エンドポイント：** `PATCH /github/repos/{owner}/{repo}/issues/{issueNumber}`

Issueのタイトル・本文・状態（open/closed）・担当者・ラベルを更新するAPIです。
GitHub REST API の Issues Update エンドポイントをラップしています。

また、**認証処理を `verifyBearer` として共通化**しました。
各ハンドラで `Authorization: Bearer <token>` を個別にパースしていたのを一箇所にまとめたことで、コードがすっきりしました。

---

### 4. Issueへのブランチ・PR紐付け API

**エンドポイント：**
- `POST /github/repos/{owner}/{repo}/issues/{issueNumber}/linked-branch`
- `POST /github/repos/{owner}/{repo}/issues/{issueNumber}/linked-pr`

#### ブランチ紐付け
GitHub GraphQL の `createLinkedBranch` を使い、Issueの開発パネルにブランチを直接作成して紐付けます。
GitHubのWebUI上でIssueを開いたときに「Development」パネルにブランチが表示されるようになります。

#### PR紐付け
PRの本文に `closes #{issueNumber}` を追記することで、IssueとPRをリンクします。
GitHubのキーワード機能（closes/fixes/resolves）を活用したシンプルな実装です。

---

### 5. プロジェクトへのアイテム追加 API

**エンドポイント：** `POST /github/projects/{projectId}/items`

GitHub GraphQL の `addProjectV2ItemById` ミューテーションを使い、IssueやPRをプロジェクトボードに追加します。
IssueにブランチとPRが紐付いていれば、プロジェクトボード上でブランチの進捗状態も追跡できます。

---

### 6. API仕様書の作成

**ファイル：** `doc/api.md`

今日実装した含む全11エンドポイントの仕様書を作成しました。
各エンドポイントのリクエスト・レスポンス・エラーの詳細に加え、フロントエンド向けに**認証フロー**と**典型的な利用フロー**も追記しています。

---

## 今日の実装を振り返って

### GitHub GraphQL vs REST API の使い分け

今回の実装で改めて整理できたのは、**Projects v2はGraphQL専用**という点です。

| 操作 | API |
|------|-----|
| プロジェクト一覧取得 | GraphQL |
| プロジェクトへのアイテム追加 | GraphQL (mutation) |
| ブランチ作成・Issue紐付け | GraphQL (mutation) |
| Issue CRUD | REST API |
| PR操作 | REST API |

Projects v2はREST APIに対応していないため、GraphQLの使い方に慣れることが必要でした。

### OAuthスコープの罠

プライベートリポジトリのIssueを取得する際に `content` が `null` になる問題は、
最初はGraphQLクエリの書き方を疑っていました。
しかし実際の原因はスコープ不足で、`repo` を追加するだけで解決。

スコープは必要最小限にしたい気持ちはありますが、プライベートリポジトリへのアクセスには `repo` が必要だと理解しました。

---

## まとめ

今日は GitHub の Projects v2 / Issue / Branch / PR を連携させる一連のAPIを実装しました。
GraphQL APIを多用する日になりましたが、柔軟にデータを取得・操作できる点は便利でした。

次のステップはフロントエンドとの繋ぎ込みです。今日作成したAPI仕様書をベースに進める予定です。
