---
layout: single
title:  "Go HTTP Routers - Chi/Echo/Gin/gorilla mux 實作細節與比較"
date:   2020-01-21 08:00:00 +0800
categories: 
  - Go
toc: true
toc_icon: "cog"
---
## 前言
之前做 API server，制定 route path 時有遇到一些問題，於是就順手看了幾個 HTTP router / web framework 的 router 部分實作方式，並且記錄下來，提供給大家做個參考。

## 問題
需要加入

1. /stations/instances
2. /stations/{id}

這兩個 path 對應到不同的 handlers。

## Chi
package `github.com/go-chi/chi`

知名 HTTP router library `chi`，其輕量、快速、handler 使用原生 `net/http` struct，都是不錯的優勢。和多數主流的 HTTP router 相似，chi 的 router 是採用 [radix tree](https://en.wikipedia.org/wiki/Radix_tree) 結構，比較特別的是 chi 將 node type 分成四種：

1. ntStatic (`/home`)
2. ntRegexp (`/{id:[0-9]+}`)
3. ntParam (`/{user}`)
4. ntCatchAll (`/api/v1/`)

與其他 router 相較起來，多了 `Regexp` 這個 type，而在插入新的 node 時候，會根據 type 來將 node 插入到適當的位置。

假設我們要插入兩個 path：
1. `/stations/instances`
2. `/stations/{id}`

其樹狀圖如下：

![study]({{ site.url }}/assets/images/router-chi.png)

為什麼要將 nodes 分類插入呢？因為就 path match 的嚴謹程度而言是 Static > Regexp > Param > CatchAll，因此如果先分類好， router 在進行尋找的時候，就會先從 Static 分類 (index = 0) 開始找起，以符合使用者預期的行為。

不過這樣做也會有個小問題，就是沒有用到的 bucket 會空著。(因為 node children 是 **array** 型態呈現)

```go
type node struct {
// node type: static, regexp, param, catchAll
typ nodeTyp

// more members

// child nodes should be stored in-order for iteration,
// in groups of the node type.
children [ntCatchAll + 1]nodes
}
```

另外一個問題是， Chi 在 search 的時候是使用 recursion 方式，這點對於效率與記憶體使用上都會有所影響。其他多數的 router 在 search tree 的時候是使用 iteration，只有在一些必要的部分才使用 recursion。

## Echo

package `github.com/labstack/echo/v4`

echo 是蠻老牌的 web framework，在 2015 年第一次釋出正式版本，目前已來到 V4 版本。同樣地，echo 的 router 結構也是採用 radix tree ，不過在表現上和 chi 略有些差異。

echo 的 node type 只有三種：
1. skind (static)
2. pkind (params)
3. akind (any)

缺少 Regexp type，因此 chi 相對來說，在 route path 設定上更豐富一些 。再來看一下插入 path 時的樹狀圖：

1. `/stations/instances`
2. `/stations/{id}`

![study]({{ site.url }}/assets/images/router-echo.png)

echo 的 node children 是 **slice** 型態，而 slice 內的 node 順序是根據插入先後來決定的，當要尋找 node 的時候，echo 也是會按照 skind > pkind > akind 的順序去找，不過由於 children 插入時並沒有預先分類，因此必須要 loop children 篩選出 type priority 高的 node，這裡是與 chi 明顯的差異處。

```go
// from echo/router.go
type (
    node struct {
		kind          kind
		label         byte
		prefix        string
		parent        *node
		children      children
		ppath         string
		pnames        []string
		methodHandler *methodHandler
	}
	kind          uint8
	children      []*node
)
```

但即使如此，整體上 echo 的 search algorithm 還是寫的很不錯，在 Benchmark 上有優異的表現。

## gin

package `github.com/gin-gonic/gin`

同樣為老牌 web framework，gin 最為人所知的就是 router 的速度快，當然要快的話肯定也是基於 radix tree 所組成。

gin 的 node type 大同小異也是三種：
1. static
2. params
3. catchAll

但與上面兩個 router 設計不同的是，gin 在插入 node 會有一些限制，例如上面的例子：
1. `/stations/instances`
2. `/stations/:id`

![study]({{ site.url }}/assets/images/router-gin.png)

gin 不允許這樣的 path 設計，其原因 node 中有一個 `wildChild` member 是用來進行搜尋判斷，而當插入含有 params path 時，這個 wildChild 會被設定成 true，進而導致同一層的 static path child 會變成 unreachable 狀態。

```go
type node struct {
	path      string
	indices   string
	children  []*node
	handlers  HandlersChain
	priority  uint32
	nType     nodeType
	maxParams uint8
	wildChild bool // wildcard path
	fullPath  string
}
```

此外，今天假設我們有些 path 需求，例如：

1. `student/:id`
2. `student/others/:id`

如果要兼容 gin 本身的限制，就要將 path 設定成：

```go
router.GET("/student/:id", StudentHandler)
router.GET("/student/:id/:others", OtherHandler)
```

這樣就能夠根據不同的 path 來進入到對應的 handler 了。

## gorilla/mux

package `github.com/gorilla/mux`

以上幾個 router 都是採用 radix tree，所以最後來介紹一個很常見，但是採用單純 matcher list 設計的 `gorilla/mux`。

當我們在加入新的 path 時，gorilla mux 會將其新增到 routes slice 上，順序採用加入先後排列。

![study]({{ site.url }}/assets/images/router-mux.png)

(Note: route 中其實有 matchers list，可以結合不同種類的 matcher，例如 headerMatcher, schemeMatcher 等)

在搜尋適合的 route 時候，會從第一個開始匹配，也就是說即使我們要找的 path 是 `/stations/instances` ，但是由於第一個 route 的 matcher 已經符合，所以就不會繼續比對到下一個更精確的 route。

因此也可以知道，gorilla mux 在搜尋上的速度一定是較慢，但可以搭配想要的 matcher，彈性也是蠻高的。

## Benchmark

最後不可免俗地跑個 Benchmark，基於 [julienschmidt/go-http-routing-benchmark](github.com/julienschmidt/go-http-routing-benchmark) 所測試的結果。老實說 Chi 的表現有點出乎我意料，除了 recursion 因素之外，推測還與 context reuse 方式有關，可參考 [chi/tree.go](https://github.com/go-chi/chi/blob/master/tree.go#L374)。

```c
BenchmarkChi_Param               	 2000000	       916 ns/op	     432 B/op	       3 allocs/op
BenchmarkEcho_Param              	20000000	        67.1 ns/op	       0 B/op	       0 allocs/op
BenchmarkGin_Param               	20000000	        73.0 ns/op	       0 B/op	       0 allocs/op
BenchmarkGorillaMux_Param        	  500000	      2239 ns/op	    1280 B/op	      10 allocs/op
BenchmarkChi_Param5              	 1000000	      1215 ns/op	     432 B/op	       3 allocs/op
BenchmarkEcho_Param5             	10000000	       155 ns/op	       0 B/op	       0 allocs/op
BenchmarkGin_Param5              	10000000	       143 ns/op	       0 B/op	       0 allocs/op
BenchmarkGorillaMux_Param5       	  500000	      3130 ns/op	    1344 B/op	      10 allocs/op
BenchmarkChi_Param20             	 1000000	      2009 ns/op	     432 B/op	       3 allocs/op
BenchmarkEcho_Param20            	 3000000	       487 ns/op	       0 B/op	       0 allocs/op
BenchmarkGin_Param20             	 5000000	       382 ns/op	       0 B/op	       0 allocs/op
BenchmarkGorillaMux_Param20      	  200000	      7037 ns/op	    3451 B/op	      12 allocs/op
BenchmarkChi_ParamWrite          	 2000000	       969 ns/op	     432 B/op	       3 allocs/op
BenchmarkEcho_ParamWrite         	10000000	       144 ns/op	       8 B/op	       1 allocs/op
BenchmarkGin_ParamWrite          	10000000	       121 ns/op	       0 B/op	       0 allocs/op
BenchmarkGorillaMux_ParamWrite   	 1000000	      2269 ns/op	    1280 B/op	      10 allocs/op
BenchmarkChi_GithubStatic        	 2000000	       848 ns/op	     432 B/op	       3 allocs/op
BenchmarkEcho_GithubStatic       	20000000	       100 ns/op	       0 B/op	       0 allocs/op
BenchmarkGin_GithubStatic        	20000000	        88.8 ns/op	       0 B/op	       0 allocs/op
BenchmarkGorillaMux_GithubStatic 	  100000	     11857 ns/op	     976 B/op	       9 allocs/op
BenchmarkChi_GithubParam         	 1000000	      1241 ns/op	     432 B/op	       3 allocs/op
BenchmarkEcho_GithubParam        	10000000	       182 ns/op	       0 B/op	       0 allocs/op
BenchmarkGin_GithubParam         	10000000	       186 ns/op	       0 B/op	       0 allocs/op
BenchmarkGorillaMux_GithubParam  	  200000	      7122 ns/op	    1296 B/op	      10 allocs/op
BenchmarkChi_GithubAll           	    5000	    260724 ns/op	   87697 B/op	     609 allocs/op
BenchmarkEcho_GithubAll          	   50000	     36859 ns/op	       0 B/op	       0 allocs/op
BenchmarkGin_GithubAll           	   50000	     39361 ns/op	       0 B/op	       0 allocs/op
BenchmarkGorillaMux_GithubAll    	     300	   3869867 ns/op	  251672 B/op	    1994 allocs/op
BenchmarkChi_GPlusStatic         	 2000000	       875 ns/op	     432 B/op	       3 allocs/op
BenchmarkEcho_GPlusStatic        	20000000	        64.3 ns/op	       0 B/op	       0 allocs/op
BenchmarkGin_GPlusStatic         	20000000	        63.2 ns/op	       0 B/op	       0 allocs/op
BenchmarkGorillaMux_GPlusStatic  	 1000000	      1865 ns/op	     976 B/op	       9 allocs/op
BenchmarkChi_GPlusParam          	 1000000	      1013 ns/op	     432 B/op	       3 allocs/op
BenchmarkEcho_GPlusParam         	20000000	        97.1 ns/op	       0 B/op	       0 allocs/op
BenchmarkGin_GPlusParam          	10000000	       125 ns/op	       0 B/op	       0 allocs/op
BenchmarkGorillaMux_GPlusParam   	  500000	      2998 ns/op	    1280 B/op	      10 allocs/op
BenchmarkChi_GPlus2Params        	 1000000	      1146 ns/op	     432 B/op	       3 allocs/op
BenchmarkEcho_GPlus2Params       	10000000	       148 ns/op	       0 B/op	       0 allocs/op
BenchmarkGin_GPlus2Params        	10000000	       177 ns/op	       0 B/op	       0 allocs/op
BenchmarkGorillaMux_GPlus2Params 	  300000	      5355 ns/op	    1296 B/op	      10 allocs/op
BenchmarkChi_GPlusAll            	  100000	     14494 ns/op	    5616 B/op	      39 allocs/op
BenchmarkEcho_GPlusAll           	 1000000	      1602 ns/op	       0 B/op	       0 allocs/op
BenchmarkGin_GPlusAll            	 1000000	      1702 ns/op	       0 B/op	       0 allocs/op
BenchmarkGorillaMux_GPlusAll     	   30000	     47365 ns/op	   16112 B/op	     128 allocs/op
BenchmarkChi_ParseStatic         	 2000000	       850 ns/op	     432 B/op	       3 allocs/op
BenchmarkEcho_ParseStatic        	20000000	        63.3 ns/op	       0 B/op	       0 allocs/op
BenchmarkGin_ParseStatic         	20000000	        66.7 ns/op	       0 B/op	       0 allocs/op
BenchmarkGorillaMux_ParseStatic  	  500000	      2589 ns/op	     976 B/op	       9 allocs/op
BenchmarkChi_ParseParam          	 2000000	       972 ns/op	     432 B/op	       3 allocs/op
BenchmarkEcho_ParseParam         	20000000	        79.3 ns/op	       0 B/op	       0 allocs/op
BenchmarkGin_ParseParam          	20000000	        81.2 ns/op	       0 B/op	       0 allocs/op
BenchmarkGorillaMux_ParseParam   	  500000	      2590 ns/op	    1280 B/op	      10 allocs/op
BenchmarkChi_Parse2Params        	 1000000	      1068 ns/op	     432 B/op	       3 allocs/op
BenchmarkEcho_Parse2Params       	20000000	       106 ns/op	       0 B/op	       0 allocs/op
BenchmarkGin_Parse2Params        	20000000	       111 ns/op	       0 B/op	       0 allocs/op
BenchmarkGorillaMux_Parse2Params 	  500000	      2695 ns/op	    1296 B/op	      10 allocs/op
BenchmarkChi_ParseAll            	   50000	     27547 ns/op	   11232 B/op	      78 allocs/op
BenchmarkEcho_ParseAll           	  500000	      2681 ns/op	       0 B/op	       0 allocs/op
BenchmarkGin_ParseAll            	  500000	      2977 ns/op	       0 B/op	       0 allocs/op
BenchmarkGorillaMux_ParseAll     	   20000	     97551 ns/op	   30289 B/op	     250 allocs/op
BenchmarkChi_StaticAll           	   10000	    158210 ns/op	   67825 B/op	     471 allocs/op
BenchmarkEcho_StaticAll          	  100000	     23191 ns/op	       0 B/op	       0 allocs/op
BenchmarkGin_StaticAll           	  100000	     22989 ns/op	       0 B/op	       0 allocs/op
BenchmarkGorillaMux_StaticAll    	    1000	   1155625 ns/op	  153240 B/op	    1413 allocs/op
```