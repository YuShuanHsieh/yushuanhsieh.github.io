---
layout: single
title:  "The Main Flow of Rendering Engines"
date:   2018-06-23 12:00:00 +0800
categories: WebDevelopment
---
## 前言：
最近在寫 CSS Animation 部分，於是比較深入的看瀏覽器渲染流程，希望在進行 Animation 時可以更加地流暢。

## The Main Flow of Rendering Engines
在寫 HTML 的時候，我們很理所當然地寫出這樣的頁面：
```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <link rel="stylesheet" href="styles.css">
  <title>My Website</title>
</head>
<body>
  My Website
  <script src="./main.js"></script>
</body>
</html>
```
這時候就會有個疑問，那瀏覽器是如何讀懂這些檔案？又如何將頁面呈現在我們眼前呢？首先，瀏覽器要渲染出網頁頁面，需要幾個主要步驟：

![study-Render-process.png]({{ site.url }}/assets/images/study-Render-process.png)

### 1. Files
從 server 獲得 HTML / CSS / JavaScript 等檔案。

### 2. Parsing
HTML parser 會將 HTML 檔案解析成樹狀結構的 `DOM`。相同地， CSS parser 也會將 CSS 解析成樹狀結構的 `CSSOM`。兩者都會被轉化為樹狀資料結構，以利後續的合併匹配。

### 3. Render Tree
接下來，DOM 和 CSSOM 會合併成為一個 `Render Tree`，以供後續繪製時使用。在產生 Render Tree 時，會根據 DOM 的元素，來匹配適合的 CSS 樣式，並且產生新的 Render Tree。不過，假設 CSS 為 `display: none`，則不會被放置到 Render Tree 中。如下方圖所示的 `p span`。
(image source from [Google Developer](https://developers.google.com/web/fundamentals/performance/critical-rendering-path/constructing-the-object-model))
![](https://developers.google.com/web/fundamentals/performance/critical-rendering-path/images/cssom-tree.png?hl=zh-tw)

### 4. Layout
有了 Render Tree，此時還需要將這些 node 配置在適合的位置，才能正確地顯示出我們想要的頁面，而這段版面配置的過程就是 `Layout` 階段。為了避免不斷更新版面配置，Layout 流程設計為 `Dirty Check`，當配置發生變化時，只會去調整標記為 `dirty: true` 的 node。

### 5. Paint
配置好樣式和 Layout ，就可以開始繪製物件。不過要注意的是，這階段所產生的drawing commnad 只是會被紀錄起來，而不會真正地被顯示在螢幕上。（例如 Chrome 使用 [SkPicture](https://skia.org/) 來記錄）

### 6. Composite
這階段會將上敘有提到的 drawing command 進行處理，並且透過 GPU Process 將結果輸出到介面上。而最主要的目的是為了**提升瀏覽效能**，它是一個**獨立的 thread**，並會 cache 最後的頁面結構，然後在特定情況下直接使用 cache 結構，進而讓畫面能更快速地呈現在使用者面前。(例如我們 scroll 捲動畫面)

## Threads

從上面的流程可以知道，瀏覽器從 server 取得檔案到將頁面渲染完成，需要好幾個步驟，而一般使用者又會預期網頁在四秒內就要能呈現完成，到底該如何處理這些流程，會是重要的問題。

### Main Thread
首先，從 HTML Download 到組成 Render Tree 以及 Paint 這整個主要流程，都是由 `Main Single Thread` 來處理。而這也意味著只要有其中一個流程需要外部資源或是執行比較長的時間，就會讓整個畫面停滯住(Render-Blocking)。舉例來說，HTML 中有一個從 server 端引入的 CSS 檔案，由於 CSSOM 在 Render Tree 中是必要的，所以 Main Thread 會被暫停住，等待 CSS 檔案下載完畢後進入 CSS Parsing ，而這處理期間，使用者可能會看到頁面呈現一片空白（或是不完整畫面），必須等到整個 Render Tree 出來並且被繪製，才會看到完整頁面。

根據上面所述，這對使用者來說可能不會是好的體驗，因此現在瀏覽器會盡可能地提早將頁面繪製出來，即使**這個頁面並不是完整頁面**。理論上來說，繪製的流程必須等到 Render Tree 建構完成之後才會發生，不過瀏覽器實際在執行時會讓第一次繪製在 Render Tree 建構完整之前發生。

![rendering-engine-2.png]({{ site.url }}/assets/images/rendering-engine-2.png)

即使如此，網站還是要等到整個渲染出來才有其意義在，所以雖然瀏覽器會提前將部分畫面顯示出來，但只是為了提高用戶體驗，不影響本節所要討論的重點：**當 HTML / CSS Parsing 以及 JavaScript 執行或是在處理版面配置繪製時，都需要佔用 Main Thread，進而延長頁面顯示所需的時間。**

當然我們可以透過一些手段來加快 Main Thread 的流程，例如使用 `<script async>` 的非同步載入檔案來避免 Main Thread 被暫停住，或是盡可能地只先載入必要內容。不過這邊只先大概說明，後續文章會進行比較完整的補充。

### Compositor Thread
[Compositor Thread Architecture](https://dev.chromium.org/developers/design-documents/compositor-thread-architecture)
上面有提到 Composite 這個部分，在現代瀏覽器中，它被分成獨立的 Thread，主要目的是藉此提高捲動畫面或是動畫呈現的速度。前面有提到在 Main Thread 的 paint 階段中會產生即將繪製在螢幕上的物件群，這些物件會被複製一份到 Compositor Thread 中，然後透過 GPU process 輸出在畫面上。

當使用者捲動畫面時，原本流程是要在 Main Thread 重新進行 Layout / Paint 的行為，不過透過 Compositor Thread 的協助，可以在最短時間內使用 Cache Tree 去輸出網頁畫面，同時 Compositor Thread 也會通知 Main Thread 進行re-layout / re-paint，並且將更新後的資料逐步地呈現在畫面上，而不會因為 Main Thread 需要執行複雜的運算導致畫面停擺。

![study-scroll-event.png]({{ site.url }}/assets/images/study-scroll-event.png)

## 結論
這篇最主要的目的是概要地介紹瀏覽器渲染網頁頁面的流程，透過流程敘述，可以獲得更多關於改善畫面呈現速度的啟發。例如我們可以知道當在進行 CSS parsing 或是 JavaScript 執行的時候，會佔用 Main Thread，所以在改善部分，可以嘗試將 CSS 分成不同階段進行讀取和 parsing ，以及把工作量較大的 JavaScript code 分成不同階段來逐步執行，以加快整體畫面的呈現時間。

## Reference
1. [How browsers work](http://taligarsiel.com/Projects/howbrowserswork1.htm#Webkit_CSS_parser)
2. [Critical Rendering Path](https://developers.google.com/web/fundamentals/performance/critical-rendering-path/render-tree-construction)
3. [A Quick Overview of Chrome's Rendering Path](http://www.masonchang.com/blog/2016/10/10/a-quick-overview-of-chromes-rendering-path)