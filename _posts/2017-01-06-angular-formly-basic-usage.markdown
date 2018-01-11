---
layout: single
title:  "AngularJS - angular-formly study notes"
date:   2018-01-05 12:00:00 +0800
categories: WebDevelopment
---
## Angular formly 運作原理
將 form 中的每個 field 獨立出來寫成共用 template，並供整個 app 使用。使用者先在 angular-formly 中進行 configuration，包含設定 type 和 wrapper ，接著只要在 component 的 controller 中建立指定 type 的 field objects 之後，angular-formly 就會結合之前設定好的 field template 和 objects， render 出完整的表單。
![study.png]({{ site.url }}/assets/images/angular-formly-usage-1.png)
#### Angular-formly field render source code
Angular-formly 套用 field template 的順序流程：
```javascript
getFieldTemplate(scope.options)
      .then(runManipulators(fieldManipulators.preWrapper))
      .then(transcludeInWrappers(scope.options, scope.formOptions))
      .then(runManipulators(fieldManipulators.postWrapper))
      .then(setElementTemplate)
      .then(watchFormControl)
      .then(callLinkFunctions)
      .catch(error => {
        formlyWarn(
          'there-was-a-problem-setting-the-template-for-this-field',
          'There was a problem setting the template for this field ',
          scope.options,
          error
        )
      })
```
## Set Up Field Templates
### - Prebuilt Template
在使用 angular-formly 之前必須先定義好各 field 的 template。目前有幾個已經建好的簡易 formly template，包含：
1. Boostrap
2. LumX
3. lonic

不過目前這些 template module 都已經沒在維護，所以還是自己寫一個 template 比較妥當。

#### Pre-built template 使用方式
```javascript
import 'angular-formly-templates-bootstrap';
// inject to the module
.module('myApp', ['formly', 'formlyBootstrap']);
```
**[Notice]** 使用之前，請先看 template 的 source code，確認好有哪些 type 有被定義，不然會報錯。（例如: formlyBootstrap 並沒有定義switch type，因此無法使用。）


-----


### - Use Custom Template
##### [API Documents: formlyConfig](http://docs.angular-formly.com/v8.0.0/docs/formlyconfig)
使用 `formlyConfig` 或 `formlyConfigProvider` 來定義自己的 field template。
##### Example1: use `formlyConfig`
在 module.run 中設定的好處是，可以 inject 其他想要的 service。
```javascript
module.run((formlyConfig, MyService) => {
  formlyConfig.setType({
    // Set a type object.
    // MyService could be used in controller.
  });
});
```
##### Example2: use `formlyConfigProvider`
在 cinfig 階段設定，則無法引入 service ，因為該階段 service 還未被建立。
```javascript
module.config(formlyConfigProvider => {
  formlyConfigProvider.setType({
    // Set a type object.
    // You cannot inject your services in a config stage.
  });
});
```
### API Reference
### 1. setType
##### Example 1:
```javascript
formlyConfig.setType({
  // 定義 field type
  name: 'input',
  template: '<input ng-model="model[options.key]" />'
  // 必須使用 model[options.key]，因為是要指定 options.key 的值為 model 對象
});
```
##### Example 2:
```javascript
formlyConfig.setType({
  name: 'ipAddress',
  extends: 'input',
  // 用來包裝 field template 的 wrapper template.
  wrapper: 'someWrapper',
  // 加入自訂 property。
  defaultOptions: {},
  link: function(scope, el, attrs) {},
  // 該controller 的觸發時間為 formly-field(internal API) 之後，field controller 之前。
  controller: function($scope, yourInjectableThing) {},
  // 欄位驗證
  apiCheck: {}
});
```
**[Notice]** Just be aware that angular-formly does make use of *common parameters* like required or onClick to automatically add attributes to ng-model elements.

#### Some properties of config:
- **name**
field refernece.(Required!)
- **templateUrl or template**
(use $templateCache) 套用於該 field 的 template(通常是HTML格式).
- **wrapper**
額外加入 template 在 field template 外層。
- **apiCheck**
Default setting uses `apiCheck` library.
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
- **extend**
(需注意如果在 extend type 設置和母 type 一樣的 wrapper 會造成衝突，請重新設置一個新的type。)

- **defaultOptions**
在該欄位中加入自定義的 options 供使用者調整。
##### *Example for defaultOptions:
```javascript
defaultOptions: {
  ngModelAttrs: {
    // 在 type的 attribute 中加入 rows 和 mdMaxLength 屬性。
    rows: {attribute: 'rows'},
    mdMaxLength: {attribute: 'md-maxlength'}
  }
}
```

### 2. setWrapper
定義 wrapper 的寫法和定義 field 寫法雷同，不過template 中需加入指定 html tag。
- 需使用 `<formly-transclude></formly-transclude>` 來指定 field template 的位置。
- 至少要有 `templateUrl` 或 `template`。
##### *wapper template example：
```html
<div>
  <label>
    {{to.label}}
    {{to.required ? '*' : ''}}
  </label>
  <!-- 需使用<formly-transclude> 來指定 field template 放置的位置 -->
  <formly-transclude></formly-transclude>
</div>
```
---
###[ 其他重點 ] Template's $scope  

在寫 controller 或 expressionProperties function 的時候，可以使用\$scope 來操作 template 中的變數。這邊要注意的是， angular-formly 的 \$scope 有調整過，新增一些 formly 特有屬性，所以要使用指定的 property 去取得 model 參數。


#### 常用的property
| Variable Name  |      Description     |
|----------|:-------------:|
| options |  此 object 包含設定 field 的參數，例如: formControl, initModel等。 |
| to |  = `options.templateOptions`|
| fc|  = `options.formControl` |


##### *template's $scope example 1
```html
<label>{{ to.label }}</label>
```
##### *template's $scope example 2
```javascript
  key: 'priority',
  type: 'radio',
  defaultValue: this.item.priority || 1,
  templateOptions: {
    label: 'Priority',
    options: []
  },
  controller: function($scope) {
    $scope.to.options = [
      { value: 1, label: 'Urgent' },
      { value: 2, label: 'Normal' }
    ];
  }
```
-----


## Field configuration
### Property
- key
key 會與 controller 的 model 綁定(controller.model.keyName)。預設 key 值是 field object array 的 index。
- type
這個是指欄位的型別，例如 input / checkbox / switch / select 等。**注意：type 需在 formlyConfig 被定義，也可直接使用現有的template**。
- templateOption
- defaultValue
initialize model. 如果該 model 在 compile-time 中為 undefined，則會 assign 此初始值。*if you want to default values in the form, simply initialize your model with that state and you're good. However, there are some use cases where it is much easier to use the form config to initialize the model, which is why this property was added.*
- hideExpression (string || function)
- expressionProperties (object)
- model
這個用在當 field 的 model 和 formly 的model 為不同的時候。可配合`expressionProperties`（a **deep watch** is added to the formly-field directive's scope to run the **expressionProperties** when the specified model changes）。
