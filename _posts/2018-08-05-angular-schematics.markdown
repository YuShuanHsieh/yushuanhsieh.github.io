---
layout: single
title:  "Angular Schematics"
date:   2018-08-05 08:00:00 +0800
categories: WebDevelopment
---
## 前言：
最近專案需要提供工具讓外部人員一同參與 Web App 的開發，由於我們的 Web App 已經有基本架構和開發方式，為了讓外部人員能夠更方便的 follow 架構，所以就決定使用 Schematics 來創建專屬樣板，讓協作開發人員可以迅速地建立專案用的 component page，然後他們只需要修改部分程式碼即可。

## Angular Schematics
因為我們使用 Angular 框架，所以就使用 Angular Schematics 來建立樣板。Schematics 是一個改善 workflow 的工具，而透過 Angular Schematics ，我們可以模組化地調整專案現有檔案，例如在專案內新增指定檔案，或是修改既有檔案的內容等，減少手動調整 effort 和錯誤發生。

如同圖片所示，我們使用 Schematics 將要修改的部分（橘色方塊）apply 在 staging  area，當所有內容調整完成，再 apply 到既有的專案中。（為什麼要有 staging  area，主要是因為
Schematics 中可能有不只一個修改步驟，透過先應用在 staging  area ，可以避免中間有錯誤發生，而導致直接影響到既有專案） ![Schematics are atomic](https://static1.squarespace.com/static/5618e3e3e4b0b6c3336cd921/t/5b1019b5352f53e4588abea8/1527781833021/GIF1.gif?format=1500w)
Image source：http://recurship.com/blog/2018/5/31/xfvrq9aauqkayhkd4kzp7gsbfg2bfl

以下為使用 Angular 內建的 Schematics 範例：
```shell
ng generate component menu
# ng - angular cli
# generate - Angular cli 指令，後面需接 Schematics collections
# component - Angular Schematics 內建的 collection
# menu - collection 中規定的參數，這邊是指 component 的名稱
```
這樣就會快速產生出 menu component 的相關檔案。
![angular-schematics-1.png]({{ site.url }}/assets/images/angular-schematics-1.png)

想知道更多 Angular 內建的 schematcis ，可參考 [`@schematics/angular`](https://www.npmjs.com/package/@schematics/angular)

## 套用第三方 Schematics @nrwl/schematics
透過 Angular Cli 使用 Schematics 功能，不一定要使用 Angular 內建的 Schematics ，而是可以套用他人建立的 Schematics Collections。

如果有用 Nx(Nrwl Extensions for Angular)，會發現預設的 Angular Schematics Collections 被替換成 `@nrwl/schematics` 的 Collections，即使還是使用相同的 Command 指令 `ng generate component menu` ，產生的檔案可能會和之前的有所不同。

而這樣的差異主要是因為 `angular.json` 的 `defaultCollection` 參數從 `@schematics/angular` 變成 `@nrwl/schematics`
```json
"cli": {
    "defaultCollection": "@nrwl/schematics"
  }
```

因此，透過修改 `angular.json` file 的 `defaultCollection`，你就可將任何第三方的 Schematics 設為預設。（如果沒修改 defaultCollection 也可以使用第三方 Schematics，只是在輸入指令時就必須宣告 Collections 的位置）

```shell
# 有改變 defaultCollection 參數
ng generate <schmatics名稱> menu

# 沒有改變 defaultCollection 參數
ng generate <schematics位置>:<schmatics名稱> menu
```

## 建立自己的 Schematics 
我們可以建立自己的 schematics collections，然後透過 ng 指令來使用它。要建立 schematics collections，有幾個檔案很重要：
1. collection.json - Collections 定義檔
2. schema.json - Schematic 定義檔
2. 實作 Schematic 的 js files

### 1. collection.json
一個基本的 collection.json 格式如下，一個 Collection 可以定義多組 Schematic：
```json
{
  "$schema": "../node_modules/@angular-devkit/schematics/collection-schema.json",
  "schematics": {
    "southbound-ui": {
      "aliases": ["south"],
      "description": "A blank schematic.",
      "factory": "./southbound-ui/index#southboundUi",
      "schema": "./southbound-ui/schema.json"
    }
  }
}
```
#### $schema
規範 Collection 的結構，因為要遵守 Angular 的規則才會正確被執行，所以這個 property 的值是不會改變。或是你會看到有人這樣寫：
```json
{
  "extends": ["@schematics/angular"],
  "schematics": {
  }
}
```
extends 原有 Angular $schema 的設定。

#### schematics
此 property 是定義 schematic 的內容，例如 schematic 的名稱，以及 schematic 結構等，例如以下寫法：

```json
"southbound-ui": {
  "aliases": ["south"],
  "description": "Create a southbound ui",
  "factory": "./southbound-ui/index#southboundUi",
  "schema": "./southbound-ui/schema.json"
}
```
1. `southbound-ui` *(Required)* 代表 Schematic 的名稱，所以在 Command line 可以下 `ng g southbound-ui`
2. `aliases` 是別名，在 Command line 可以改下 `ng g south`
3. `description` 是描述 Schematics 的用途
4. `factory`  *(Required)* 這是標示實作 Schematics 的function。例如 `./southbound-ui/index#southboundUi` 就是 function 位於 index 檔案， function 名稱是 southboundUi
5. `schema`  *(Required)* 是代表 Schematic 的結構，例如 `ng g south --name=Test` ， `--name` 就是在 schema 檔案定義。

### 3. schema.json
前面提到，`schema.json` 定義了 Schematic 的結構，其例子如下：
```json
{
  "$schema": "http://json-schema.org/schema",
  "id": "southbound-ui",
  "title": "ThingsPro Southbound UI",
  "type": "object",
  "properties": {
    "path": {
      "type": "string",
      "format": "path",
      "description": "The path to create the component."
    },
    "name": {
      "type": "string"
    },
    "url": {
      "type": "string"
    }
  }
}
```
其中最重要的 `properties`，這個屬性定義出可以讓使用者修改的參數，例如上面例子，那使用者就可以透過 `--url=/src/app` 來設定 url 的數值。

#### Tips
這邊有個小技巧，如果不想要使用 `--url` 這個方式來獲得使用者輸入的數值，可以改寫 `schema.json` 為：
```json
 "url": {
  "type": "string",
  "description": "The url.",
  "$default": {
    "$source": "argv",
    "index": 0
  }
},
```
這樣使用 Commnad line `ng generate southbound-ui /src/app` 系統就會自動抓取 Schematics 名稱後方第一個數值作為 url 的值了。

### 3. 實作 Schematic 的 files
以上定義好 Schematic ，那最後就是要來實作它。以下為一個簡單的實作範例：

```javascript
import { Rule, apply, url, template, move, branchAndMerge, chain, mergeWith } from '@angular-devkit/schematics';
import { normalize } from '@angular-devkit/core';
import { dasherize, classify } from '@angular-devkit/core/src/utils/strings';
import { schemaOptions } from './schema';

//這邊的 options  就是 schema.json 中定義的 properties。在執行 Schematics 時，系統會將使用者輸入的設定，轉成一個 object 放入 function 中。
export function southboundUi(options: schemaOptions): Rule {
  
  // Angular 內建的 string 方法，可以將使用者輸入的字串改寫成其他形式
  const utils = { dasherize, classify };
  
  // 如果使用者有設定 `--path` 的數值，則套用該數值，沒有的話就使用 default 'src/app/'
  options.path = options.path ? normalize(options.path) : 'src/app/';

  // 使用設定好的 template files。
  const applyTemplate = apply(url('./files'), [
    
    // 將 options 數值和 Angular 內建的 string 方法套用在 template files 上
    template({
      ...utils,
      ...options
    }),
    
    // 移動到指定位置
    move(options.path)
  ]);

  // 將行為串連起來，這些修改行為會應用在一開始有提到的 staging area，最後套用到既有專案上
  return chain([
    branchAndMerge(chain([
      mergeWith(applyTemplate)
    ])),
  ]);
}
```

#### Template Files
上面行為之中，有提到所謂的 Template files，這個是指可以**把預設好的檔案，透過 Schematic 方式，加入到專案當中**。例如，如果想要新增一個 component.ts 檔案，就可以事先在 `files` folder 中寫好 component.ts，當使用者在使用 Schematics 時，就會自動地在目錄中新增這些預設檔案。

![angular-schematics-2.png]({{ site.url }}/assets/images/angular-schematics-2.png)

那麼，**將 options 數值和 Angular 內建的 string 方法套用在 template files 上**這又是什麼意思呢？

仔細看上方 template file 的檔案名稱 `__name@dasherize__.compoennt.ts`，其中 `name` 就是 options 中的 name 參數， `dasherize` 指的則是 dasherize function，所以：

```javascript
apply(url('./files'), [
  template({
    ...utils,
    ...options
  })
]);
```
的意思就是，將這些 options 參數和 functions 使用在 template file 中。如此一來，透過改寫，原先的 `__name@dasherized__.compoennt.ts` 就會變成 `my-name.component.ts` (使用 --name="myName")

## 在專案中使用自定義的 Schematics (測試)
#### Example:
```
├── @ysSchematics
|   └── package.json
|   └── src
|   |   └── collection.json
|   |   └── southbound-ui
├── libs
├── src
├── package.json
```
1. 在自定義的 Schematics folder 中使用 `npm link`（範例為：@ysSchematics folder），記得要加 package.json 歐！
2. 跳回到專案根目錄，使用 `npm link @ysSchematics`
3. Command line 輸入 `ng generate @ysSchematics:southbound-ui`

如此一來，就可以看到你實作的 Schematics 被正確地套用在專案上了。