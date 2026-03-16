---
title: "Git Flowをブラウザに描く〜IssueDock ビジュアライズ実装とIssue管理UI〜"
emoji: "🌿"
type: "tech"
topics: ["nuxtjs", "vue", "go", "github", "個人開発"]
published: false
---

昨日はモックデータを実APIに切り替えた。
今日はいよいよ**Git Flow のビジュアライズ画面**を実装し、あわせて Issue 管理UIも大幅に強化しました。

---

## 作っているシステム

**IssueDock** — GitHub の Projects / Issue / Branch / PR を一元管理するツールです。

- **サーバー：** Go（自作 REST API + GitHub GraphQL/REST）
- **フロントエンド：** Nuxt.js（Vue 3）

---

## 今日やったこと

### 1. Git Flow ビジュアライズ画面の実装（フロント）

プロジェクトに紐付けたリポジトリの**ブランチ・コミット履歴を Git Flow 形式でグラフ描画**する画面を実装しました。

データソースは以下の3つの API を組み合わせています。

| API | 用途 |
|---|---|
| `GET .../branches` | ブランチ一覧と色コード |
| `GET .../branches/{name}/commits` | ブランチごとのコミット一覧 |
| `GET .../pulls` | PR一覧（マージコミットとの照合に使用） |

デフォルトブランチをレーン0（最上部）に固定し、feature/hotfix/release/develop/main の優先度順でレーンを割り当てています。

#### ブランチ間のマージ線（ベジェ曲線）描画

コミットグラフで難しいのが**マージ線の描画**です。
バックエンドで計算した `fork_points`（どのコミット時点でブランチが分岐したか）を使い、featureブランチの最終コミット → マージコミット間をベジェ曲線で結んでいます。

```ts
// forkConnectors: fork_pointsをもとにベジェ曲線の制御点を計算
const forkConnectors = computed(() => {
  return (props.forkPoints ?? []).flatMap((fp) => {
    const fromLane = laneIndex(fp.branch);
    const toLane = laneIndex(fp.base_branch);
    const fromCommit = commitPosition(fp.branch, fp.base_commit_hash);
    // 制御点をレーン間の中点に設定してS字カーブを描く
    return [{ from: fromCommit, to: mergePoint, ctrl: midPoint }];
  });
});
```

#### 削除済みブランチの扱い

マージ済みで削除されたブランチは `/branches` API には存在しませんが、
PR の `head_branch` 情報には残っています。
そこでフロント側では**PR履歴から全ブランチ名を収集**し、存在するブランチと合わせて表示するようにしました。

削除済みブランチには色コードがないため、ブランチ名のプレフィックス（`feature/`、`hotfix/` など）でパターンマッチして色を自動割り当てしています。

---

### 2. Git Flow 用 API の追加（バックエンド）

フロントのビジュアライズを支えるため、バックエンドに以下のAPIを追加しました。

#### ブランチ関連 API（Issue #34 #35）

```
GET /projects/{id}/repositories/{repoId}/branches
GET /projects/{id}/repositories/{repoId}/branches/{branchName}/commits
```

ブランチ一覧は色コード付き、コミット一覧は `limit` クエリパラメータで件数指定可能です。
また、削除済みブランチを `/branches/{name}/commits` で取得しようとした際に GitHub API が `422` を返すケースがあり、これを `404` に変換してフロントに伝えるようにしました。

#### PR 一覧 API（Issue #37）

```
GET /projects/{id}/repositories/{repoId}/pulls
```

OPEN と マージ済みPR を取得し、各 PR にコミット一覧を含めて返します。
`since` / `until` 未指定時は最新30件、指定時は期間内全件を返す仕様にしました。

#### fork_points の計算（Issue #32）

Git Flow のマージ線を描画するには「どのコミット地点でブランチが分岐したか」が必要です。
GitHub Compare API（`GET /repos/{owner}/{repo}/compare/{base}...{head}`）でマージベースコミットを取得し、`fork_points` としてレスポンスに含めるようにしました。

