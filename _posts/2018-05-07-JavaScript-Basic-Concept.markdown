---
layout: single
title:  "JavaScript - Pass Params Into Functions"
date:   2018-05-07 12:00:00 +0800
categories: WebDevelopment
---
## 前言
這篇文章起源於我為了瞭解瀏覽器的運作原理，以利針對瀏覽器進行效能優化時，所延伸出來的基礎文章。剛開始學習切入點是 Chrome 的 V8 JavaScript Engine，看到有一位作者仔細地說明 V8 Engine 如何 Compile JavaScript 的流程，進而引發我對 JavaScript Memory Allocation 的興趣。此外，剛好最近也在用 TypeScript 寫類 static type object，所以想更清楚知道使用 TypeScript 這個 pre-processor，在瀏覽器中會怎樣提高 JavaScript Compile 的性能。

以下內容會使用 `ECMA-262` 的標準來做探討。

## Call by value & Call by sharing
在 `ECMA-262`的實作規範中，JavaScript 主要採用 `Call by sharing` 和 `Call by value` 的方式來傳遞參數。
- `Primitive Value` 使用 Call by value。
- `Object` 則是使用 Call by sharing。又或者你可以說是 Call by value，只是這個 value 是 memory address 而已。

在這部分，JavaScript 的開發者 *Brendan Eich* 也有提及：
>Brendan Eich: There is no copy of any object data.  The only thing that's copied is a *reference* (a safe pointer, if you will) that uniquely addresses the object.

而 JavaScript 的 `Primitive Value` 包含六種 Type：
- string
- number
- boolean
- null
- undefined
- symbol (ECMAScript 2015)

因此，當你想要引入參數時，以上 Type 都可以直接在 function 內修改，而不會影響到原有的 variable。
```javascript
let student = 'Cherie';
// variable student is a string(Primitive Value), so it is passed by value
function changeName(name) {
  name = 'Cherry';
}
changeName(student);
console.log(student); //Output is `Cherie`
```

```javascript
let student = {name: 'Cherie'};
// By contrast, variable student is an object, 
// so it is passed by value with memory address(sharing)
function changeName(student) {
  student.name = 'Cherry';
}
changeName(student);
console.log(student.name); //Output is `Cherry`
```

## The Difference of Call by Value / Reference / Sharing
如果是 C/C++ 背景的人，對於 Call by value 和 Call by reference 的差異一定很清楚。查找網路文章，會發現有些人會覺得 JavaScript 的 object 是 call by reference，但其實這還是有些許差異。 

### Create Copied Variables - Call by Value / Call by Sharing
Call by value 或是 Call by sharing，在傳遞參數的時候，都會**新建**一個 variable，然後**複製**其值。所以，無論如何他都需要有新的記憶體空間。理論上，在處理不當導致系統來不及回收下，有可能因為 variable 數量龐大而造成 memory leak。

### Use reference(address) without copy action - Call by Reference
Call by reference 則是會直接使用 variable 的 reference (memory address) 來取得 object，在傳遞參數過程中並**不會產生**一個新變數並且進行複製行為。

## Immutable Data?
程式開發過程中，很常需要傳遞 variable 但不影響原有 variable 的情況發生。在 JavaScript 傳遞 object 是透過 Call by sharing 的方式之下，修改 variable 內的 property 都會影響到原有的 variable。這也意味著在 JavaScript 中，如果想要實現 Immutable Data 而 variable 又是 object type，理論上需要複製 object 結構內的所有 property 才有可能達成。

當然現在有很多 libs 可以在提升效能的前提之下完成這類的 deep copy (意指在記憶體空間中新建一個和原有 object 一樣 size 的 object，而不會只有複製其 memory address)，不過上述所提供的概念，可以讓新手更清楚地掌握 JavaScript 的記憶體運用。

## 結論
JavaScript 和 Java ㄧ樣，都是採用 `Call by Sharing` 的方式在傳遞 variable。也因此在使用 function 處理 variable 的時候，要注意避免在 function 內直接修改 variable。如果希望修改 variable 的值，又希望只在 function 內有效，那就請 deep copy 此 variable，然後再進行修改。
