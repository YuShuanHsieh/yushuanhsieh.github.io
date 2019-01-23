---
layout: single
title:  "工作 - Redux State 被異常更新除錯紀錄"
date:   2019-01-23 08:00:00 +0800
categories: WebDevelopment
---
## 問題：

今天收到 back-end 同事回饋，說是在新版本的 APP UI 中出現不正常行為。由於我們的 menu 必須根據 Embedded System 中的 Applcation 來增減，因此就使用 menu state 來讓其他 component 也可以透過 dispatch 控制 menu 項目。

menu state 中有一個 property 是用來記錄 menu 的縮放，結果現在這個縮放行為會在使用者按 menu item 時出現異常，property value 會回到原始狀態 (initial state)。


## 找出原因：

當同事回報這個問題時，第一個想到的就是在 menu state 被 re-create，才會引發 re-render 行為，進而讓 menu 回到沒有縮放的初始狀態。

### 那是什麼原因導致 state 被 re-create？

前面有提到 menu 必須根據 Application 目前數量來做增減，而我們的 store 中就有存放 Applcations state，並且 subscribe 這個 Applcations state 來變動 menu items。所以說，如果 menu component 被 re-render，十之八九是因為 applications state 被更新。

接下來就很平常的開啟 redux dev tool 來驗證是不是哪段 code 邏輯有問題，導致在不正確的時間點更新到了 Application State。

### 奇怪，沒有任何跟 Application State 相關的 Action 被 Dispatch

很有趣的是，從 tool 中沒有看到任何 action 觸發 application state 的更新，這讓我本來預想的結果完全不同，因為 store 的 state 照理說只會透過 dispatch action 來更新，如果不是，那問題就有點嚴重了，等於 state 會在意外之處被更動。

### 既然沒有方向，那就從 action 的流程開始找起吧！

一時之間實在看不出來我的 code 是哪裡有問題，與其瞎猜或是 try and error，倒不如從 redux dev tool 找端倪。首先在看 action dispatch 時，發現：

(1) 在 dispatch `@ngrx/store-router/update-reducer` 這個 action 後，就會觸發 menu re-create 的行為。

(2) 當 Angular module 為 lazy-load 模式時， re-create 行為才會觸發。

從這兩點看起來，有可能是使用的 libs 有邏輯改變，不過還是不知道問題出在哪裡。於是：

(1) 先從相關 @ngrx/store 的 issues 板上看，看是不是其他使用者有類似的問題。

(2) 由於是從更新 libs 後才出現這樣的行為異常，所以看一下更新版本的 commit 到底是更新什麼內容。

### 找到確切問題點

就在這樣找尋當中，發現其中一個 libs 的 commit 竟然跟 `@ngrx/store-router/update-reducer` 有關，它擷取了這個 action 對 state 進行 deep copy 。

```javascript

if ((action.type === INIT_ACTION || action.type === UPDATE_ACTION) 
  && rehydratedState) {
      nextState = merge({}, nextState, rehydratedState);
    }
    
```

這個發現可真是不得了，因為一但更新 state object，所有相關的 components 都會 re-render。而這些 render 行為其實是沒必要的，因為 state value 根本沒有變化，只是因為 deep copy 行為，導致 reference 被更新。（重點是這些資訊完全不會顯示在 redux dev tool上，所以不會發現各 state 被默默更新了）

接著查看一下前面版本的 code，之前 code 是用 `Object.assign` 是屬於 shadow copy，跟 `lodash.merge` deep copy 是完全不一樣的。這也難怪之前不會出問題，而在  libs 換了新版本後就會出現異常。

## 解決辦法

找到這個問題後，有幾個解決辦法：
1. 首先發個 issue 給作者，看他要不要解決
2. 降到可運行的版本。（當然我相信它改成 merge 一定是因為有 propery assign 的問題，不過之前版本我們並沒有出錯，在找到替代方式之前，先沿用舊版本）
3. Update 版本要更加嚴謹。由於 front-end 的 libs 更新速度相當快，所以一個不小心就很有可能會發生升版後的異常狀況，這部分可能需要想個比較好的流程或是 Test，才能避免沒有在一開始時就發現升版問題。



由於這個問題是發生在第三方 lib 上，所以紀錄以下除錯流程。我覺得自己 code 邏輯還容易找，但是出在這種第三方 lib ，要找問題就會比較困難一些。