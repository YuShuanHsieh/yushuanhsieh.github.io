---
layout: single
title:  "React Hooks with memoizedState"
date:   2019-02-27 08:00:00 +0800
categories: WebDevelopment
---
## 前言
React Hooks 自從正式 release 後，就出現很多相關教學文章，所以這篇不是講如何實作，而是說他如何在 stateless functional component 中保存當前 state。 此外，最近也開始嘗試把 class component 轉換成使用 hooks 的 components，但過程中還是需要蠻多調整，包含之前有使用一些 lifecycle functions ，藉此機會順便檢視是否真的必要使用這些 functions，是否可以透過其他架構方式來簡化。

## useState

以下是我們使用 useState function 的基本範例：

```javascript

function Example(props) {
const [state, setState] = useState(initialState)
  return (
  /// ReactElement
  )
}

```

依照 functional component 邏輯，我們應該每次都會新建一個 state，導致 state 值無法保存。不過透過 React Hooks，我們卻可以在每次 render 時得到上次更新後的 state 值，這是為什麼呢？

trace code 後，可以發現這個 function 其實是呼叫 internal dispatcher 的 useState

```javascript

export function useState<S>(initialState: (() => S) | S) {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}
```

而 `dispatcher` 是什麼呢？其實可以把它想成每個 component 內部的 mount or update 調度器，這個 dispatcher 會在 component render 階段 `renderWithHooks` 被 assign。

```javascript

export function renderWithHooks(
/// 略過
ReactCurrentDispatcher.current =
  nextCurrentHook === null ? HooksDispatcherOnMount : HooksDispatcherOnUpdate;
}

const HooksDispatcherOnMount: Dispatcher = {
  //略過
  useState: mountState,
};

const HooksDispatcherOnUpdate: Dispatcher = {
  //略過
  useState: updateState,
};

```

從上述 code 可以知道，**HooksDispatcher 又分成 mount 和 update**，當 hook 還未建立，會套用 mount ，反之則會使用 update methods。藉由這種方式，可以根據目前 render 的階段，讓 useState 實際上是執行不同的實作 function。

## memoizedState

接續以上 source code，我們先來看 mountState，在 mountState function 中會建立一個新的 Hook object: 

```javascript

function mountWorkInProgressHook(): Hook {

  const hook: Hook = {
    memoizedState: null,
    baseState: null,
    queue: null,
    baseUpdate: null,
    next: null,
  };

  if (workInProgressHook === null) {
    // This is the first hook in the list
    firstWorkInProgressHook = workInProgressHook = hook;
  } else {
    // Append to the end of the list
    workInProgressHook = workInProgressHook.next = hook;
  }
  return workInProgressHook;
}

function mountState<S>(
  initialState: (() => S) | S,
): [S, Dispatch<BasicStateAction<S>>]{
  // 略
  const hook = mountWorkInProgressHook();
  hook.memoizedState = hook.baseState = initialState;
  
  return [hook.memoizedState, dispatch];
}

const queue = (hook.queue = {
    last: null,
    dispatch: null,
    eagerReducer: reducer,
    eagerState: (initialState: any),
  });
```

可以看到 initialState 會被 assign 給 `hook.memoizedState`。而從 `workInProgressHook.next = hook` 可以知道， hook 是一個 linked list 結構，紀錄了該 hook 的 state 值並有 pointer 指向下一個 hook。

而這個 hook linked list 又會被記錄在 fiber 的 memoizedState。

```javascript

const renderedWork: Fiber = (currentlyRenderingFiber: any);
renderedWork.memoizedState = firstWorkInProgressHook;

```

> Hooks are stored as a linked list on the fiber's memoizedState field.

這樣看下來就清楚了，原來在 stateless functional component 中使用 useState ，之所以能夠紀錄下當前 state 值，是因為這些 state 被組成一串 hook linked list 並保存在 fiber 欄位。

![React-Hooks]({{ site.url }}/assets/images/React-hooks.png) 

當第一次 render 執行 `useState` 時，會執行 `mountState` 並把 user 設定的 initial State 放到 hook 中的 memoizedState。

而後續再次執行 render 或是 update state 時，就會**直接從 fiber memoizedState (hook ) 中取出保存的 memoizedState** (怎麼有點繞口令)。

## Rules of Hooks

知道了上面的流程後，再來看一下官方聲明的 Hooks 使用規則：

### 1. Only Call Hooks at the Top Level. Don’t call Hooks inside loops, conditions, or nested functions.

由於 Hooks 是使用 Linked List 的方式來保存，所以順序對於 Hooks 來說相當重要，如果像是使用 conditions 導致 useState 順序出現變動，那在透過 Linked List 依序取出 memoizedState 就會出現問題。

### 2. Only Call Hooks from React Functions

從 source code 可以知道 hooks 其實是呼叫 dispatcher 的 functions，如果是非 React Component，那麼在 get dispatcher 就會出現 error 啦！


## Class Component 的 State 保存

既然說到使用 hooks 就可以保存 state，那不免要來談一下我們常用的 class component 是如何保存 state 的。

在 `ReactFiberClassComponent.js` 中有提到新建 ClassInstance 的method:

```javascript

function constructClassInstance(
  workInProgress: Fiber,
  ctor: any,
  props: any,
  renderExpirationTime: ExpirationTime,
){
  // 略
  const instance = new ctor(props, context); //ctor 就是 component class
  // 將我們所寫的 this.state assign 給 fiber memoizedState
  const state = (workInProgress.memoizedState =
    instance.state !== null && instance.state !== undefined
      ? instance.state
      : null);
  adoptClassInstance(workInProgress, instance);
  // 略
}

```

透過上面程式碼可以知道， class component state 最後還是會被 assign 給 fiber memoizedState，這樣說起來， hooks 只是使用另一種方式來達到一樣的效果。


## References
1. [React source code](https://github.com/facebook/react)