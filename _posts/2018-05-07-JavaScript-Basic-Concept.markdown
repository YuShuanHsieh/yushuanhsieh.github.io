---
layout: single
title:  "JavaScript Memory Allocation"
date:   2018-05-07 12:00:00 +0800
categories: WebDevelopment E2Etesting
---
## 前言
這篇文章起源於我為了瞭解瀏覽器的運作原理，以利針對瀏覽器進行效能優化時，所延伸出來的基礎文章。剛開始學習切入點是 Chrome 的 V8 JavaScript Engine，看到有一位作者仔細地說明 V8 Engine 如何 Compile JavaScript 的流程，進而引發我對 JavaScript Memory Allocation 的興趣。此外，剛好最近也在用 TypeScript 寫類 static type object，所以想更清楚知道使用 TypeScript 這個 pre-processor，在瀏覽器中會怎樣提高 JavaScript Compile 的性能。

以下內容會使用 `ECMA-262` 的標準來做探討。

## Call by value & Call by sharing
在 `ECMA-262`的實作規範中，JavaScript 主要採用 `Call by sharing` 和 `Call by value` 的方式來傳遞參數。
- `primitive value` 使用 Call by value。
- `object` 則是使用 Call by sharing。又或者你可以說是 Call by value，只是這個 value 是 memory 位置而已。

>Brendan Eich: There is no copy of any object data.  The only thing that's copied is a *reference* (a safe pointer, if you will) that uniquely addresses the object.

### The difference of Call by value / reference / sharing
如果是 C/C++ 背景的人，對於 Call by value 和 Call by reference 的差異一定很清楚。如果去查找網路文章，會發現有些人會覺得 JavaScript 的 object 是 Call by reference，但其實這還是有些許差異。 

#### Create Copied Variables - Call by value / sharing
Call by value 或是 Call by sharing，在傳遞參數的時候，都會**新建**一個 variable，然後**複製**其值。所以，無論如何他都需要有新的記憶體空間，並且理論上有可能因為 variable 數量龐大而造成 memory leak。

#### Use reference(address) without copy action - Call by reference
Call by reference 則是會直接使用記憶體位置來取得 object，在傳遞參數過程中並不會產生一個新變數並且進行複製行為。

#### Immutable Data?
當然，使用 Call by value 的好處是可以新建一個 variable，在 function 內對新 variable 的值進行變動，也不會影響到原有的 variable。不過 Call by sharing 的值只是複製位置，實際上還是會依循位置而找到對應的 object，這個意思是說所以如果調整 object data 內的值，那麼原有的 variable 還是有可能會受到影響。
