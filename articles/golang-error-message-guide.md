---
title: "Go言語のエラーメッセージの書き方"
emoji: "💭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Golang"]
published: true
---

Go言語で`error`型のメッセージを書く指針をいくつかのスタイルガイドを参考にまとめます。

# `failed to`などの定型句は不要

https://github.com/uber-go/guide/blob/6faf78242fbb4c4296861eff65b6cfa274acaa87/style.md#error-wrapping

`error`型である以上、何かに失敗したメッセージであることは明白です。

```go
// bad
s, err := store.New()
if err != nil {
    return fmt.Errorf("failed to create new store: %w", err)
}

// good
s, err := store.New()
if err != nil {
    return fmt.Errorf("new store: %w", err)
}
```

前者は`failed to x: failed to y: failed to create new store: the error`のように出力され冗長です。後者は`x: y: new store: the error`のように出力されます。

# 重複なくエラーに情報を追加する

https://github.com/google/styleguide/blob/8487c083e1faecb1259be8a8873618cfdb69d33d/go/best-practices.md?plain=1#L863

基本的にGo言語では呼び出した関数からのエラーを伝播させていきます。その際に重複した情報を追加してはいけません。

例えば標準パッケージの`os.Open()`でファイルを開けなかった場合、エラーには既にファイル名が含まれているため、追加する必要はありません。

```go
// Bad:
if err := os.Open("settings.txt"); err != nil {
    // Output: could not open settings.txt: open settings.txt: no such file or directory
    return fmt.Errorf("could not open settings.txt: %w", err)
}

// Good:
if err := os.Open("settings.txt"); err != nil {
    // Output: open settings.txt: no such file or directory
    return err
}
```

またエラーの背景についての情報を追加することが推奨されます。
下記例では`settings.txt`ファイルを開けなかったことによって「プログラムの起動コードが動作していない」情報を追加します。

```go
// Good:
if err := os.Open("settings.txt"); err != nil {
    // launch codes unavailable: open settings.txt: no such file or directory
    return fmt.Errorf("launch codes unavailable: %v", err)
}
```
