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
上面有提到，Oauth 主要精神是提供第三方授權驗證機制，今天使用者要用 Google 帳號登入我的網站，帳號擁有者不是我而是 Google ，那我必須要有個方法去取得使用者在 Google 的資訊，所以 Google 提供了 Oauth 的授權流程，讓我能先引導使用者前往指定的 Google 授權頁面，授權頁面上會告知使用者我需要的的資料範圍，當使用者確認授權之後， Google 會傳送一組 token 給我，而我就可以用這組 token 去跟 Google 相關資源 Server 要資料。

## 實作流程
Oauth 2.0 中定義了四個取得認證的模式：`Authorization Code Grant(授權碼)` | `Implicit Grant(簡化)` | `Resource Owner Password Credentials Grant(使用者密碼)` | `Client Credentials Grant(客戶端)`
，四個模式的實際差異可以參考 [The OAuth 2.0 Authorization Framework](https://tools.ietf.org/html/rfc6749) 第4節，在這邊不多加贅述差異，本次實作是使用 `Authorization Code Grant` 的方式來進行，而目前常見的實作方式為 `Authorization Code Grant` | `Implicit Grant`，不過 `Implicit Grant` 主要流程是在 browser 上進行，可能會有 token 暴露風險。

### Step 1. 建立授權 Request
#### 故事場景
使用者想要使用 Google 帳號登入我的網站，但是我必須確認這個使用者確實為 Google 用戶，因此我需要先將使用者引導到 Google 授權頁面，同時告知我需要取得使用者基本資訊，讓使用者決定要不要將資訊資料授權給我。

根據上麵所敘述的場景，首先要做的是建立一個**發送給 Google Authorization Server 的請求**來引導用戶轉向到該 Server，而請求中需要包含你的 Web Application 資訊以供 Google Server 辨識。一個簡單的 Google Authorization Server 請求如下:
```html
https://accounts.google.com/o/oauth2/v2/auth?
scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fdrive.metadata.readonly&
state=state_parameter_passthrough_value&
redirect_uri=http%3A%2F%2Foauth2.example.com%2Fcallback&
client_id=client_id
```
從上面一串網址，可以知道我們需要填入的基本參數有 `scope` | `client_id` | `redirect_uri` 這幾項，其中 `client_id` 是需要透過申請憑證來取得，這樣 Google Authorization Server 才會知道來者何人～因此在寫 code 之前，先前往 (Google 憑證頁面)[https://console.developers.google.com/apis/credentials] 申請憑證，有了憑證之後，就使用 Google 所提供的 Golang lib 來建立請求吧！
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
    //告知 Google auth server 授權範圍，在這邊是取得用戶基本資訊和Email，Scopes 為 Google 提供
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

### Step 2. 引導使用者前往授權頁面來確認授權
有了請求網址，我們就可以將使用者導引到該網址去進行授權：
```go
func LoginWithGoogleOAuth(c *gin.Context) {
	redirectURL := CreateGoogleOAuthURL()
	c.Redirect(http.StatusSeeOther, redirectURL) // http.StatusSeeOther 為 303
}
```

在授權頁面上，可以看到你所設定的 scope 和網站資訊，以便使用者進行授權確認。

#### Http Status code
這邊要注意的地方是，Http Status code 必須為 3xx 系列，才能正確執行轉址，此外，建議 Status code 為 303，因為 303 明確定義出：
- 重定向到新地址時，客戶端必須使用`GET`方法請求新地址。
- 客戶端收到的新的URI，不是原始請求資源的替代引用。

不過有個缺點，許多HTTP/1.1版以前的瀏覽器不能正確理解303狀態，所以也能使用 302 替代。

### Step 3. 接收 Google Authorization Server 的回應
當使用者在授權頁面執行動作時，Google Authorization Server 就會向第一個步驟所設定的 `redirect_url` 發送回應，讓你得知使用者要不要進行授權。如果使用者同意的話，你會接收到以下回應
```
https://oauth2.example.com/auth?code=4/P7q7W91a-oMsCeLvIaQm6bTrgtp7&state=state_parameter_passthrough_value
```

最重要的是後面 `code=4/P7q7W91a-oMsCeLvIaQm6bTrgtp7` 這一段，根據 OAuth 流程，Google Authorization Server 並不會直接回應 token ，而是會給一組 code 表示使用者許可授權，而我們必須再拿這組 code 外加憑證申請到的 `client_secret` 去跟 Google 索取 token。此外， OAuth 2.0 建議 code 有效時間最多只有 10 分鐘，所以基本上在拿到 code 之後，就會緊接換 token 的流程。

在這串回應中還有夾帶我們所發出去的 `state` ，此時也要進行驗證，確保這個 state 和發出去的值為一致。

```go
// check state，這邊的 c 指的是 c *gin.Context
s := c.Query("state")
if s != "state_parameter_passthrough_value" {
  c.AbortWithError(http.StatusUnauthorized,, "error message")
  return
}
```

### Step 4. 使用換取 access token，並且拿 token 去跟 Google Server 索取授權資料
有了 `code` 之後，我們就可以換取 access token，並且使用這組 access token 獲得我們所需要的資料。 

```go
// 這邊的 c 指的是 c *gin.Context
code := c.Query("code")
// 拿 code 跟 Google Server 交換 token
token, err := config.Exchange(oauth2.NoContext, code)
```

Google Server 的回應範例：
```json
{
  "access_token":"1/fFAGRNJru1FTz70BzhT3Zg",
  "expires_in":3920,
  "token_type":"Bearer",
  "refresh_token":"1/xEoDL4iW3cxlI7yDbSRFYNG01kVKM2C-259HOF2aQbI"
}
```

#### 如何處理 token
在 Google OAuth 2.0 官方文件有提到，在拿到回應之後，最好把 `access_token` 和 `refresh_token` 都保存在一個長期有效的安全地方，例如有人會存在 shared session。 `refresh_token` 這組 token 是當 `access_token` 過期時，可以拿 `refresh_token` 再去交換一組有效的 `access_token`。

> Important: Your application should store both tokens in a secure, long-lived location that is accessible between different invocations of your application. The refresh token enables your application to obtain a new access token if the one that you have expires. As such, if your application loses the refresh token, the user will need to repeat the OAuth 2.0 consent flow so that your application can obtain a new refresh token.

最後，就拿 `access_token` 去 Google 指定的 Server 位置取資料吧！

```go
// 這個 client 是 http wrapper，讓你可以直接使用 `client.Get` ，它會執行時會自動在網址後方帶上 access_token 字段
client := config.Client(oauth2.NoContext, token)
res, getErr := client.Get("https://www.googleapis.com/oauth2/v3/userinfo")
defer res.Body.Close()
// 讀取返回的資料
rawData, _ := ioutil.ReadAll(res.Body)
```

#### `refresh_token`
上面有提到 `Implicit Grant` 的模式，在此模式之下，是不會有 `refresh_token` 的功能。

## Reference
1. https://developers.google.com/identity/protocols/OAuth2WebServer
2. https://tools.ietf.org/html/rfc6749





