---
layout: single
title:  "Golang - Request test using net/http/httptrace"
date:   2019-03-14 08:00:00 +0800
categories: WebDevelopment
---
## 前言
在撰寫 HTTP handler 測試程式時，除了測試 response 結果是否如預期之外，我們還需要知道過程中需要耗費多少時間（request latency）。市面上有一些 libraries (e.g `opencensus`) 能提供相關的 HTTP 事件 trace，不過仔細看會發現他們大多也是整合 golang 本身提供的 `http/trace` 來實現追蹤功能，因此我們就直接來了解 `http/trace` 的運作原理和應用方式，再結合 `http/httptest`，讓 handler  test 更完整。

## Request 處理流程

在說明 `http/trace` 之前，由於 request test 中是藉由 http package 來發起的 ，所以先來了解一下 golang 中，透過 `http` package 的 client 處理 http request 的流程。

### New Request
首先我們透過 `http.Get` 發出 request， 過程中會經由 NewRequest  產出 request instance 並交由 client.Do(request) method 處理。

```go
http.Get("/example")

// is equal to: 

req := http.NewRequest("GET", "/example", nil)
resp := http.DefaultClient.Do(req)
```

### Send Request
`client.do` method 主要負責發送 request 和回傳 response，可說是整段流程的核心，而過程中會先處理 HTTP Headers 相關內容，接著將 request 正式透過 `client.send` 送出。
 
```go
// source code from client.go

// 1. Client Do
func (c *Client) do(req *Request) (retres *Response, reterr error) {
  // ...some code to deal with Headers
  if resp, didTimeout, err = c.send(req, deadline); err != nil {
	  // error handle	
	}
	// ...more
}

// 2. Send Request
func (c *Client) send(req *Request, deadline time.Time) 
 (resp *Response, didTimeout func() bool, err error) {
  // ...some code to add cookies
  // send request with transport method
  resp, didTimeout, err = send(req, c.transport(), deadline)
  // ...more
}
```

### RoundTripper (Transport)

接著來到重頭戲啦，上面 code 中可以看到 `c.transport()`，它會回傳 `RoundTripper`，真正執行整個 **http transaction** 就是 RoundTripper 中的 `RoundTrip` method。

```go
// 3. Start RoundTrip
func send(ireq *Request, rt RoundTripper, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {
  // ...more
  resp, err = rt.RoundTrip(req)
  // ..more
}
```

`RoundTripper` 是一個 interface，http package 在預設情況下是使用實作 RoundTripper 的 DefaultTransport 進行基本傳輸， 如果對於 http 流程很清楚的使用者，當然可以自己自製一個 transport 來完成整個 transaction。

緊接著繼續看 DefaultTransport 是如何實作 RoundTrip 的。

```go
// source code from transport.go

// internel RoundTrip methods
func (t *Transport) roundTrip(req *Request) (*Response, error) {
  // get trace from ctx
  ctx := req.Context()
  trace := httptrace.ContextClientTrace(ctx)
}
```

頭幾行馬上看到我們今天要介紹的主角 - `httptrace` 中的 trace instance，它在 `roundTrip` method 被取出來，而從這邊我們就可以很快地了解到為什麼 `httptrace` 可以實現追蹤 http 的流程了，因為在接下來的 transaction 過程中，這個 trace 中的 functions 都會在適時的階段觸發。 

```go
func (pc *persistConn) readResponse(rc requestAndChan, trace *httptrace.ClientTrace) (resp *Response, err error) {
	if trace != nil && trace.GotFirstResponseByte != nil {
		if peek, err := pc.br.Peek(1); err == nil && len(peek) == 1 {
		  // trigger trace hook functions
			trace.GotFirstResponseByte()
		}
	}
}
```

![httptrace]({{ site.url }}/assets/images/httptrace-1.png)

## httptrace packages
httptrace 主要提供 HTTP client requests 的事件中追蹤，package 中包含了 `ClientTrace` struct，我們可以客製化 `ClientTrace` 中的 methods，並把 trace 放入 request context，這樣的話，這個 trace 就會在上面剛剛所提到的 roundTrip 中被觸發。

> Package httptrace provides mechanisms to trace the events within HTTP client requests.

目前 ClientTrace 有幾個 hooks methods ，分別是：
1. `GetConn func(hostPort string)` - called before a connection is created
2. `GotConn func(GotConnInfo)` - called after a successful connection is obtained
3. `PutIdleConn func(err error)` - called when the connection is returned to the idle pool
4. `GotFirstResponseByte func()`
5. `Got100Continue func()`
6. `Got1xxResponse func(code int, header textproto.MIMEHeader) error`
7. `DNSStart func(DNSStartInfo)`
8. `DNSDone func(DNSDoneInfo)`
9. `ConnectStart func(network, addr string)`
10. `ConnectDone func(network, addr string, err error)`
11. `TLSHandshakeStart func()`
12. `TLSHandshakeDone func(tls.ConnectionState, error)`
13. `WroteHeaderField func(key string, value []string)` - called after the Transport has written each request header
14. `WroteHeaders func()`
15. `Wait100Continue func()`
16. `WroteRequest func(WroteRequestInfo)` - called with the result of writing the request and any body

蠻多節點可以追蹤的，就看使用者自己的需求啦～

## 實作
了解上述的流程之後，我們就可以實際實作一次，在 request test 中追蹤 http request 流程了! 假設我們要追蹤 `GotConn` 到 `GotFirstResponseByte` 這段時間。

1. 首先引入必要的 packages

```go
import (
  "time"
  "net/http"
  "net/http/httptest"
  "net/http/httptrace"
)
```

2. 接著產生一個包含我們自己實作 methods 的 trace

```go
type Tracer struct {
	start       time.Time
	end         time.Time
	latency     time.Duration
}

func (l *Tracer) GotConn(connInfo httptrace.GotConnInfo) {
	l.start = time.Now()
}

func (l *Tracer) GotFirstResponseByte() {
	l.end = time.Now()
	l.latency = l.end.Sub(l.start)
}

trace := &httptrace.ClientTrace{
	GotConn: tracer.GotConn,
	GotFirstResponseByte: tracer.GotFirstResponseByte,
}
```

3. 然後把 trace 放入 request context 內

```go
req, _ := http.NewRequest("POST", server.URL, bytes.NewReader(data))
req = req.WithContext(httptrace.WithClientTrace(req.Context(), trace))
```

4. 執行 RoundTrip

```go
// Use DefaultTransport directly
res, err := http.DefaultTransport.RoundTrip(req)
```

剛剛上述流程中有介紹到，預設下是使用 `http.DefaultTransport` 這個 `RoundTripper` 來執行 HTTP transaction 的，所以我們在測試時，就直接呼叫 `http.DefaultTransport.RoundTrip(req)` 即可。如此一來，就可以看到剛剛所設定好的 trace hooks 在過程中被觸發，同時也會獲得最後 response 結果。

## Notice: use `httptest` to start your test server
上面是包含 client 發 request 階段，所以別忘了要先啟動 server，測試才可以正常執行。

```go
server = httptest.NewServer(handler) // Your test handler
```

## References:
- https://golang.org/pkg/net/http/httptrace/

