---
layout: single
title:  "Write a collection"
date:   2017-08-24 19:57:11 +0100
categories: ProgrammingSkills
---
**前言**  
記得在6月份面試的時候，有考到自己寫Stack時，應該注意什麼地方，不過那時候很緊張，只有說到使用array來建構stack會有極限值或空值的問題，後來想想，其實還有element被pop之後的記憶體問題。剛好最近在看書複習時，書中只有簡單將top index - 1來限制可取得的element，導致即使array[index]的值被return後，array[index]實際上還是存在於記憶體中。

**處理方式**  
`Java`  
針對這個問題，去看了一下Java原始碼對於這部分的處理(arraydeque)，果然它會先建立一個Result物件，然後將array[index] = null之後，返回Result，來避免上述所提到的問題。


`C++`  
C++沒有garbage collection支援，所以處理上相對比較複雜一些。從vector的原始碼來看，裡面需要一個struct紀錄start, finish, end的值，再搭配allocate去配置記憶體，所以在執行stack pop時候，必須先.back()取得最後一個element的值，然後使用.pop_back()呼叫destory()來銷毀該element.

```
Alloc::destory(this->_M_impl, this->_M_impl._M_finish);
```

自己實作這些資料結構時候，可能有些小細節沒發現，所以參考原始碼能得到比較多啟示。而且即使是Java，涉及到collection依然會遇到記憶體處理問題(reference)，所以還是要告誡自己在使用這些collections的時候要特別注意。
