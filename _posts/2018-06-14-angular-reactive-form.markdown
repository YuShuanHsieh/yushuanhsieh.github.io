---
layout: single
title:  "Angular - Reactive Form"
date:   2018-06-14 12:00:00 +0800
categories: WebDevelopment
---

## 前言
在建立表單的部份，Angular 與 AngularJS 最大的不同就是多了 `Reactive Form` 的方式。由於 Reactive Form 比起一般 Template-Driven Form 更具彈性且好掌控，所以就來大致介紹一下與 Reactive Form 相關的流程機制。

## FormControl
`FormControl` 是 reactive form 中最基本且必要的角色。它是一個**完全獨立的 Instance**，裡面包含了 value / validators / status 等重要 member variables。想像一下，如果我們需要一個輸入暱稱的 input 欄位，而 formControl 的任務就是要儲存使用者所輸入的暱稱，以及操作內部的 Validators 對輸入的暱稱字元進行驗證。

### FormControl 運作流程
下圖簡單地介紹 FormControl 的內部機制，首先在建立 FormControl Instance 時，我們可以透過 constructor 初始化 value 和設置想要的 validators function。舉例來說，我們提供給使用者修改暱稱的欄位，需要在欄位上先填好使用者原本的暱稱，以及設置好暱稱驗證方式。

![Angular FormControl]({{ site.url }}/assets/images/angular-formControl.png)

而當有新的 value 進來時，會透過 setValue() 這個 function 來更新 value 參數。在 setValue() 更新完 value 之後，就會進行值驗證，看看新進來的 value 是否符合規範，，並將 status 狀態更新為 invalid 或是 valid。

```typescript
const control = new FormControl('', Validators.required);
control.setValue('newValue');
```

由於有了值的驗證機制，所以開發者除了可以使用 setValue() 來改變 value variable 之外，還可以透過 FormControl 中的 status 來得知值的驗證結果。這在顯示欄位輸入提示上非常有用。

```javascript
// Formcontrol source code
setValue(value: any, options: {
  onlySelf?: boolean,
  emitEvent?: boolean,
  emitModelToViewChange?: boolean,
  emitViewToModelChange?: boolean
  } = {}): void {
    // update value
    (this as{value: any}).value = this._pendingValue = value;
    
    // 中間省略

    // update status function
    this.updateValueAndValidity(options);
}
```

## formControl Directive
上面有提到， FormControl 本身是一個獨立物件，那該如何讓 FormControl 與 DOM 產生連結? `formControl Directive` 所扮演的角色就是 **FormControl Instance 和 DOM 之間的橋樑**，讓使用者所輸入的值可以被更新在 FormControl 的 value 參數上，或是當 FormControl 的 value 改變時，新的值可以被寫上 DOM element 。

我們可以在官網看到以下例子:

```typescript
@Component({
  ///...
})
export class ExampleComponent {
  control = new FormControl('', Validators.required);
}
```
```html
<input type="text" [formControl]="control">
<p>{{control.value}}</p>
```
這就是將新建立好的 ForControl Instance 傳遞給 formControl Directive 來做操作的意思。

藉由 formControl Directive 的協助，ForControl Instance 就可以獲得使用者所輸入的值，然後我們就能把值顯示在畫面上。 (如同上方的 `{{control.value}}` )

```typescript
// formControl Directive source code
@Directive({
  selector: '[formControl]', 
  providers: [formControlBinding], 
  exportAs: 'ngForm'})
export class FormControlDirective extends NgControl implements OnChanges {
  //...
  ngOnChanges(changes: SimpleChanges): void {
    //...
    setUpControl(this.form, this);
    //...
  }
}

// setUpControl
function setUpControl(control: FormControl, dir: NgControl): void {
  // write a value to target DOM element. 
  dir.valueAccessor !.writeValue(control.value);
  setUpViewChangePipeline(control, dir);
  setUpModelChangePipeline(control, dir);
  //...
}
```

## ValueAccessor

下方圖顯示 formControl Directive 如何連結 FormControl Instance 和 DOM element。

![Angular formControl directive]({{ site.url }}/assets/images/angular-directive.png)

可以看到 DOM element 和 formControl Directive 之間有個 `ValueAccessor`，它負責與 DOM element 進行互動。當使用者在欄位中輸入字元，觸發 input event 的時候， formControl Directive 就會透過 `ValueAccessor` 取得使用者輸入的值，然後對 FormControl 的 value 進行更新。

反之，當 FormControl 的 value 發生變動時， formControl Directive 會藉由 `ValueAccessor` 的 writeValue() function，將變動的值寫入 DOM element。

```javascript
// DefaultValueAccessor source code
writeValue(value: any): void {
  const normalizedValue = value == null ? '' : value;
  this._renderer.setProperty(this._elementRef.nativeElement, 'value', normalizedValue);
}
```

## 結論
本篇文章介紹 Angular 機制是如何透過建立 `FormControl`，並且引入 `formcontrol Directive` 來完成使用者輸入互動和欄位驗證。而這樣的概念也會應用在 `formGroup` 和 `formBuilder` 中，往後在寫 Reactive Form 時，就可以更清楚地知道，該如何透過這樣的流程來完成心目中的表單設計。
