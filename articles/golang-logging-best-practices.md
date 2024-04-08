---
title: "Go言語のログに関するベストプラクティス"
emoji: "🔖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Golang"]
published: true
---

# `log/slog`を使う

[`log/slog`](https://pkg.go.dev/log/slog)は構造化ログを提供し、ログへのレベル設定やKey-Valueの追加を行えます。

Go1.20以前では、[`log`](https://pkg.go.dev/log)による非構造化ログしかなく、構造化ログを使うには[github.com/uber-go/zap](https://github.com/uber-go/zap)などを使うしかありませんでした。そのため非標準パッケージ側が構造化ログに対応するため、一貫性のないInterfaceがいくつも作られたことで開発上のコストになっていました。

標準パッケージである`log/slog`を使う文化が浸透すれば、外部パッケージへ特にロガーを渡すことなく一貫したログ構造で出力できるようになるでしょう。

# 適切なレベルを使う

`log/slog`は4つのレベル（`DEBUG`,`INFO`,`WARN`,`ERROR`）を提供しています。以下の指針で使い分けます。

- `DEBUG`: 普段の本番運用時には出力せず、障害調査時に必要になりそうなものを記録します。
- `INFO`: ユーザーやシステムが行ったことを記録します。個人的には、このログだけで障害の調査から修正までできることを理想として記録しています。そうしておけば、わざわざ障害発生後にDEBUGログを有効にして再現させる必要が無くなります。
- `WARN`: エラーになる可能性のある箇所を記録します。例えば入力サイズによってOutOfMemoryを引き起こす可能性があるとき、WARNログを出力しておけば障害発生前の兆候を掴む手がかりになります。
- `ERROR`: 開発者が修正するべきエラーを記録します。HTTPでいえば5xx系のエラーです。あくまで統一が大事ですが、開発者の責任ではない4xx系のものは`INFO`にしておくことで、`ERROR`ログは全て修正するべきであるという指針が定まります。

# ログのメッセージは動的にしない

例えばユーザーがログインした際のログを出力する場合、前者より後者が好ましいです。

```go
userID := 123
userName := "foo"

// {"time":"...","level":"INFO","msg":"User 123 (foo) logged in"}
slog.Info(fmt.Sprintf("User %d (%s) logged in", userID, userName)) // bad

// {"time":"...","level":"INFO","msg":"User logged in","userID":"123","userName":"foo"}
slog.Info("User logged in",
  slog.String("userID", fmt.Sprintf("%d", userID)),
  slog.String("userName", userName)) // good
```

動的な箇所はKey-Valueとして出力することで、ログを解析しやすくなります。例えばjqコマンドでは以下のようにログをフィルタリングして出力できます。

- ログイン時のログのみを出力: `jq '.[] | select(.msg == "User logged in")'`
- userIDが1のログのみを出力: `jq '.[] | select(.userID == "1")'`

# TraceIDを付加する

ひとつのHTTPリクエストのログを全て見たい時、同一のTraceIDがKey-Valueとして設定されてあると便利です。
ここでは`log/slog`での実装方法を紹介します。まずどこかの始点で`context.Context`にTraceIDを持たせます。

```go
ctx = context.WithValue(ctx, "trace_id", uuid.New().String())
```

次に`slog.Handler`側を実装し、`Handle()`でTraceIDがあれば自動でKey-Valueを追加するようにします。

```go
type TraceIDLogHandler struct {
	slog.Handler
}

func NewTraceIDLogHandler(h slog.Handler) *TraceIDLogHandler {
	return &TraceIDLogHandler{Handler: h}
}

func (h *TraceIDLogHandler) Handle(ctx context.Context, r slog.Record) error {
	if v, ok := ctx.Value("trace_id").(string); ok {
		r.AddAttrs(slog.String("trace_id", v))
	}
	return h.Handler.Handle(ctx, r)
}
```

あとはログ時の`context.Context`を渡すと`trace_id`が追加され、一連の処理を追えるようになります。

```go
// {"time":"xxx","level":"INFO","msg":"Program started","trace_id":"874e2db1-25c6-43df-b386-c25fdd51555f"}
slog.InfoContext(ctx, "Program started")

// {"time":"xxx","level":"INFO","msg":"Program finished","trace_id":"874e2db1-25c6-43df-b386-c25fdd51555f"}
slog.InfoContext(ctx, "Program finished")
```

# 読み手の視点になって過不足なく

基本的にログの用途は障害時の調査のはずです。よって「いつ」「どこで」「だれに」「なにが」「なぜ」「どのように」起こったかを伝えることを意識します。
開発者であれば、何らかの障害が発生しそうな箇所はある程度目をつけられるはずです。その調査に必要そうな情報を過不足なく伝えることが目指すべきログの姿だと考えます。


# 参考文献

- [Logging Best Practices: The 13 You Should Know](https://www.dataset.com/blog/the-10-commandments-of-logging/)
