---
layout: single
title:  "Jekyll"
date:   2017-08-24 10:55:11 +0100
categories: UsefulTools
---
這是第一次用`Jekyll`寫紀錄，雖然我Project是使用Jekyll的概念來開發的，不過實際上我並沒有真的使用它過，主要原因是怕自己想法會因為這樣而被定型，所以當初在開發的時候，只有參考Jekyll網站上的流程圖而已。現在Project的Prototype已完成，就來使用一下，看有什麼可以改進的地方。就整體架構來說，其實差沒有很多，不過Jekyll在程式上分得更細，提供相當多的參數供開發者設定，以及很有彈性而且完整的template render，這些都是值得我後續學習。

This my first time to write articles with Jekyll. To be honest, I'd never use it before although my project was developed by using the concept of static site generators. The reason was my idea might be restricted due to the experiences of Jekyll. So I just read the process of Jekyll during project development. But now I thought it is time to try this system. I hope I could get some ideas that how can I improve my system.

Overall, Jekyll provides much more functions that enable developers to set up their Web sites. Plus, the template render is really flexible and comprehensive that is worth referring.

先看**data serialization**部分，Jekyll使用`YAML`來將資料內容(key-documents)放入template指定位置中，而我則是用非常原始的`XML`。當初會選用XML很單純只是因為Java的標準library有支援XML parse，再加上code只有我可以看得到（畢竟是給non-CS users的系統，跟Jekyll這種給開發者使用的不同），所以就沒有去挑其他的data serialization language。不過相較於`Json`來說，YAML真的容易讀許多，Json架構夾雜很多大框小框，會需要一些時間才能理解整個資料架構，而YAML少了很多框框，不但寫起來和看起來都舒服許多。


再來看**layout**部份，Jekyll是將一整頁的html拆成好幾個部分，然後在render階段include到主頁面中，而我的系統必須要額外再對DOM parse，讓使用者可以直接透過UI去修改這些attribute設定。另外，我在CSS樣式部份有額外再進行一次template，就是提供一個CSS template file，然後使用一個XML去紀錄用戶修改的CSS值，最後將template和值render出一個完整的CSS file.這也是和Jekyll比較不一樣的地方。


![Layout Screenshot]({{ site.url }}/assets/images/project_layout.jpg)
