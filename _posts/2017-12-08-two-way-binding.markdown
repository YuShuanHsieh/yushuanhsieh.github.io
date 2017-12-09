---
layout: single
title:  "Angular1 - Two way data binding"
date:   2017-12-08 21:00:00 +0800
categories: WebDevelopment
---
## 前言
最近開始接觸前端技術，預計以Angular和NodeJS為主。之前其實有稍微點一些網頁相關技能，不過大概都才Lv.1，知道大概怎麼使用，卻不太清楚背後的原理和概念。所以這次在學習的時候，會著重在框架理論和實作方式上，希望自己除了能應用這些框架之外，還可以了解這些框架背後的原理，並且進而有能力比較各框架之間的好壞。雖然Angular目前已經出到v5.1.0，但起點會從Angular1開始學起，之後文章預計會去比較新版的Angular和Angular1有怎樣的差異，以及新版的運作方式可以帶來怎樣的好處。

Angular1主打特色之一包含 Two-way data binding (這邊先不在意這種方式帶來怎樣的壞處xD)，這篇文章紀錄 Two-way data binding 的原理和實現方式，以及對網站來說會帶來怎樣的影響。文章將不會示範如何去應用 Two-way data binding，因為我覺得隨便 Google 就可以找到很多例子，就不特別介紹了。

## One-way data binding & Two-way data binding  
### One-way data binding
Angular 官方給的 One-way data binding 說明是指 template system 結合 model data 所產出的結果網頁，也就是說，我們會按照 template engine 要求來編寫一份  template file，然後當 browser 接收到資料時（可能是透過Ajax），就會透過 Template engine 將 data 放入指定的欄位中，並 render 出完整的 HTML file。
![One-way data binding](https://docs.angularjs.org/img/One_Way_Data_Binding.png)  
<p style = "font-size:small;"> (Image reference: https://docs.angularjs.org/guide/databinding) </p>  
使用一般 template system 的問題是，當使用者在網頁上輸入文字或進行其他行為來變動欄位數值時，對應的 model data 不會自動地隨之變動，因為 template system 並沒有內建觸發 browser event 的功能。（當然這個前提是開發者只有單純地使用 template system，並沒有額外編寫其他 javascript code）。

### Two-way data binding  
為了解決上述所討論到的問題，例如：我們希望當使用者輸入資料時，對應的資料也能隨之更新。 Angular1 框架提供了 Two-way data binding 的機制，當網頁中的欄位資訊變動時， data 也會跟著變化，反之亦然。

## 涉及名詞
### $scope
scope object 在官方說法是一個 data-model(和 Model-Controller-View 中的 model 不同，Angular 的 MVC model 叫做 service)，當 Angular controller object 形成時，會透過 dependency injection 的方式來引入一個新的 scope object。scope 聯繫著 controller 和 view (HTML DOM)，並且用來監控特定 data-model 的變化並觸發 event。
![Angular1-Scope]({{ site.url }}/assets/images/Angular1-scope.png)  
### $watch() / watch list
$watch 是 $scope 中的一個 method `$watch(watchExpression, listener, [objectEquality]);`，負責註冊 listener 和對應的 watchExpression，當 watchExpression 變化時，就會執行 listener callback。 watch list 則是指同一個scope中所有的 $watch 集合。  
**注意：**這裡的 $watch 運作方式和 observer pattern 不同。  

典型的使用方式：$watch method 會在開發者使用 ng-model = 'dataName' 來綁定到 $scope data 時被觸發 `function setupModelWatcher(ctrl)`，每綁定一個 data 就會執行一次 $watch method，這也就是為什麼會有多個 $watch 所組成的 watch list。
![Angular1-watch]({{ site.url }}/assets/images/Angular1-watch.png)
### $digest()
$digest 也是 $scope 中的 method 之一，負責尋訪當前 scope 及子 scope 中的所有 watchers，在執行過程中發現某個 watch 的 watchExpression 值有改變，則會執行該 watch 所設定好的 listener callback。
### Browser event
除了關心 Angular1 架構之外，也要看一下瀏覽器執行與 Angular 之間的關聯性。這邊會用到的就是 Browser event。瀏覽器提供不同 Web API 來觸發各種 Event，例如 onclick / onChange，Angular 框架也是依賴這些既有 event API 來運作。

## 運作原理
![Angular1-process]({{ site.url }}/assets/images/Angular1-process.png)  

使用 Angular1 框架時，如果要使用 click 事件，我們會寫 `ng-click = "ctrl.functionName(args)"` ，而 ng-click directive 能透過 Angular1 中的 $compiler 展開成 onclick() 的原生 Web API。所以當使用者點擊某個 Button 時，其實就是觸發 Browser click event ，然後進入 angular 框架中。

接著，指定 Controller 中的 $scope.$digest() 會被觸發，開始執行 digest process loop。每次 loop 都會檢查 watch list 中的 $watch watchExpression 值是否有變更，如果有就執行它的 registered listener。這個 loop 會持續到它檢查所有的 watch 都沒有變動，才會停止，然後進行最後的 update DOM 步驟。這個檢查流程又被稱為 **dirty-checking** ，意味當值有變動過，則 dirty: true，直到 dirty: false 才會退出 loop。

我想這樣說明應該很快就可以看出這樣的流程有什麼問題，確實，為了能達到 Two-way data binding 的效果，程式必須執行至少兩次 digest loop 去檢查和調用 callback，進而影響整個執行效率。當然這個 loop 是有上限的，為了避免無限迴圈， **這個 loop 被限定最多只會執行十次** ，超過則會跳出 Error。不過不管這麼說，這樣的方式看起來是有比較有缺陷一些，在後續的 Angular 版本中也已針對這議題作改善。

另外，如果當前 scope 中有其他子 scope ，程式也會觸發子 scope 的 $digest 來檢查子 scope 的 watch list。簡單來說就是一環扣著一環，這個代價也是蠻高的。

## 觸發 $digest 情況整理
整理一下會觸發 $scope.$digest method 的情形：
1. DOM event
2. Ajax callback
3. Timer with callback
4. Browser location change
5. Manual invocation ($scope.$apply())

## Two-way data binding 相關議題
1. 較低效率且耗費資源 (Dirty-checking)
2. 當遇到多筆資料需要大量更新，數據間依賴性又很嚴重時，就會發生問題。（Dirty-checking次數影響）

## 參考資源
1. [AngularJS - Understanding digest cycle](https://www.youtube.com/watch?v=SYuc1oSjhgY)
2. [$scope object 以及 $watch() $digest()等官方說明 ](https://docs.angularjs.org/api/ng/type/$rootScope.Scope)