```go
// GitHub Compare API でマージベースを取得
func fetchMergeBase(owner, repo, base, head string) (string, error) {
    url := fmt.Sprintf(".../compare/%s...%s", base, head)
    // merge_base_commit.sha を返す
}
```

---

### 3. Issue 管理 UI の大幅強化（フロント）

#### New Issue 作成モーダル（Issue #13）

Issue を画面から作成できるモーダルを実装しました。

- **リポジトリ選択** → そのリポジトリのラベル一覧を自動取得してドロップダウン表示
- **担当者・優先度・ラベル**を入力可能
- Issue 作成後にプロジェクトへの自動追加まで一気通貫で処理

リポジトリ切り替え時にラベル一覧を再取得し、選択済みラベルもリセットする細かい配慮も入れています。

#### 全 Issue 画面（/issues）の新設（Issue #21）

ダッシュボードは「Open Issue のみ」に絞り、すべての Issue を見たい場合は `/issues` 画面に遷移する構成にしました。

`/issues` には **Open / Closed / All タブ**と**リポジトリ絞り込み**・**キーワード検索**フィルターを実装。Issue テーブルには Priority 列も追加しています。

---

### 4. ラベル CRUD API の追加（バックエンド、Issue #43）

フロントの Issue 作成モーダルでラベルを選択式にするため、ラベル管理 API を実装しました。

| エンドポイント | 概要 |
|---|---|
| `GET /github/repos/{owner}/{repo}/labels` | ラベル一覧（ページネーション対応） |
| `POST /github/repos/{owner}/{repo}/labels` | ラベル作成 |
| `PATCH /github/repos/{owner}/{repo}/labels/{name}` | ラベル更新 |
| `DELETE /github/repos/{owner}/{repo}/labels/{name}` | ラベル削除 |

---

## 本日気づいたこと

### ループ内の `defer` はリソースリークになる

ページネーションでループを回しながら HTTP レスポンスボディを `defer resp.Body.Close()` していたコードを PR レビューで指摘されました。

`defer` はその**関数が終了するまで実行されない**ため、ループを繰り返すたびにボディがクローズされずメモリが積み上がります。
ループ内では `defer` を使わず、処理後に即時 `Close` するのが正解です。

```go
// NG: ループ内 defer
for {
    resp, _ := http.Get(url)
    defer resp.Body.Close() // ループが終わるまで閉じられない
}

// OK: 即時 Close
for {
    resp, _ := http.Get(url)
    data, _ := io.ReadAll(resp.Body)
    resp.Body.Close() // 即時クローズ
}
```

### URLエンコードは `QueryEscape` と `PathEscape` を使い分ける

昨日はブランチ名の `QueryEscape` 忘れを踏みましたが、今日はラベル名の `PathEscape` 忘れを踏みました。

- `url.QueryEscape` → クエリパラメータ用（スペースを `+` に変換）
- `url.PathEscape` → パスセグメント用（スペースを `%20` に変換、`/` はエンコードしない）

ラベル名はパスに含まれるので `PathEscape` が正解。似ているので混同しがちです。

### 削除済みブランチはバックエンドとフロント両側で考慮が必要

削除済みブランチへの API アクセスはバックエンドで `404` に変換しましたが、
フロントのビジュアライズでは「PR 履歴には残っているが、ブランチ API からは消えている」状態を別途ハンドリングする必要がありました。
バックエンドでのエラーハンドリングとフロントのデータ補完、両方のレイヤーで考慮する必要があると改めて感じました。

---

## まとめ

| 項目 | 内容 |
|---|---|
| Git Flow画面 | ブランチ・コミット・PRをベジェ曲線で描画、削除済みブランチも対応 |
| バックエンド | ブランチ・PR・ラベルCRUD・fork_points計算API |
| Issue管理 | 作成モーダル、全Issue画面（Open/Closed/All タブ、リポジトリ絞り込み） |
| 気づき | ループ内defer・PathEscape vs QueryEscape・削除済みブランチの二重管理 |

Git Flow のグラフが実際のリポジトリデータで表示されたときは達成感がありました。
次はプロジェクト設定画面のリポジトリ紐付けUIやダッシュボードのさらなる充実を進める予定です。
