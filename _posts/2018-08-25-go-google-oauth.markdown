---
layout: single
title:  "Google Sign-in with OAuth 2.0"
date:   2018-08-25 08:00:00 +0800
categories: Go
---
## 前言：
由於目前專案是以 Embedded System 為主，比較少有機會接入第三方 api 的機會，所以這次 side project 就以 Google Sign-in for Web application with Go 的流程當作練習。由於網路上可以找到很多範例，所以在以下文章中會側重在原理 + 為什麼要這樣做，希望除了寫 code 之外，還能建立起基本概念。

## Oauth 2.0
要實作 Google Sign-in 流程之前，不免要來簡單提到 Oauth 2.0。 OAuth 是一個用於授權認證的標準協議(RFC 6749)，與一般網站常用到的個人帳號驗證機制不同的地方是，OAuth 強調**第三方應用程式授權機制**，所以它的內容有明確定義出各角色定義，包含：
- resource owner
- resource server
- client
- authorization server
  
以這次要實作的場景為例，使用者要透過 Google 帳號來登入我的網站，這裡 `resource owner` 是指使用者，`client` 是指我的網站，`resource server` 和 `authorization server` 則都是指 Google 所提供的服務。

### 為什麼用 Oauth 2.0
上面有提到，Oauth 主要精神是提供第三方授權驗證機制，今天使用者要用 Google 帳號登入我的網站，帳號擁有者不是我而是 Google ，所以 Google 就要提供 Oauth 的服務，讓我可以取得使用者在 Google 上的基本資訊，好讓我能進行後續流程（像是在網站紀錄這個 Google 帳號可以使用我的服務等）

## 實作流程
Oauth 2.0 中定義了四個取得認證的模式：`Authorization Code Grant` | `Implicit Grant` | `Resource Owner Password Credentials Grant` | `Client Credentials Grant`
，本次實作是使用 `Authorization Code Grant` 的方式來進行，而目前常見的實作方式為 `Authorization Code Grant` | `Implicit Grant`，其中 `Implicit Grant` 雖然不是本次重點，但是因為概念差不多，所以也會在最後提一下。

### Step 1. 建立授權 Request
首先要先建立一個**發送給 Google Authorization Server 的請求**來引導用戶轉向到該 Server，而請求中需要包含你的 Web Application 資訊以供 Google Server 辨識。一個簡單的 Google Authorization Server 請求如下:
```html
https://accounts.google.com/o/oauth2/v2/auth?
scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fdrive.metadata.readonly&
state=state_parameter_passthrough_value&
redirect_uri=http%3A%2F%2Foauth2.example.com%2Fcallback&
client_id=client_id
```
從上面一串網址，可以知道我們需要填入的基本參數有 `scope` | `state` | `client_id` | `redirect_uri` 這幾項，其中 `client_id` 是需要透過申請憑證來取得，因此在寫 code 之前，先前往 https://console.developers.google.com/apis/credentials 申請憑證。

有了憑證之後，就使用 Google 所提供的 Golang lib 來建立請求吧～
```go
import (
  "golang.org/x/oauth2"
  "golang.org/x/oauth2/google"
)

func CreateGoogleOAuthURL() string {
  // 使用 lib 產生一個特定 config instance
  config := &oauth2.Config{
    //憑證的 client_id
    ClientID: "Your ID",
    //憑證的 client_secret
    ClientSecret: "Your Secret",
    //當 Google auth server 驗證過後，接收從 Google auth server 傳來的資訊
    RedirectURL:  "http://localhost:8080/google-login-callback",
    //告知 Google auth server 授權範圍，在這邊是取得用戶基本資訊和Email
    Scopes: []string{
      "https://www.googleapis.com/auth/userinfo.email",
      "https://www.googleapis.com/auth/userinfo.profile",
    },
    //指的是 Google auth server 的 endpoint，用 lib 預設值即可
    Endpoint: google.Endpoint,
   }

   //產生出 config instance 後，就可以使用 func AuthCodeURL 建立請求網址
   return config.AuthCodeURL("state")
}
```
#### `config.AuthCodeURL` 所需要的參數 `state` 是指什麼呢？
在上面範例中，可以看到:
```go 
requestUrl := config.AuthCodeURL("state")
```
這個 state 是為了防止 `CSRF(跨站請求偽造)` 攻擊而設置的。建議可以透過**隨機產生**的方式來產生出一個 state，當 Google server 驗證完後，會原封不動地把 state 再回傳給網站 server，如此一來我們就可以驗證 state 是否為網站所發出的 state，以確保正確性。


待完成。