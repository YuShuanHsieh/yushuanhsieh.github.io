---
layout: single
title:  "CSS Typed Object Model"
date:   2018-06-24 08:00:00 +0800
categories: WebDevelopment
---
## 前言
Chrome 66 版本全新支援 `CSS Typed Object Model`，透過這種新型的 CSS Typed OM，可以有效提升使用 JavaScript 操作 CSSOM 屬性的效率。雖然目前僅有 Chrome 支援（ Firefox 據說在努力中， Edge 目前沒下文），不過既然此方式能對網頁呈現效能有所提升，讓開發者和使用者都因此受惠，未來想必會成為主流方式。

## CSSOM & CSS Typed Object Model
### CSSOM
傳統的 CSSOM 是使用 string 格式來進行 style  修改或是讀取等行為，當開發者傳遞一個 string 內容給 CSSOM 時，它會再解析 string 成真正需要的數值。這樣一來一往需要蠻大的成本，再加上開發者傳送 string 時不好偵錯，以及不方便開發者進行一些數值的運算等，讓傳統 CSSOM 似乎不太適合樣式操作。

Example
```javascript
const el = document.querySelector('.container');
el.style.opacity = '0.2'; //pass string type value to CSSOM
```

### CSS Typed Object Model
針對 CSSOM 產生的種種不便，於是大神們提出 `CSS Typed Object Model` 解決方案，使用更直覺的方式來操作 CSS 樣式數值。大致概念就是使用專屬的 JS object 來代表這些屬性，同時並新增 CSS Value Object 來規範可接受的數值。 這樣不但有利於開發者運用，而且更強化可讀性。

想像一下，如果你需要獲得 Element 當前的 opacity 數值並且增加它，根據 CSSOM 的方式，它會回傳一個 string type 的 value 給你，而新型的 CSS Typed Object Model 是會回傳 number type，後者的方式還是比較好吧。

Example
```javascript
const el = document.querySelector('.container');
el.attributeStyleMap.set('opacity', 0.2);
const value = el.attributeStyleMap.get('opacity').value;
// value = 0.2
```

## CSS Typed Object Model
### StylePropertyMap
在上面例子有看到一個新的 Element Property `attributeStyleMap`，這是在 `CSS Typed Object Model` 定義一種代表 CSS declaration block 的物件。(interface 為 `StylePropertyMap`)，可以透過內定義的 methods 來讀取或是修改特定 CSS property。

```java
interface StylePropertyMap : StylePropertyMapReadOnly {
  void set(USVString property, (CSSStyleValue or USVString)... values);
  void append(USVString property, (CSSStyleValue or USVString)... values);
  void delete(USVString property);
  void clear();
};
```

這比起使用 CSSOM 所產生的 [`CSSStyleDeclaration`](https://drafts.csswg.org/cssom-1/#cssstyledeclaration)有很大的不同，使用方式也更為簡潔。

### computedStyleMap()
```java
partial interface Element {
  [SameObject] StylePropertyMapReadOnly computedStyleMap();
};
```
`CSS Typed Object Model` 另一個好處就是提供了運算後的數值結果，例如在官方文件有提到：
```javascript
el.attributeStyleMap.set('z-index', CSS.number(15.4));
el.computedStyleMap().get('z-index').value === 15   // computed style is rounded.
```
由於 `z-index` 屬性僅接受 interger ，所以即使是設定不可被接受的小數點數值，在最後運算時還是會被調整成 Property 正確的範圍數值。

### CSS Numeric Factory Functions
在 CSS 中有好幾種代表單位，例如 `em` `px` `deg`等，以往我們在設定時就會直接寫：
```javascript
el.style.transform=`rotate(0,0,1,${angle}deg)`;
```
這其實是很不直覺的寫法，有可能會因為打錯單位而無法正確顯示。因此透過 `Numeric Factory Functions` 的協助，我們可以透過指定的 functions 來產生出正確的數字和單位。
```java
partial namespace CSS {
  CSSUnitValue number(double value);
  CSSUnitValue percent(double value);
  CSSUnitValue em(double value);
  // ...more
}
```
example
```javascript
const rotate = new CSSRotate(0, 0, 1, CSS.deg(angle));
```

### CSS Typed Object Model Support
目前僅有 Chrome 66 版本支援 CSS Typed OM，所以如果想要切換 CSSOM 和 CSS Typed OM 使用，可以透過以下方式來進行支援性判斷：
```javascript
if (window.CSS && CSS.number) {
  // Supports CSS Typed OM.
}
```

## CSS Typed Object Model with Web Animation API
這種新型的 CSS Style 寫法也可以使用在 Web Animation API 上，同樣地讓開發者更好撰寫所要的樣式控制。

example
```javascript
const animate = el.animate({
  transform: [
    // 'rotate(0deg)'
    new CSSRotate(0, 0, 1, CSS.deg(0)), 
    // 'rotate(360deg)'
    new CSSRotate(0, 0, 1, CSS.deg(360))
   ]
  }, {
    duration: 3000,
    iterations: Infinity
  });
animate.play();
```


## 結論
`CSS Typed Object Model` 主要目的是在解決以往使用 JavaScript 操作 CSSOM 時所需要耗費的解析成本，以及透過更直接的操作 property object 來提升開發便利性。目前雖然支援性不高，不過還是希望以後能都使用這種方式來操作 CSS，畢竟傳統的方式對於開發者來說確實不太方便～

## Reference
1. [Working with the new CSS Typed Object Model](https://developers.google.com/web/updates/2018/03/cssom)
2. [CSS Typed OM Level 1](https://drafts.css-houdini.org/css-typed-om/#dom-element-computedstylemap)