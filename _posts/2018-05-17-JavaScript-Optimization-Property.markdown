---
layout: single
title:  "How Does JavaScript Engine (V8) Optimize JavaScript code - Fast Property Access"
date:   2018-05-17 12:00:00 +0800
categories: WebDevelopment
---
# Properties Access in V8
## 前言
傳統的 JavaScript 引擎，在抓取 object 內的 property 位置時，常常是利用 object 內的 `hash table` 來取得對應 property (通常稱為 dictionary )。不過這樣需要花費搜尋時間，相對來說效率就不是太好。也因此在現代的 JavaScript 引擎，會根據 object 內的 property 架構來進行不同方式的效能處理，避免一直使用 hash table，而是改採其他更有效率的結構，來達成快速得到 property 的目的。

## Property definition in V8
- Property: 命名不為 indexed integer 的 property。
 `const student = {id: 'B123456'}`
- Elements: 命名為 indexed integer 的 property。
 `const students = {1: 'student1', 2: 'student2'}`

為了有效率地讀取 property，V8 會使用不同的方式來儲存 property 位置。首先會依據 property 的命名方式，如同上一篇有提到，V8 將 indexed integer property 視為 `elements`，elements 和 一般 property 就會以不同架構來處理。在這邊僅討論一般命名之 property。

## in-object property
要最快速可以 acccess 到 property 的方式，就是直接在 object 內放置 property 的值，也就是 `in-object property`。`in-object property` 在 Object initializer  就會建立，所以如果是在 object 建立後才加入的 property ，那就不會是 in-object property 了。

![in-object.png]({{ site.url }}/assets/images/in-object.png)  

```javascript
const student = {id: '1234', name: 'student'} // in-object property
```

## Fast Property
當有其他 property 加入到 object 時，通常情況下，V8 會將其儲存在 property store (Fixed Array)，這就是所謂的 `Fast Property`。透過 array 的 index ，可以直接讀取 property。不過由於我們是透過 property 的命名來取得 property，因此需要一份名單來紀錄 property 的 key / 值以及其他資訊。這份名單叫做 `Descriptor Array`。

> The descriptor array contains information about named properties like the name itself and the position where the value is stored. Note that we do not keep track of integer indexed properties here, hence there is no entry in the descriptor array.

`Descriptor Array` 存在於 `Hidden Class` 內，至於為什麼要大費周章地維護 `Descriptor Array` 並從其中讀取 property 的位置，主要是因為 inline cache 機制，所以藉由 `Descriptor Array` 可以比傳統 `Dictionary` 方式更有效率。

上面有提到 `Hidden Class`，現代 JavaScript Engine，在創建 JavaSsript object 時，是利用 hidden class 的方式，並用於實作 inline cache 機制。不過這說起來會很長，所以先快速帶過。
>  The HiddenClass stores information about the shape of an object, and among other things, a mapping from property names to indices into the properties(descriptor array).

[Read More Inline Cache](https://github.com/sq/JSIL/wiki/Optimizing-dynamic-JavaScript-with-inline-caches)

![fast-property.png]({{ site.url }}/assets/images/fast-property.png)

## Slow Property
但是，如果當 object 內的 property 新增刪除頻繁時，維護 `Descriptor Array` 就會需要很大的成本，因此這時候 JavaScript engine 會嘗試將這類 object 的 property store 轉成 property dictionary ，就是使用傳統的方式來讀取 property。

![dict-property.png]({{ site.url }}/assets/images/dict-property.png)

要注意的時，由於這時候是直接透過 property dictionary 來取得 property，因此 `Descriptor Array` 失去其作用，不會有 inline cache 的優化效果，整體效率就會相較慢。

## 結論
1. 在非必要情況下，減少重複刪除 property，讓 JavaScript engine 能透過內部機制來優化。
2. 一些必要又不會經刪減的 property，可以在 object initialize 階段就先設置好，以利用 in-object 的特性。

# Reference
1. https://v8project.blogspot.tw/2017/08/fast-properties.html