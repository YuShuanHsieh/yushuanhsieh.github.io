---
layout: single
title:  "Understanding Router in Angular 6"
date:   2018-05-24 12:00:00 +0800
categories: WebDevelopment
---
## 前言：
這陣子在看 Reactive Extension (Rx) 和 Angular ，而 Angular 中常使用的 Router 內就有些 Observable Type 的 instance，因此就打算從這邊深入了解一下 Angular Router 的流程和 Observable 應用方式。

## Router Navigation
### Overview
![router.png]({{ site.url }}/assets/images/router.png)

首先，先來說一下大致流程，當使用者點擊設有 `routerLink` attribute 超連結時（或是在 component 裡面 trigger router navigation），設置的連結會先經過分析步驟，然後透過 Router 找出對應的 Component，再將此 Component 呈現在設置好 `<router-outlet></router-outlet>`  的地方。

### 建立 Router Service
網上有很多案例會教導如何設置 Component 的 Router，如下：
```javascript
import { RouterModule, Routes } from '@angular/router';
const routes: Routes = [{path: 'content/:id', component: ArticleComponent}];
@NgModule({
  imports: [RouterModule.forRoot(routes)],
  declarations: [ArticleComponent],
})
```
而 `RouterModule.forRoot(routes)` 實際上會建立一個 `Router` 的 Singleton Service，裡面有 instance `Router.RouterState` 來管理目前的 Route 狀態，以及一些 methods 來讓使用者透過 Router 來進行 navigation。
[Router class API](https://angular.io/api/router/Router)
```javascript
class Router {
  // 省略其他 methods or instance
  get routerState: RouterState
  navigateByUrl(url: string | UrlTree, extras: NavigationExtras = { skipLocationChange: false }): Promise<boolean>
  navigate(commands: any[], extras: NavigationExtras = { skipLocationChange: false }): Promise<boolean>
}
```
### RouterState 樹狀結構與 ActivatedRoute 結點
#### RouterState
`RouterState` 是一個由 ActivedRoute 所組成的樹狀結構，以及 `RouterStateSnapshot` 存放當前的 Route 狀態。所以如果想要查看當前狀態，可以直接透過此 Service 進行查詢。
```javascript
constructor(private router: Router){}
this.router.routerState.snapshot.url; /** 如果當前路徑在/Article/1，就會印出 /Article/1 */
```
[RouterStateSnapshot Interface](https://angular.io/api/router/RouterStateSnapshot)
而當使用者在進行 navigation 時，就是透過 `RouterState` 內的樹狀結構來找到對應的 Component。

#### ActivatedRoute
上面提到 `RouterState` 是個樹狀結構，而 `ActivatedRoute` 就是其中每個 node。 `ActivatedRoute` 除了有 `routeConfig` 配置要顯示 Component 和 matcher (負責驗證當前路徑是否匹配)，更重要的是還帶有 `params` 和 `data`，讓頁面在切換到指定 Component 時，也能將資料帶往 Component。

```javascript
interface ActivatedRoute {
  params: Observable<Params>,
  data: Observable<Data>,
  get paramMap: Observable<ParamMap>
}
```
假設配置路徑為
```javascript
const routes = [{path : 'article/:id', component : ArticleComponent, data : {content : 'my article'}}]
```
當點擊以下路徑
```html
<a routerLink="/article/1"></a> 
```
就可以透過以下方式來取得對應資料
```javascript
constructor(private route: ActivatedRoute){}
this.route.params.subscribe(params => params) // {id: 1}
this.route.data.subscribe(data => data) // {content : 'my article'}
```
#### ActivatedRoute Injection
要注意的是，一個 ActivatedRoute 對應一個 Component，所以當從 Component Inject ActivatedRoute 時，他會引入關聯到此 Component 的 ActivatedRoute。如果你是想從 Service Inject ActivatedRoute，那要記得**將 Service 放置在 Component 的 providers 中**，假設是放在 module 的 providers，那就會 Inject 到 RouterState.root 這個 ActivatedRoute，這樣就不會是我們想要的結果。

```javascript
@Component({
  selector: 'article',
  templateUrl: 'article.html',
  providers: [ArticleService] // 在此宣告 Service，讓 Service 是依賴在此 Component.
})
```
### Subscribe ActivatedRoute Service
最後，假設我們需要追蹤 ActivatedRoute.params 的變動，根據最新的 params 值做處理，很理所當然地，我們會使用 subscribe，例如：
```javascript
constructor(private route: ActivatedRoute){}

ngOnInit{
  this.route.params.subscribe(params => {
    // deal with params
  })
}
```
如此一來，當 route.params 值發生改變時，我們就可以透過 subscribe 來追蹤。思考 observable 的做法，如果能進行 subscribe，那必定有個執行 next(params) 的地方。

```javascript
function advanceActivatedRoute(route: ActivatedRoute): void {
  if (route.snapshot) {
    if (!shallowEqual(currentSnapshot.params, nextSnapshot.params)) {
      (<any>route.params).next(nextSnapshot.params);
    }
  }
}
```
當在進行 navigate 時，會執行 `runNavigate()` ，然後依序處理 navigate 的流程，並執行 `activateRoutes()` 來觸發上面那段 next(params) 通知 observers。

如此一來，observer 就可以處理最新 params 了！

## Reference
http://vsavkin.tumblr.com/post/145672529346/angular-router
https://leanpub.com/router
https://vsavkin.com/angular-router-understanding-router-state-7b5b95a12eab