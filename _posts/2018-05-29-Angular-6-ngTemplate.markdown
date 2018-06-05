---
layout: single
title:  "Angular - ng-template & ng-container"
date:   2018-05-29 12:00:00 +0800
categories: WebDevelopment
---
## 前言：
昨天剛好在看 Angular CDK Overlay [(Overlay Document)](https://material.angular.io/cdk/overlay/overview) 這部分，因為 Overlay 是 CDK 中 Portal 的ㄧ種，而 Portal 又和 Angular 中的 Template 機制有關係，這才發現自己對於 Angular 的 `ng-template` 和 `ng-container` 概念不是很清楚，所以特別再看了一下這部分的實作原理和應用方式。

## `<ng-template>`
在說明 `ng-template`之前，先從我們很常用到的 `*ngIf` 來說起，在 Angular 中 `*` 星號代表 `ng-template` 的 sugar syntax，讓開發者簡化語法。所以下面的語法：
```html
<div *ngIf="condition">Hello World!</div>
```
實際上就是下面語法的縮寫：
```html
<ng-template [ngIf]= "condition" >
   <div>Hello World!</div>
</ng-template>
```
**那麼，`<ng-template>`到底是什麼東西呢？**
> The `<ng-template>` is an Angular element for rendering HTML. **It is never displayed directly**. In fact, before rendering the view, Angular replaces the `<ng-template>` and its contents with a comment.

從官方文件中，我們可以知道，Angular Render 機制中並不會直接把這個 selector 和裡面所包含的內容顯示在最後 HTML 上。所以如果開發者沒有對 `<ng-template>` 進行額外處理，基本上就等於無用。

**所以，該如何正確地使用 `<ng-template>` ？

首先，我們可以把 `<ng-template>` 想像成定義一個 template 的外框，你可以在裡面放入想要的 DOM。
```html
<!--定義一個有你好文字和登出按鈕的 Template-->
<ng-template>
  <h1>Hello Yu-Shuan！</h1>
  <button>Logout</button>
</ng-template>
```
只有這樣寫是沒有任何作用的，所以我們可以透過 `TemplateRef` ，來決定如何讓裡面的 DOM 顯示。根據官方文件說明：

> You can access a `TemplateRef` via a directive placed on a <ng-template> element (or directive prefixed with *) and have the TemplateRef for this Embedded View injected into the constructor of the directive using the TemplateRef Token. 

透過上述內容，我們可以知道，使用 `<ng-template>` 是獲得 `TemplateRef` 的方法之一。在這邊的 `TemplateRef` 指的就是 `<ng-template>` 裡面的 DOM。當我們在 `<ng-template>` 放上 directive， 就能在 directive.ts 中 inject 這個 `TemplateRef`，並且控制 DOM 的顯示邏輯。

讓我們用最一開始的例子來解釋：
![router.png]({{ site.url }}/assets/images/ng-template.png)

在 `<ng-template>` 內放入 `ngIf`，而 ngIf 這個 directive 中會引用 `TemplateRef` ，以圖為例，`TemplateRef` 就是 `<div>Hello World!</div>`。

在引入 `TemplateRef` 之後， directive 內的邏輯會判斷 @input  condition 是否為 true，是的話就在 viewContainerRef 中插入 templateRef，這樣呈現就會是我們想要的結果。

### 小結

1. 只寫 `<ng-template>` 而沒有寫上任何 directive 或是額外處理，其內容不會被顯示在最後 HTML 上。
2. 我們在編寫 directive 時，可以透過 `<ng-template>` 的幫助來取得 `TemplateRef` （被 ng-template 所包住的 DOM）。

## `<ng-container>`
說完 `<ng-template>`，再來談談 `<ng-container>`，其實 `<ng-container>` 比 `<ng-template>` 更單純，就是一個放置 DOM 的容器。 `<ng-container>` 這個 Tag 並不會顯示在 Html DOM 上，僅供 Angular 辨認使用。不過與 `<ng-template>` 不同的是，被 `<ng-container>` 所包住的 DOM 會顯示在畫面上，僅有 `<ng-container>` 這個 Tag 不顯示。
> The `<ng-container>` is a syntax element recognized by the Angular parser. It's not a directive, component, class, or interface. 

現在我們來改寫上面剛剛提到的 `ngIf` 例子：
```html
<ng-template [ngIf]= "condition" >
  <ng-container>
   <div>Hello World!</div>
  </ng-container>
</ng-template>
```
這樣的結果其實是一樣的，因為 `<ng-container>` 在最後 render 時並不會顯示，所以最後 render 出的 DOM 會和沒有寫 `<ng-container>` 的情況相同。

### 使用 ng-container 的時機
如果裡面內容會顯示，只有 `<ng-container>` 這個 Tag 不顯示，那為什麼要用 `<ng-container>`？**當你發現沒有適合的 container 可以裝你想要的 DOM**，就可以使用 `ng-container`。回到剛剛的例子，如果改用 `<div *ngTemplateOutlet="greet">` 那 `<div>` 這個 Tag 就會顯示在最後的 HTML 上，假設你剛好有預設 `<div>` 的 CSS style，那有可能就會讓你裡面的 DOM 被影響到。  

### 其他議題

**等等！可是我看很多例子，都會看到 ng-container 和 
NgTemplateOutlet 並用啊！為什麼沒有提到這個？**

大家會在官方網站教學上，常常看到以下例子：
```html
<!--這邊的 #greet 是指把這個 DOM 元素宣告成 variable，然後就可以直接在下方引入此 variable(而此 variable 就是 templateRef)-->
<ng-template #greet><span>Hello</span></ng-template>
<ng-container *ngTemplateOutlet="greet"></ng-container>
```
現在來看看 `ngTemplateOutlet` 這個 Directive 的解釋：
> Inserts an embedded view from a prepared TemplateRef.

讓我們把 `<ng-container *ngTemplateOutlet="greet"></ng-container>` 拆開來看：
```html
<ng-template [ngTemplateOutlet]="greet">
  <ng-container></ng-container>
</ng-template>
```

可以發現，以上面例子， `<ng-container>` 其實也可以用其他來取代，例如改用 div 寫成 `<div *ngTemplateOutlet="greet">` 也沒啥不可以。

## The Difference between `ng-template` 和 `ng-container`
最後來說一下 `<ng-template>` 和 `<ng-container>` 的差異，兩個 Tag 皆不會被 render 在最後的 HTML DOM 上，是供 Angular 機制使用。不過 `<ng-template>` 內的 DOM 如果沒有經過處理，並不會被 render 在畫面上。

```html
<ng-template>
  <h1> Hello World! </h1>
</ng-template>

<!-- Render 之後，以上內容都不會顯示在畫面上 -->
```
反之， `<ng-container>` 只是個容器，所以他裡面的內容在經過 Render 後，會顯示在畫面上。

```html
<ng-container>
  <h1> Hello World! </h1>
</ng-container>

<!-- Render 之後 -->
<h1> Hello World! </h1>
```

## Reference
1. https://angular.io/guide/structural-directives
2. https://www.youtube.com/watch?v=PTwKhxLZ3jI
3. https://ithelp.ithome.com.tw/users/20020617/ironman/1263