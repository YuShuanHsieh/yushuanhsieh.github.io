---
layout: single
title:  "Side Project for daily study trello"
date:   2019-06-14 08:00:00 +0800
categories: WebDevelopment
---
![study]({{ site.url }}/assets/images/side-project.png)
## 前言
用 Trello 紀錄自己的每日學習進度也好一陣子了，雖然 Trello board 搭配 plugin `Calendar` 很好用，但是卻有資訊分散在各張卡的問題。因此為了便於在月末寫當月學習報告，以及整理所有曾經讀過的 article/post link，就開發了一個小工具 `trello-transform` 來從 trello cards 中擷取資訊。其實這個 side project 寫了有一段時間了，之前曾經立志要寫一個有前後端的小網站，這樣不會寫 code 的人也可以使用，但是後來就有點擱置了 XD 原因是我把這個功能寫成一個小框架，以因應每個人紀錄習慣不同，不過這樣的調整就需要會寫一點 code 並知道如何修改，對於不會寫 code 的人來說，可能便利性就沒這麼高。另外就是寫成網頁反而不利於我寫 blog，基於這個 side project 本來就是希望先滿足自己需求，因此後來目標就改成就先以 cli tool 實作為主。

現在這個 side project 成為我每個月必使用的工具，用來擷取學習日誌和 links，除了寫 blog 之外，後續還可以很方便地回顧之前看過的文章，可以說是目前寫過最實用的 project XD 

## 架構規劃

在寫這個 project 的時候，第一個想法就是要可以彈性加入不同的 methods 來產出各種格式的 output，這樣我在之後如果有其他輸出需求，就可以直接新增 method 又不會改到主架構。另外就是我習慣分成不同 list 來紀錄不同月份的日誌，所以 list selector 也很重要。

![study]({{ site.url }}/assets/images/side-project-2.png)

由於目前是朝向簡單工具的開發，所以就採用最無腦的方式：從 Trello 網站 export json file 然後再 import 進來，這樣程式也不用處理 auth 問題，不過可能之後還是會再接 Trello API 來讓工具更自動。 

## 實作細節

### Accumulator

Accumulator 目的是在進行 card loop 的時候，累加處理過後的結果。因為結果的值可能是各種不同 type struct，所以就使用 interface。另外，最後的值根據不同 struct 可能有不同輸出方式，因此需要實作 `String() string` 來支援 output。

> If an operand implements method String() string, that method will be invoked to convert the object to a string, which will then be formatted as required by the verb (if any).

```go
type Accumulator interface {
	String() string
}

// Implementation
type markdownURLs []string

func (m markdownURLs) String() string {
	var builder strings.Builder
	for _, v := range m {
		builder.WriteString("- " + v + "\r\n")
	}
	return builder.String()
}
```

### Transformer

有了 Accumulator 之後，就來實作 Transformer method. Transformer 會傳入三個 params:
1. Context 中含有一些 list, label information，在藉由 card list ID 反查可以獲得更多相關資訊。
2. Accumulator 當前的值，可以調整內容並且返回最新的值。這邊有點 tricky 是必須先檢查這個 Accumulator interface 是否有 concrete type，不然會報 error。
3. *trello.Card 目前 loop 輪到的 card。其實 card 應該是不能改，照理說不要使用 pointer 比較妥當。不過因為 trello card 完整結構其實蠻大的，擔心會有比較大負擔的 copy，所以就傳 pointer。

```go
type Transformer func(Context, Accumulator, *trello.Card) (Accumulator, error)

func MarkdownURLTransformer(
	ctx Context,
	acc Accumulator,
	c *trello.Card,
) (Accumulator, error) {
  value, ok := acc.(markdownURLs)
  // Type assertion to prevent nil value
	if !ok {
		value = markdownURLs{}
	}
	reg, err := regexp.Compile(`[[][^]]+[]][(][^)]+[)]`)
	if err != nil {
		return acc, err
	}
	targets := reg.FindAllString(c.Desc, -1)
	return append(value, targets...), nil
}
```

### Selector

Selector 主要用於判斷這張 trello card 是否是我們要的。如果沒有設置 Selector，就會讓所有 card 都進入 Transformers。
```go
type Seletor func(Context, *trello.Card) bool

func ByListNames(names ...string) Seletor {
	m := make(map[string]struct{})
	for _, name := range names {
		m[name] = struct{}{}
	}
	return func(ctx Context, c *trello.Card) bool {
		list, ok := ctx.Lists[c.IDList]
		if ok {
			if _, ok := m[list.Name]; ok {
				return true
			}
		}
		return false
	}
}
```

### Use

最後簡單附上整個 transform 套用 transformer 和 selector 並且執行的方式。
```go
trans := transform.New(data)
trans.Select(transform.ByListNames("List1", "List2"))
trans.Use("headers", transform.CardHeaderTransformer)
trans.Exec()

for k, v := range trans.GetAllResult() {
  fmt.Printf("%s: \r\n", strings.ToUpper(k))
  fmt.Println(v)
}
```

