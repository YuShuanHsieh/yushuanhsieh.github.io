---
layout: single
title:  "Node.js Addons N-API example (v10.11.0)"
date:   2018-09-24 08:00:00 +0800
categories: WebDevelopment
---
## 前言
由於專案需要整合 [Node-RED](https://nodered.org/) ，我們必須開發自己的 Node 來讓用戶可以直接透過視覺化方式來建立簡單邏輯，又我們的 SDK 是 C 版本，因此有這機會可以接觸 Node.js 所提供的 C/C++ Addons 功能。另外，自己寫 server-side 都是使用 Golang，所以這也是難得的機會可以試試 Node.js。

目前網路上使用 Node.js `N-API v10.x` 版本的範例並不多，初期在使用時也因為對於原理不熟悉，因而踩過一些坑，希望這篇文章能講述基本概念，讓其他使用者能避免也踩到相同的坑。

## 相關技術
- C
- Node.js N-API 與v8一些原理知識
- JavaScript

## How C/C++ Addons Work With JavaScript

> **C/C++ Addons** Node.js Addons are dynamically-linked shared objects, written in C++, that can be loaded into Node.js using the require() function, and used just as if they were an ordinary Node.js module.

簡單來說，我們用 C/C++ 撰寫自己的 Addons，就能像 Node.js module 一樣讓 Node.js 可以直接引入並使用。

```javascript
// Your addons module
const example = require('../lib/build/Release/example.node')
// use the function of Your addons module
example.success();
```

不過， Addons 到底是如何運作的呢？
首先要提及 Google 的 opensource **V8 JavaScript engine**，V8 負責編譯 JavaScript Code 成機器碼，同時也提供 runtime 環境，而 Node.js 就是建立在 V8 Engine 的基礎之上，當我們撰寫 JavaScript Code 來建立 object 時，則 V8 runtime 中就會分配一個記憶區塊來放置它。

而我們在編寫 Addons 時，就需要透過 N-API 來取得這些由 JavaScript Code 所產生的 object 的 reference，然後再進行邏輯處理（為什麼需要透過 N-API 呢？回想一下， JavaScript 所定義的 Type 和 C/C++ Type 有很大差異，藉由 API 轉換，才可以取得正確的值）。

## What is N-API?

> N-API is an API for building native Addons. It is independent from the underlying JavaScript runtime (ex V8) and is maintained as part of Node.js itself. The set of APIs that are used by the native code. Instead of using the V8 or Native Abstractions for Node.js APIs, the functions available in the N-API are used.

> APIs exposed by N-API are generally used to create and manipulate JavaScript values. 

N-API 主要功能是輔助 Addons 來對 JavaScript 所產生的值進行使用或操作。如果在網路上搜尋撰寫 Addons，可能會看到很多透過 `include <v8.h>` 來實作的文章，不過現在使用 N-API ，就不需要透過 include 那些 lib ，也能完成相同的實作。

## Create an C/C++ addons 
### 1. Register a Module and methods
前面簡單地概述觀念，現在就直接來實作一個 addons。首先寫個 `.c file` 來註冊 methods 和 module。
```c
#define NAPI_EXPERIMENTAL
// #define NAPI_EXPERIMENTAL (optional) 目前 N-API 有一些實驗性質的 function，透過 define NAPI_EXPERIMENTAL 來使用這些 methods
#include <node_api.h>

napi_value hello_word (napi_env env, napi_callback_info info) {
  // 邏輯實作，先暫時略過
}

napi_value Init (napi_env env, napi_value exports) {
    // 將實作的 method 註冊成 property，如果沒有此定義，則無法在 JavaScript code 中使用。
    napi_property_descriptor desc = { "helloWorld", 0, hello_word, 0, 0, 0, napi_default, 0 };
    napi_status status = napi_define_properties(env, exports, 1, &desc);
    if (status != napi_ok) return NULL;
    return exports;
}
// 註冊 module
NAPI_MODULE(NODE_GYP_MODULE_NAME, Init); 
```

在使用我們寫好的 addons 前，我們必須**註冊此 module 與其 property**，如此一來 v8 在執行時，才能根據註冊內容來取得正確的 mothods。這邊會看到 `NODE_GYP_MODULE_NAME` ，這是指 在 binding.gyp 定義好的名稱，稍後會介紹到， 在實務上，一律用 `NODE_GYP_MODULE_NAME` 這串文字即可。

### 2. Write a `binding.gyp` file

寫好的 addons 必須再 compile 成 Node.js module，才能供 Node.js 引入。
這流程是使用 `node-gyp` 這個內建工具，因此我們必須撰寫對應的 `.gyp` file，來讓 `node-gyp` 正確地 compile 成我們想要的 node。(實際上，node-gyp 是基於 [GYP](https://gyp.gsrc.io/) 的延伸工具，如果想要了解更多 `.gyp file` 的撰寫方式，可以參考 [GYP LanguageSpecification](https://gyp.gsrc.io/docs/LanguageSpecification.md))

> `node-gyp` is a cross-platform command-line tool written in Node.js for compiling native addon modules for Node.js. 

一個基本的 `.gyp` 包含以下內容：

```
{
  "targets": [
    {
      "target_name": "binding",
      "sources": [ "src/binding.cc" ]
    }
  ]
}
```

待續補完。

## Reference
1. https://nodeaddons.com/how-not-to-access-node-js-from-c-worker-threads/
2. https://nodejs.org/api/n-api.html