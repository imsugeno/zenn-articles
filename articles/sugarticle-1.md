---
title: "Ginを理解するためにnet/httpを触るやつ 1"
emoji: "🐣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go", "シリーズ", "初心者"]
published: true
---

# はじめに
Web系の事業会社に新卒入社して半年ほど経ちました。

3か月前に配属されGinを触り始めたのですが、業務ではビジネスロジックの実装が中心なため基礎的な部分が全然理解できていません。

さすがにそろそろGinがnet/httpのラッパーとして何をやっているのか気になってきたので、それぞれ同じ機能のサーバを書いてみて比較しながら理解してみたいと思います。

# 環境
OS: Ubuntu 20.04.2 LTS (WSL2)

Go 1.16

Gin 1.7.4

# 制作物
`localhost:3000/public/hello`へのGETリクエストで簡単なjsonを返すAPIサーバを実装してみました。

今回の目的からは若干逸れますが、ディレクトリ構成はこちらの[標準レイアウト](https://github.com/golang-standards/project-layout/blob/master/README_ja.md)を参考にしています。

## Ginでの実装
リポジトリ：https://github.com/imsugeno/gin

ディレクトリ構成
```
.
├── README.md
├── cmd
│   └── main.go
├── go.mod
├── go.sum
└── internal
    └── handler
        ├── handler.go
        └── route_hello.go
```

### go:main.go
```
package main

import (
	"github.com/imsugeno/gin/internal/handler"
)

func main() {
	r := handler.NewGinEngin()
	r.Run(":3000")
}
```

### go:handler.go
```
package handler

import (
	"github.com/gin-gonic/gin"
)

func NewGinEngin() *gin.Engine {
	r := gin.Default()

	public := r.Group("/public")
	{
		bindRouteOfHello(public.Group("/hello"))
	}
	return r
}
```

### go:route_hello.go
```
package handler

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func bindRouteOfHello(r *gin.RouterGroup) {
	r.GET("", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"message": "hello world",
		})
	})
}
```

## go:net/httpでの実装
リポジトリ：https://github.com/imsugeno/net-http

ディレクトリ構成
```
.
├── README.md
├── cmd
│   └── main.go
├── go.mod
└── internal
    └── handler
        └── route_public.go
```

### go:main.go
```
package main

import (
	"net/http"

	"github.com/imsugeno/net-http/internal/handler"
)

func main() {
	mux := http.NewServeMux()

	mux.HandleFunc("/public/hello", handler.Hello)
	http.ListenAndServe(":3000", mux)
}
```

### go:route_public.go
```
package handler

import (
	"encoding/json"
	"net/http"
)

func Hello(w http.ResponseWriter, r *http.Request) {
	message := map[string]string{"message": "hello world"}
	jsonMessage, err := json.Marshal(message)
	if err != nil {
		panic(err.Error())
	}
	w.Write(jsonMessage)
}
```

# 次回以降の追及点
深堀りするのは次回以降に回して、今回湧いた疑問点を書き起こしておこうと思います。

* Ginでのルーティンググループはどのように実装されているの？
`gin.RouterGroup`で`/public`以下のルーティングをまとめているのですが、仕組みがよくわかっていないのでnet/httpでは実装できず、フルパス`/public/hello`でハンドラ関数に紐づける実装しかできませんでした。

この辺りはGinがうまいことやってくれてる部分だと思うので、これからの投稿のネタにしたいと思います。

* Ginではどうやってリクエストメソッドに応じたルーティングをしてるの？
GETリクエストでjsonを返すと冒頭で言ったものの、上の`net-http`リポジトリの実装だとHTTPリクエストメソッドに応じたルーティングはされておらず、POSTでもPUTでも`200 OK`でレスポンスが返ります。

こちらも`gin.RouterGroup`がうまくやってくれてるようなので、ルーティングの話題として理解していこうと思います。

* `gin.Context`の仕事
net/httpでハンドラやハンドラ関数に渡す`http.ResponseWriter`や`*http.Request`を構造体として持っているのはわかるのですが、それ以外にフィールドで持っている`queryCache`やらはなんなんだろうか...

* そもそも`gin.Engine`って何？
何？
