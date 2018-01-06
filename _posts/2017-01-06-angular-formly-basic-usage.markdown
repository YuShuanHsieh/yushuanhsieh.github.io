---
layout: single
title:  "AngularJS - angular-formly study notes"
date:   2018-01-05 12:00:00 +0800
categories: WebDevelopment
---
## Angular formly 運作原理
將 form 中的每個 field 獨立出來寫成共用 template，並供整個 app 使用。使用者先在 angular-formly 中進行 configuration，包含設定 type 和 wrapper ，接著只要在 component 的 controller 中建立指定 type 的 field objects 之後，angular-formly 就會結合之前設定好的 field template 和 objects， render 出完整的表單。
![study.png]({{ site.url }}/assets/images/angular-formly-usage-1.png)
## Add template
### - Prebuilt template
Some existing templates could be used.
1. Boostrap
2. LumX
3. lonic  
  
不過目前這些 template module 都已經沒在維護，所以還是自己寫一個 template 比較妥當。

-----

### - Custem template
##### [API Documents: formlyConfig](http://docs.angular-formly.com/v8.0.0/docs/formlyconfig)
Use `formlyConfig` in your *run function* (or `formlyConfigProvider` in a config function).

### setType
#### For example:
```javascript
formlyConfig.setType({
  name: 'input', // 定義 field type.
  template: '<input ng-model="model[options.key]" />'
});
```
```javascript
formlyConfig.setType({
  name: 'ipAddress',
  extends: 'input',
  wrapper: 'someWrapper', // 用來包裝 field template 的 wrapper template.
  defaultOptions: {},
  link: function(scope, el, attrs) {},
  controller: function($scope, yourInjectableThing) {},
  apiCheck: {}
});
```
#Notice: Just be aware that angular-formly does make use of common parameters like required or onClick to automatically add attributes to ng-model elements.

#### Some common options of config:
- **name**
field refernece.
- **templateUrl**
(use $templateCache) use html tags to customize your template.
- **wrapper**
額外加入 warpper template 在 field template 外層。
- **apiCheck**
Default setting uses `apiCheck` library.
**extend**
(需注意如果在 extend type 設置和母 type 一樣的 wrapper 會造成衝突，請重新設置一個新的type。)

##### *Example for apiCheck:
```javascript
apiCheck: check => ({
          templateOptions: {
            label: check.string.optional,
            required: check.bool.optional,
            labelSrOnly: check.bool.optional,
          }
        })
```
### setWrapper
- 需使用 `<formly-transclude></formly-transclude>` 來指定 field template 的位置。
- 至少要有 `templateUrl` 或 `template`。

#### Others
The benefit of the `run phase` is you can inject factories and services.

### Template's $scope
在寫 controller 或 expressionProperties function 的時候，可以使用 $scope 來操作 template 中的變數。這邊要注意的是， angular-formly 的 $scope 有調整過，新增一些 formly 特有屬性，所以要使用指定的 property 去取得 model 參數。

##### *常用的property
| Variable Name  |      Description     |
|----------|:-------------:|
| options |  此 object 包含設定 field 的參數，例如: formControl, initModel等。 |
