---
layout: single
title:  "How Does JavaScript Engine (V8) Optimize JavaScript code - Elements"
date:   2018-05-17 12:00:00 +0800
categories: WebDevelopment
---
# 前言
JavaScript 是種弱型態語言，而 JavaScript Engine (V8) 透過對於 JavaScript object 的型態強化來提升 JavaScript 執行效率。 

# Elements Kinds in V8
## What is `Elements`
JavaScript 的 object 可以有任意個 property，而這些 property 的命名又可以是任意字元，例如： 

```javascript
const student = {
  id: 1234, // property name is a string
  1: 2 // property name is an integer
}
```

在 V8 引擎中，將這些整數命名(integer names)的 property 稱為 `Elements`。其中最常見的就是透過 JavaScript Array contructor 所產生 object 的 property。 

``` javascript
const array = ['name1', 'name2', 'name3'];
array[0] = 'name1'; // 0 為 array 的 property name
```

## 根據 Elements 的種類(Elements Kinds)來進行效能提升
當 V8 引擎在對 Array object 進行操作時（例如使用 Array 的 methods: map/foreach），會根據 Array 內的 Elements 種類來做優化。

以下為幾個 V8 引擎內常用的 Elements：
![Elements Kinds in V8](https://4.bp.blogspot.com/-cfidBaKZWSA/WbfpMkT1J0I/AAAAAAAAAao/fBZ72vuw_QcRhAboyILZdoA6ir48ZupjACLcBGAs/s1600/lattice.png)

可以由命名來看出 Elements 的種類由特定類型 (specific types) 到通用類型 (generic types)。例如：`PACKED_SMI_ELEMENTS` 就是指範圍較小的 Integer，而 `PACKED_DOUBLE_ELEMENTS` 則是涵蓋 Interger 及其小數，如此地一層層降階下來。

當 Array object 在創建時，引擎會標記此 Array object 的 elements kinds，舉例如下：
```javascript
const smiArray = [1,2,3]; //PACKED_SMI_ELEMENTS
const doubleArray = [1.0,2.0,3.2]; //PACKED_DOUBLE_ELEMENTS
const array = ['name1','name2','name3']; //PACKED_ELEMENTS
```

針對不同 Elements 種類，引擎會採取不同優化策略，像是`PACKED_SMI_ELEMENTS` 標明 Array 中僅含整數，所以在V8 引擎內部在處理此種類時，自然比 `PACKED_DOUBLE_ELEMENTS` 來的快速。相同地，`PACKED_DOUBLE_ELEMENTS`又比 `PACKED_ELEMENTS` 好些。

## Elements kinds 的變動
變動 Array object 的 property value，則 Elements Kinds 也會隨之改變。例如：
```javascript
let array = [1,2,3]; //PACKED_SMI_ELEMENTS
```
如果在此 array 中插入一個 float 值，則此 array 的 elements kinds 會從原先的 `PACKED_SMI_ELEMENTS` 變成 `PACKED_DOUBLE_ELEMENTS`
```javascript
array.push(0.5); // 0.5 is not an integer, so the elements kinds transit to PACKED_DOUBLE_ELEMENTS
```
要注意的地方是，elements kinds 只能從 **specific 降轉到 generic**，無法再從 generic 轉回 specific。例如前面例子提到：
```javascript
let array = [1,2,3]; //PACKED_SMI_ELEMENTS
array.push(0.5); //PACKED_DOUBLE_ELEMENTS

array[3] = 4; //Still PACKED_DOUBLE_ELEMENTS, Cannot be back to PACKED_SMI_ELEMENTS
```

## HOLEY Elements (slower than PACKED Elements)
而上圖中常見的 Elements 可以看到分成 `PACKED` `HOLEY` 兩個項目。
- PACKED: 指在 Array object 中，所有 element 都有其對應 value。
- HOLEY: 指在 Array object 中，有部分 element 沒有對應的 value。

```javascript
// PACKED Elements
const array = [1,2,3,4]; // 每個 array 內的 elements 都有對應的值

// HOLEY Elements
let array = new Array(3); // 宣告了3個 element 的 Array object，但是其 element 卻沒有對應的值。
```
之所以會這樣區分，是因為當 Array object 中有空缺的值時，那引擎在執行時需要額外的判斷，**進而導致效能較低**。

另一個會產生 HOLEY Elements 的例子：
```javascript
let array = [1,2,3]; // 宣告了一個三個 element 的 Array object
array[5] = 6; // 宣告 Array indexed property 5 的值為 6
```
由於 Array 的 property (index) 具有連續性，所以當宣告 `array[5] = 6;` 時，引擎會自動將 Array 的長度由 3 擴展到 5，也因此造成 array[3] 和 array[4] 會沒有對應值的情況發生。

### HOLEY Elements kinds 無法再轉換成 PACKED Elements kinds
如同上面有提到，Elements kinds 只能降階，所以當引擎判定某個 Array object 為 HOLEY Elements kinds 時，即使後續將每個 Array 內 element 都配置好對應的值，還是無法改變成 PACKED Elements kinds，進而影響引擎處理的效能。

## The Tips of Optimization
1. Array object 內的 element 值盡量統一類型。例如都是 Integer 或是 string，避免 Elements Kinds 的轉換。
2. 避免讓 Array object 產生空缺，造成 HOLEY Elements。例如不要使用 new Array，盡量不要使用 Array index 來直接插入值。
3. 在處理 Array object 時，使用 `for of` `foreach` 等方式。
4. 使用 Array 來產生 array object，而不是自己產生 array-like 的 object，以便讓引擎可以根據 Array prototype 的 methods 來進行效能提升。

```javascript
// Array 是一個 property name 為 indexed number 的 object，所以所謂的 Array-like object 就是：
const array = {
  1: 'name1',
  2: 'name2',
  3: 'name3'
}
```

# Reference
1. https://v8project.blogspot.tw/2017/09/elements-kinds-in-v8.html
2. https://www.youtube.com/watch?v=EhpmNyR2Za0