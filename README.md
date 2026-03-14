# note-articles

note に投稿するエンジニア記事を管理するリポジトリです。

## ディレクトリ構成

```
note-articles/
├── README.md
└── YYYY-MM-DD_<slug>.md   # 記事ファイル（日付_スラッグ）
```

## 記事一覧

| 日付 | タイトル | note URL |
|------|---------|----------|
| 2026-03-14 | [GitHub API統合バックエンドを一気に実装した話](./2026-03-14_github-api-implementation.md) | [Zenn](https://zenn.dev/shintyakos/articles/20260314-github-api-implementation) / [note](https://note.com/deep_harte300/n/n7153b179790c) |

## noteへの投稿手順

1. 記事Markdownを作成・編集してコミット
2. [note.com](https://note.com) にログイン
3. 「投稿」→「テキスト」を選択
4. Markdownの内容をコピーして貼り付け（noteはMarkdown記法を一部サポート）
5. 投稿後、上記の記事一覧にnote URLを追記してコミット

> **注意:** note公式のMarkdown投稿APIは現時点で一般公開されていないため、
> 手動での貼り付け投稿となります。
