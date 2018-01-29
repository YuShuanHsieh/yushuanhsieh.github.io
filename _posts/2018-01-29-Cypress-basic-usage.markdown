---
layout: single
title:  "Cypress & Cucumber Introduction"
date:   2018-01-29 12:00:00 +0800
categories: WebDevelopment E2Etesting
---
# Cypress E2E Testing Framework
## Features
1. **No more async hell**  
Cypress automatically retries the query until the element is found.
2. **Real time reloads**  
Cypress automatically reloads whenever you make changes to your tests.
3. **Spies, stubs, and clocks**  
Verify and control the behavior of functions, server responses, or timers.
4. **Run with native browsers**  
Keep fast, consistent and reliable tests.

## 運作方式
![Screen Shot 2018-01-27 at 20.34.59.png]({{ site.url }}/assets/images/Cypress-usage-process.png)

## 基本測試撰寫
```javascript
// describe()- Group your test cases.
describe('ThingsPro', () => {

  // context() - Usage is the same as describe().
  context('Login', () => {
  
    // it() - A test case.
    it('error - incorrect email', () => {
    
      // get() - Querying behavior.
      // type() - Action
      // should() - Create an assertion
      cy.get('input[type=email]').type('rooot@moxa.com');
      cy.get('input[type=password]').type('root1234');
      cy.get('button')
        .contains('Sign in')
        .click();
      cy.get('.toast').should('contain', 
        'Please check your email and password');
        
    });
  });
});
```
### Common Querying Methods
[API Document](https://docs.cypress.io/api/introduction/api.html)

Use jQuery-style selectors.
1. `get(selector, options)`  
**Yield:** DOM elements.

2. `find(selector, options)`  
Get the descendent DOM elements of a specific selector.  
**Yield:** DOM elements.

3. `contains(content, options)`  
Get the DOM element containing the text.
**Yield:** DOM elements.

#### Example
HTML file
```html
<header><p class="subtitle">This is a website</p></header>
```
test case
```javascript
cy.get('header')
  .find('.subtitle')
  .contains('This is a website');
```

---

### Common Action Methods
1. ```click(position, options)``` 

2. ```submit(options)``` 

2. ```select(value, options)``` 

All **Yield** current DOM elements.

#### Example
```javascript
cy.get('select')
  .select('apples')
  .select('orange');
```
---

### Assertion Methods
1. ```should(chainers, method, value)```  
**Notice:**  Any valid chainer that comes from `Chai` or `Chai-jQuery` or `Sinon-Chai`.

#### Example
```javascript
cy.get('a').should('have.attr', 'href', '/login')
```

## Cypress Commands Are Asynchronous
[Reference: Commands Are Asynchronous](https://docs.cypress.io/guides/core-concepts/introduction-to-cypress.html#Commands-Are-Asynchronous)
### Use a chain of promises
結合 Promise, Retry 和 timeout 來實作 Chainer object 實現非同步測試。

![Screen Shot 2018-01-28 at 11.12.09.png]({{ site.url }}/assets/images/Cypress-usage-chainer.png)

#### Example
How to set up timeout object.
```javascript
// How to add timeout to cy command.
// For instance: get(selector, options);
// you can use timeout object to *options* params.
cy.get('.subtitle', {timeout: 5000});
```
---

### Run tests serially
將 command 加到特定佇列之中，並在最後才根據佇列順序來執行測試流程。
```javascript
it('error - incorrect email', () => {
  
  // Add commands to a queue.
  cy.get('input[type=email]').type('rooot@moxa.com');
  cy.get('input[type=password]').type('root1234');
  cy.get('button')
    .contains('Sign in')
    .click();
  cy.get('.toast').should('contain', 
    'Please check your email and password');
        
});
  // Run tests in order.
```

## Somethings you MUST need to know
### 1. Cypress is the top of Mocha testing framework
[Reference: Mocha is a feature-rich JavaScript test framework](https://mochajs.org/)
- Could use Mocha related reporters(we use `mochawesome`).
- Cannot change to your preferred frameworks.

### 2. Cypress is an E2E testing framework
- It is very hard to measure code coverage. [Issue: Code Coverage #346](https://github.com/cypress-io/cypress/issues/346)
- 如果需要跑 Code Coverage，請額外使用其他 testing library (Mocha)來執行 Unit test.


### 3. Supported browser Canary/Chrome/Chromium/Electron
- 目前沒有支援 Chrome headless, 因為 Chrome extension 不支援。不過目前已有解法，還在開發中。
- Default use `headless electron`. Will support `Chrome headless` in coming days. [Issue#832: Support chrome headless and change defaults for cypress run](https://github.com/cypress-io/cypress/issues/832)
- If wanna run CI with Chrome, you can use command `cypress run --browser chrome` . (**Notice:** This is in **head** mode not headless.)
- Will support `Firefox` in the future.

# Run Cypress with Cucumber
## Cucumber
[Cucumber Github](https://github.com/cucumber/cucumber)
- A tool that supports **Behaviour-Driven Development (BDD)**
- Cucumber executes executable specifications written in **plain language**.

## Feature file & Step js file
- Feature file `.feature` - 可由PM或其他非工程師的人撰寫。
- `Step.js` - 由工程師搭配 feature file 的內容來實作測試流程。

## Write a Feature file
[Cucumber Feature Introduction](https://github.com/cucumber/cucumber/wiki/Feature-Introduction)

`.feature`, 每個 feature file 為一個功能。Feature file 使用可以閱讀的文字敘述並搭配幾個指定單字：`Given(前提假設)`, `When(當什麼改變)`, `Then(結果)`, `And`,or `But`. 一個 Feature files 通常包含一系列相關的 scenarios。

#### Example:
```
Feature: Slider Bar

  Scenario: Enable a Slider Bar
    Given 這是一個 "disabled" 的 Slider Bar
    When 我指定一個檔案存放位置
    Then Slider Bar 的狀態會變成 "enable"

  Scenario: 拉動 Slider Bar 的位置，會顯示正確的值
    When 使用者將 Slider Bar 拉至中間位置
    Then Slider Bar 的值會變成檔案存放量最大值的一半

  Scenario: 當使用者選擇的數值低於建議數值時，則顯示警告訊息
    When Slider Bar 拉到低於建議數值
    Then 會出現建議數值的警告訊息
```

#### " " - 標示此詞為 Variable
```
  Scenario: 輸入錯誤的 End IP 格式，則更新失敗
    Given 進入 DHCP server "Settings" 頁面
    When "End IP" 設定為 "192.0.0.0.0"
    When 點選 SAVE 儲存設定
    Then 顯示 "error" 資訊

  Scenario: 輸入錯誤的 Netmask 格式，則更新失敗
    Given 進入 DHCP server "Settings" 頁面
    When "Netmask" 設定為 "192.192.192.192.192"
    When 點選 SAVE 儲存設定
    Then 顯示 "error" 資訊

  Scenario: 輸入錯誤的 Primary DNS 格式，則更新失敗
    Given 進入 DHCP server "Settings" 頁面
    When "Primary DNS" 設定為 "192"
    When 點選 SAVE 儲存設定
    Then 顯示 "error" 資訊
```

## 實作 Step.js file
- 如果 .feature 中的步驟沒有被實作在 step file 中，就會報錯。

#### Example
Feature
```
  Scenario: 輸入錯誤的 Netmask 格式，則更新失敗
    Given 進入 DHCP server "Settings" 頁面
    When "Netmask" 設定為 "192.192.192.192.192"
    When 點選 SAVE 儲存設定
    Then 顯示 "error" 資訊

  Scenario: 輸入錯誤的 Primary DNS 格式，則更新失敗
    Given 進入 DHCP server "Settings" 頁面
    When "Primary DNS" 設定為 "192"
    When 點選 SAVE 儲存設定
    Then 顯示 "error" 資訊
```
Step.js file
```javascript
when(`{string} 設定為 {string}`, (item, value) => {
  cy
    .get('label')
    .contains(`${item}`)
    .siblings('input')
    .type(value);
});

then(`顯示 {string} 資訊`, status => {
  cy.get(`.toast-${status}`);
});
```

## Connect Cucumber to Cypress

### Use cucumber preprocessor plugin

- Know more about Cypress plugins [Cypress plugin Document](https://docs.cypress.io/guides/guides/plugins-guide.html)
- The current plugin we use is [cypress-cucumber-preprocessor](https://www.npmjs.com/package/cypress-cucumber-preprocessor)

![Screen Shot 2018-01-28 at 13.22.51.png]({{ site.url }}/assets/images/Cypress-usage-cucumber.png)

#### Add your plugins to plugins/index.js
```javascript
const cucumber = require('cypress-cucumber-preprocessor').default;
module.exports = (on, config) => {
  on('file:preprocessor', cucumber());
};
```
