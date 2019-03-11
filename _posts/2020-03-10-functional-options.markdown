---
layout: single
title:  "Functional options pattern in GO"
date:   2019-03-10 08:00:00 +0800
categories: WebDevelopment
---
![study-2019-02]({{ site.url }}/assets/images/functional-options.png)

## 前言
之所以使用 `Functional options` 的契機，是因為用到 gRPC 的 New Server API，發現他是用 `functional options` 來讓使用者調整 Server 預設配置，這樣的作法不但兼具了擴充性和可用性，也能避免一些使用者誤用。而除了看 source code 來學習如何實作之外，也找起相關文章，進而發現原來早在 2014 年就有人發表過類似教學文，實在是太孤陋寡聞了～

趁著這次機會，把相關文章的重點整理出來，讓大家在寫類似 API 時，也能做個參考。

## Self-Referential Functions
首先要提到的是由 `Rob Pike` 所整理出 `self-referential functions` 的[文章](https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html)，此 Pattern 方式可用於:

1. 有效地處理繁多且複雜的 setting options
2. 需要保留之前所設定的 option

以下為文章中的實作例子：

```go

// 1. 定義 Option Type. Foo 為一個 struct 內含我們需要調整的變數
type option func(f *Foo) option

// 2. 使用 closure 來定義一個可以調整特定變數的 function
func Verbosity(v int) option {
    return func(f *Foo) option {
      // 取出之前設定值
      previous := f.verbosity
      f.verbosity = v
      // 回傳包含之前設定值的 option
      return Verbosity(previous)
    }
}

// 3. 接下來建立一個 function 來套用這些 options
func (f *Foo) Option(opts ...option) (previous option) {
    for _, opt := range opts {
      previous = opt(f)
    }
    return previous
}

// 4. Usage
prevVerbosity := foo.Option(Verbosity(3))
foo.DoSomeDebugging()
foo.Option(prevVerbosity)

```

從例子中可以看到，此 Pattern 利用 Closure 特性來生成 option type ，透過這樣的方式，可以簡單地套用這些 options 來達到修改指定變數之目的，且也能回復到之前設定的值，當然如果你不想 return 先前的 option 也行。總而言之，這樣的設計可以使用於多數需要進行參數配置的場景。

## Functional options for friendly APIs

而 `Dave Cheney` 則是在這[文章](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis)中用 `functional options` 來說明類似概念，不過文章中有舉例出過往為了解決這類 setting options 所用的各種方法，並點出這些方法的優缺點和強調 `Functional options` 的優勢，好讓使用者能做個比較。

### 方法：

1. 在 funcation 中引入所有可以調整的參數

```go
func NewServer(port int, addr string)
```

這樣的寫法很直覺，但是一旦需要很多項設定時就很難擴充。

2. configuration struct

```go
type Configuration struct {
 port int
 addr string
}

func NewServer(config Configuration) {
 //
}
```

使用 configuration 應該是蠻常見的作法，這樣提高的擴充性，不過卻也增加一些風險，例如套用 default 行為時，需要判斷這些變數是否存在再套用。當然你也可以設定一個 function 然後 return default configuration 來讓使用者套用。

```go
NewServer(Configuration{}) // May cause error

defaultConfig := default()
NewServer(defaultConfig)
```

不過，還有沒有其他的作法呢？

## Functional Options

`Functional Options` 和上面所提及的 `Self-Referential Functions` 作法相似，一樣是藉由多個 functions 來修改特定變數值。而為了套用 Default 行為，我們可以事先定義一個含有預設值的變數，然後再根據使用者所引入的 options 來修改此預設變數。

```go
// 1. create a configuration type
type Configuration struct {
 port int
 addr string
}

// 2. Create a default config
var defaultConfig = {
  port: 8080,
  addr: "localhost"
}

// Create a option type and some closure functions to return option
// Same as `Self-Referential Functions`

// 3. Use options with default config
func NewServer(opts... Option) {
  // use these option functions to modify your default configuration
  for _, opt := range opts {
    opt(&defaultConfig)
  }
}
```

以上的 code 也是目前 gRPC 所使用的配置方式 - [gRPC NewServer function source code](https://github.com/grpc/grpc-go/blob/master/server.go#L355) 

```go
// gRPC Example
s := grpc.NewServer(grpc.StatsHandler(&ocgrpc.ServerHandler{}))
```

## References

1. [Self-referential functions and the design of options 
](https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html)
2. [Functional options for friendly APIs](https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis)