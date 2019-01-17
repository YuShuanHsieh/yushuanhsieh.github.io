---
layout: single
title:  "使用 React-Redux 注意事項和運作原理"
date:   2019-01-17 08:00:00 +0800
categories: Web Development
---
## 前言
網路上有很多關於如何使用 `react-redux` 的教學文章，所以在這邊就不寫如何去應用，而是會著重在一些可能會忽略的細節以及大概的 `react-redux` 實作原理。其實這些細節都寫在官網上，不過一般在教學文章內較少著墨，所以特別摘錄出來，讓大家在寫  `react-redux` 時能注意到可能會發生的問題。

## `mapStateToProps` Issues

首先來談談在建立 connectHOC 常用到的 `mapStateToProps`，由於這個 
function 關係到 component props，所以就容易產生沒有發生 render 或是 render 次數過多的問題。

### 1. render() 沒有被觸發

關於這部分，官網也有提出相關內容：
> By default, React Redux decides whether the contents of the object returned from mapStateToProps are different using === comparison **(a "shallow equality" check)** on each fields of the returned object.

> Note that returning a **mutated object of the same reference** is a common mistake that can result in your component not re-rendering when expected.

最常發生的例子像是 array，如果使用 array.push(value)，那麼 array 還是同一個 reference，那麼就不會觸發 render().

```javascript

// in reducer
case appListAction.ADD_APP:
    state.list.push(action.payload);
    // state.list 為相同 reference
    return {...state, list: state.list};

// 由於 state.list 為相同 reference，不會觸發 render
const mapStateToProps = state => ({
  list: state.list
})

// Fix
case appListAction.ADD_APP:
  // 重新建立一個 array
  return {...state, list: [...state.list, action.payload]};
```

### 2. Only Return New Object References If Needed

這個問題是在說，我們有可能會濫用 return new object ，導致觸發 render()。例如說：

```javascript
return {
  // map func always create a new array.
  list: state.list.map(item => item),
  target: state.target
}
```
這樣即使 state.list 的值不變，但是由於 map 的 return 值是新建立一個 array，導致 component 的 list 會一直被更新，進而導致頻繁觸發 render()。

建議的解決辦法是使用 `memoized selector functions`，而官方推出的 lib 是 `reselect`，以下節錄 `reselect` 中一段重要原始碼：

```javascript
export function defaultMemoize(func, equalityCheck = defaultEqualityCheck) {
  let lastArgs = null
  let lastResult = null
  
  return function () {
    if (!areArgumentsShallowlyEqual(equalityCheck, lastArgs, arguments)) {
      lastResult = func.apply(null, arguments)
    }

    lastArgs = arguments
    return lastResult
  }
}
```

簡單來說，就是會先檢查 function 的 input value 有沒有發生變化，如果沒有發生變化，則會直接返回上一次運算的結果值。

如何使用 `reselect` 一樣略過不做介紹，網上已經有很多很完整的範例可以供參考～

### 3. The Number of Declared Arguments Affects Behavior

`mapStateToProps(state, props)` 提供 state, props 兩個 args，但是如果我們在 mapStateToProps 中**沒有使用** props 的值來運算，卻引入 props 當參數，就會不斷觸發 mapStateToProps function 導致效能問題。

所以為了改善效能，只引入需要 args 即可。

> Notice: **With (state, ownProps)**, it runs any time the store state is different and ALSO whenever the wrapper props have changed.

> This means that you should **not add the ownProps argument unless you actually need to use it**, or your mapStateToProps function will run more often than it needs to.

待續

## Reference
1. [Connect: Extracting Data with mapStateToProps](https://react-redux.js.org/using-react-redux/connect-mapstate)
