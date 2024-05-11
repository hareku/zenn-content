---
title: "Go言語のtime.Tickerでスキップされる際の挙動"
emoji: "⌚️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Golang"]
published: true
---

結論：Tick間に`Duration*2`時間経過した場合、Tickが1つスキップされる。

`time.Ticker`を使い、1秒毎に経過時間を3回表示するプログラムにおいて、待ち時間による挙動の違いを示します。まず待ち時間が無いコードは以下です。

https://go.dev/play/p/_9YSV4aFJ82

```go
func main() {
	begin := time.Now() // for debug

	t := time.NewTicker(time.Second)
	defer t.Stop()

	for i := 1; i <= 3; i++ {
		<-t.C
		fmt.Printf("tick(%d): %v\n", i, time.Since(begin))
	}
}
```

```
tick(1): 1s
tick(2): 2s
tick(3): 3s
```

# 待ち時間が`Duration*1.9`の場合

https://go.dev/play/p/IavryD8xYmv

```go
func main() {
	begin := time.Now() // for debug

	t := time.NewTicker(time.Second)
	defer t.Stop()

	for i := 1; i <= 3; i++ {
		<-t.C
		fmt.Printf("tick(%d): %v\n", i, time.Since(begin))
		if i == 1 {
			time.Sleep(time.Millisecond * 1900)
		}
	}
}
```

```
tick(1): 1s
tick(2): 2.9s
tick(3): 3s
```

# 待ち時間が`Duration*2.1`の場合

https://go.dev/play/p/JYUgEj9yIHU

```go
func main() {
	begin := time.Now() // for debug

	t := time.NewTicker(time.Second)
	defer t.Stop()

	for i := 1; i <= 3; i++ {
		<-t.C
		fmt.Printf("tick(%d): %v\n", i, time.Since(begin))
		if i == 1 {
			time.Sleep(time.Millisecond * 2100)
		}
	}
}
```

```
tick(1): 1s
tick(2): 3.1s
tick(3): 4s
```

# Go言語内部の実装

https://github.com/golang/go/blob/74a49188d300076d6fc6747ea7678d327c5645a1/src/time/sleep.go#L178-L189

DurationごとにBuffer1のChannelに書き込みを試み、書き込みできなければスキップするようになっています。
`time.Ticker`は便利ですが、待ち時間が`Duration*2`以上の場合は注意が必要です。
