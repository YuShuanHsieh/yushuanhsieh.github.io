---
layout: single
title:  "使用 React-Redux 注意事項和運作原理"
date:   2019-01-17 08:00:00 +0800
categories: Web Development
---
## 前言
網路上有很多關於如何使用 `redux` and `react-redux` 的教學文章，所以在這邊就不寫如何去應用，而是會著重在一些可能會忽略的細節以及大概的 實作原理。其實這些細節都寫在官網上，不過一般在教學文章內較少著墨，所以特別摘錄出來，讓大家在使用 `react-redux` 時能注意到可能會發生的問題。

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

### 2. render()太頻繁 - Only Return New Object References If Needed

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

### 3. render()太頻繁 - The Number of Declared Arguments Affects Behavior

`mapStateToProps(state, props)` 提供 state, props 兩個 args，但是如果我們在 mapStateToProps 中**沒有使用** props 的值來運算，卻引入 props 當參數，就會不斷觸發 mapStateToProps function 導致效能問題。

所以為了改善效能，只引入需要 args 即可。

> Notice: **With (state, ownProps)**, it runs any time the store state is different and ALSO whenever the wrapper props have changed.

> This means that you should **not add the ownProps argument unless you actually need to use it**, or your mapStateToProps function will run more often than it needs to.

```javascript

// source code from wrapMapToProps.js

return function initProxySelector(dispatch, { displayName }) {
    const proxy = function mapToPropsProxy(stateOrDispatch, ownProps) {
      return proxy.dependsOnOwnProps
        // 這邊會判斷是否有使用 props 當參數來決定後續處理方式
        ? proxy.mapToProps(stateOrDispatch, ownProps)
        : proxy.mapToProps(stateOrDispatch)
    }
```


### 4. `mapStateToProps` MUST be a fast, pure and synchronous function

這題比較偏效能議題，官方建議 `mapStateToProps` 應該是一個邏輯簡單 / pure / 且 synchronous 的 function。畢竟當 state 變動時，就會呼叫此 function，如果 `mapStateToProps` 的邏輯複雜，那就會大幅降低每次變動後的處理速度。

再來複習 Pure Function 定義:

> 1. Its **return value** is the same for the same arguments (no variation with local static variables, non-local variables, mutable reference arguments or input streams from I/O devices).

> 2. Its **evaluation has no side effects** (no mutation of local static variables, non-local variables, mutable reference arguments or I/O streams).

以下例子就是會改變 variable member 而造成 side effects 的例子：
```javascript
function Test() {
  this.value = 0;
  
  this.notPureFunc = function(a, b) {
    return this.value += a + b;
  }
  
  this.pureFunc = function(a, b) {
    return a + b;
  }
}
```

從上面例子可以知道，如果不是使用 pure function，可能的風險就是會造成 return 值不會是你所預期的（因為他會修改到內部變數），因此這樣寫法也盡量避免的。


## `mapDispatchToProps` Issues

### 1. Using the **object shorthand** form of mapDispatchToProps

網路上可以看到很多例子這樣寫：

```javascript

const mapDispatchToProps = dispatch => {
  return {
    // dispatching actions returned by action creators
    increment: () => dispatch(increment()),
    decrement: () => dispatch(decrement()),
    reset: () => dispatch(reset())
  }
}
```

或是使用 `redux` 本身的 `bindActionCreators`

```javascript

import { bindActionCreators } from 'redux'

function mapDispatchToProps(dispatch) {
  return bindActionCreators({ increment, decrement, reset }, dispatch)
}
```

因為這樣的寫法實在太常見了，頻繁寫 dispatch 看起來變得很多餘，所以 `react-redux` 整合 `mapDispatchToProps` function，我們只要使用 object type 的 mapDispatchToProps，內部機制就會自動幫你套用 `mapDispatchToProps`，讓使用者用起來更方便。

```javascript

import { connect } from "react-redux";

connect(
  mapState,
  { increment, decrement, reset }
)(Counter);

```

不過即使 `react-redux` 幫你貼心設計好了，我們還是要知他是使用了 `bindActionCreators` 來綁定 dispatch 行為。

#### 如果不使用 mapDispatchToProps 這個變數呢？

```javascript

// 官網例子
connect(mapStateToProps /** no second argument */)(MyComponent)

```

那Default 設定就會是 `props.dispatch`，然後再自己手動呼叫 `dispatch(action)`。

當然，官方也有特別提醒，如果使用 mapDispatchToProps，那 dispatch function 就不會被傳入到 component 中。

> Notice: Therefore, if you define your own mapDispatchToProps, the connected component will **no longer** receive dispatch.

因此，如果想要更彈性或是有其他需求，不傳入 mapDispatchToProps，也是一種方式。


## `react-redux` 內部機制

很明顯地， `react-redux` 的 redux store 是使用 react `context` 的方式來實作。

> Internally, React Redux uses **React's "context" feature** to make the Redux store accessible to deeply nested connected components.

```jsx

// 我們通常用這種方式來設定 redux store
<Provider store={store}>
  <...>
</Provider>

// source code from `react-redux`
// `this.state` contains a store property you passed
<Context.Provider value={this.state}>
  {this.props.children}
</Context.Provider>

// The store object is observable pattern to detect value change. 
store.subscribe(() => {
  const newStoreState = store.getState()
  //...some code
  this.setState(providerState => {
    // reference comparision
    if (providerState.storeState === newStoreState) {
      return null
    }
    return { storeState: newStoreState }
  })
})

// --- source code from connectAdvanced.js. use to create connectHOC

// WeappedComponent subscribe the value of context(store)
// and the render will be triggered when selected state changed

renderWrappedComponent(value) { // The value means context value
  // more code..
  const { storeState, store } = value
  // more code..
}

<ContextToUse.Consumer>
  {this.renderWrappedComponent}
</ContextToUse.Consumer>

```

簡單來說，就是利用 React 自身機制的的 `<Context.Provider>` and `<Context.Consumer>` 來實作。當 (1) dispatch action， (2) 經過 reducer function  (3) 產生出一個新的 state 時，(4) 會透過上面 setState 來更新 context value，(5) 接著那些 subscribed context 的 component 就會接收到 state 更新訊息，然後再判斷是否要 re-render component。

簡單流程說明大概是這樣。

## 結尾

學 React 大概第 8 天，覺得雖然他和 Angular 的 template style 不同，不過學起來還是挺上手的，希望接下來能更深入了解他的 render 原理。

## Reference
1. [Connect: Extracting Data with mapStateToProps](https://react-redux.js.org/using-react-redux/connect-mapstate)
