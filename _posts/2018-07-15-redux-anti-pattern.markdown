---
layout: single
title:  "Redux Anti Pattern"
date:   2018-07-15 08:00:00 +0800
categories: WebDevelopment
---
## 前言
這篇文章是基於 [Redux Anti-Patterns - Part 1. State Management](https://blog.mgechev.com/2017/12/07/redux-anti-patterns-race-conditions-state-management-duplication/) 所進行的探討，包含在專案內是否有犯類似的錯誤，以及後續該如何改善。文章會結合目前專案所使用的 Redux(ngrx)，並檢視使用 Ngrx 是否能避免這些錯誤發生。

## 1. State Duplication
### 問題點
錯誤的建立 state 導致 state 內部 property 或是與其他 state 內容重複。文章內以 session 為例子：
```typescript
interface Store {
  sessions: Session[];
  /** The currentSession would be a part of sessions*/
  currentSession: Session;
}
```
### 解決方式
如上面的例子，將 `currentSession` 改成 index 檢索，所以我們實際取用資料時，還是使用 `sessions` 內的資料，而不是複製一份到 currentSession。
```typescript
interface Store {
  sessions: {[key: number]: Session};
  currentSession: number;
}
```
### 檢討
目前專案內在指定特定的物件時，多使用 index 檢索的方式，並沒有上面所發生的 state 重複問題。不過日前在寫 state 時，差點發生類似的錯誤，因為我必須將 state 內的資料分類，而那時候想法是避免在新增刪除時需要重新進行分類（因為State 內的資料結構是採用 Array），所以額外設置一個已分類好的 Entity。
```typescript
interface Store {
  books: Array<Book>;
  novel: Array<Book>;
}
```

不過馬上發現這樣會造成與主要資料難同步的問題，再加上考量到專案內會使用特定分類的情況不多，新增刪除的情況也不多，所以就不調整原先的資料架構，加一個 selector 的方式來篩選類別。
```typescript
const getNovels = createSelector(
  getAllBooks,
  books => book.category === 'NOVELS'
);
```
假設今天情況改變，專案需要大量的新增刪除行為時，則會調整 state 內的架構，當資料從 DB 取出時，先進行分類，避免後續重複 O(n) 的發生，會是比較好的做法。

## 2. Incorrect Information Expert
### 問題點
這邊所提到的問題比較是針對 local state，如果把 state 放在錯誤的 component，而且在其子 component 對此 local state 進行操作時，有可能造成 parent component 和 child component state 狀態不一致問題。
### 解決方式
使用 reference 的特性，將 state 整個 pass 到 child component，當 child component 操作 state 時，同樣也會影響到 parent component。
### 檢討
目前專案比較少有類似此 local state 的設計，而這問題相對上也比較好解決。

## 3. Implicit State Duplication
### 問題點
一個類似 State 重複的問題，文章給了以下例子：
```typescript
interface State {
  sessions: {[key: number]: Session};
  totalSessions: number;
  currentSession: number;
}
```
其中 totalSessions 為對 sessions Array 所得出的總數，這樣其實也是一種隱性 state 重複。
### 解決方式
建議捨棄可以從其他 state property 中得知資訊的 state，例如上方的例子，我們可以在使用時寫成：
```typescript
Object.keys(sessions).length
```
當然你可能會在在意起效能上的問題，不過使用 pure function 的方式可以達到有效優化。另外，以 ngrx 來說，他使用 observable selector，當 Array 個數發生變化時才會重新運算，所以效能上不會有太大影響。

### 檢討
目前專案全面改採用 ngrx 搭配 selector，因此這問題目前還沒發生。不過在開發期間確實可以持續注意是否有這類隱性 state 重複問題，藉此讓整個 state 保持一致性。

## 4. Overwriting State Updates
### 問題點
這個問題起因於 copy 一份 state 內的資料，而在進行 store dispatch 的時候，誤將 copy 的舊資料 pass 到 reducer，導致 state 原本已更新的資料被覆蓋掉。
```typescript
const setGuide = (id: string) => (dispatch, getStore) => {
  /** Problems */
  const session = getStore().sessions[id];
  return fetch(`/session/${id}/guide`, {
      method: 'put'
    })
    .then(r => r.json())
    .then(guide => updateSession(session.set('guide', new User(guide))));
};
```

### 解決方式
當必須在 async 情況下使用 dispatch 時，應該從上一個 action 中獲得最新的資料，再進行 dispatch，如同以下文章中的例子：
```typescript
this.props.dispatch(updateSession(this.props.session.set('title', 'Advanced Go')))
  .then(s => this.props.dispatch(setGuide(s)));
```
### 檢討
由於 ngrx 中使用 Effects 來處理 action 執行後再一次進行 dispatch action 的行為，所以這部分的錯誤使用可以較妥善地被避免掉。

## 結論
可以看出來，維持 state 內資料的一致性，是使用 redux 很重要的議題。就我的角度來看，比較有可能發生錯誤的地方就是在設計 state property 部分，不過目前 ngrx 有提供 `Entity` 方式讓我們快速組織 state，透過 `Entity` 幫助下，能夠有效避免 state 設計缺陷。總而言之，以上錯誤都會造成資料不同步問題，所以應該盡量避免嘍～