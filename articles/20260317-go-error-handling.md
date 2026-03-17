---
title: "Go初心者へ、エラーハンドリングの基本と落とし穴"
emoji: "🐹"
type: "tech"
topics: ["go", "初心者", "エラーハンドリング"]
published: false
---

今日は作業を休んだので、いつもの作業記事ではなく、個人開発を通じて身についたことを書こうと思います。

テーマは **Goのエラーハンドリング** です。

入門書で構文を覚えた後、実際にコードを書き始めると「どうエラーを扱えばいいんだろう」と迷う場面が多くありました。自分がつまずいたポイントを中心に整理します。

---

## Goのエラーハンドリングは「戻り値」で行う

他の言語（JavaやPythonなど）では例外（Exception）を `try/catch` で捕まえる方式が一般的ですが、Goには例外機構がありません。

代わりに、**エラーを関数の戻り値として返す** のがGoの基本スタイルです。

```go
func fetchUser(id int) (User, error) {
    if id <= 0 {
        return User{}, fmt.Errorf("invalid id: %d", id)
    }
    // ...
    return user, nil
}
```

呼び出し側は必ずエラーをチェックします。

```go
user, err := fetchUser(42)
if err != nil {
    // エラー処理
    return err
}
// 正常系の処理
```

最初は `if err != nil` だらけのコードに違和感を覚えます。でもこれはGoが意図的に選んだ設計で、「エラーを見えないところに隠さない」という思想から来ています。

---

## エラーを無視しない癖をつける

`_` でエラーを捨てることができてしまうため、慣れないうちはついやってしまいます。

```go
// NG: エラーを無視している
data, _ := json.Marshal(payload)

// OK: 確認する
data, err := json.Marshal(payload)
if err != nil {
    return fmt.Errorf("marshal failed: %w", err)
}
```

`_` を使っていい場面は「本当にエラーが起きないことが保証されている」か「エラーが起きても問題ない」ときだけです。外部ライブラリや入出力を伴う処理ではほぼ必ず確認すべきです。

---

## エラーをラップして文脈を伝える

`fmt.Errorf` に `%w` を使うと、元のエラーを包んだ（ラップした）新しいエラーを作れます。

```go
func readConfig(path string) (Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return Config{}, fmt.Errorf("readConfig: %w", err)
    }
    // ...
}
```

呼び出しが深くなってもどこで何が起きたか追いやすくなります。エラーメッセージに「どの関数で」「何をしようとして」失敗したかを書いておくと、デバッグのときに助かります。

ラップしたエラーは `errors.Is` や `errors.As` で元のエラーを取り出せます。

```go
err := readConfig("config.json")

// 元のエラーがos.ErrNotExistかチェック
if errors.Is(err, os.ErrNotExist) {
    fmt.Println("設定ファイルが見つかりません")
}
```

---

## HTTPサーバーでのエラーの扱い方

APIサーバーを作ると、エラーを「どのHTTPステータスコードで返すか」という判断が増えます。

Goの `net/http` では `http.Error` でレスポンスを返せます。

```go
func handleGetUser(w http.ResponseWriter, r *http.Request) {
    id, err := strconv.Atoi(r.PathValue("id"))
    if err != nil {
        http.Error(w, "invalid id", http.StatusBadRequest) // 400
        return
    }

    user, err := fetchUser(id)
    if err != nil {
        http.Error(w, "internal error", http.StatusInternalServerError) // 500
        return
    }

    json.NewEncoder(w).Encode(user)
}
```

ポイントは `http.Error` の後に **必ず `return` を書くこと** です。`return` を忘れると、エラーレスポンスを返した後も処理が続いてしまいます。

---

## まとめ

Goのエラーハンドリングで意識してほしいことを3つに絞るとこうなります。

**エラーは必ず確認する**。`_` で捨てたくなる気持ちはわかりますが、実際の開発では原因不明のバグの温床になります。

**`%w` でラップして文脈を残す**。どこで何が起きたか分かるエラーメッセージは、後からコードを読む自分（や他の人）を助けてくれます。

**`return` を忘れない**。HTTPハンドラでは `http.Error` の後の `return` が抜けると気づきにくいバグになります。

最初は冗長に感じますが、慣れてくるとこのスタイルがむしろ読みやすいと感じるようになります。Goを書き始めたばかりの方の参考になれば幸いです。
