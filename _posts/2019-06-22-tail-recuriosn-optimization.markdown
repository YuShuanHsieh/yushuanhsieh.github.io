---
layout: single
title:  "Tail Recursion Optimization"
date:   2019-06-14 08:00:00 +0800
categories: WebDevelopment
---
## 前言
最近在實驗效能分析時，有使用到 Tail Recursion 寫法，因此也好奇在 Go 中是否有跟 C 一樣進行 Tail Recursion optimization 優化。在文章中，首先會用 C asm 來說明 tail call optimization，接著 dump Go asm code 來觀察結果。

## Tail Call
在說明 Tail Recursion Optimization 之前，先來說說 [Tail Call](https://en.wikipedia.org/wiki/Tail_call)。 Tail Call 概念很簡單，就是在 function 的最後語句執行另一個 function call。

```c
int search(int data)
{
  return advanced_search(data);
}
```

要注意的是， `Tail Call` 意指最後的行為只允許使用 function call，所以以下的寫法就不是 `Tail Call`，因為它需要先進行 `2 * 1 + funcB(a-1)` 的運算，才能返回。

 ```c
int non_tail(int a)
{
  if (a < 1)
  {
    return 1;
  }
  return 2 * 1 + funcB(a-1);
}
 ```

## Tail recursion

Tail Call 可以應用在 Recursion 上，在尾端再次呼叫自身程式，就被稱為 `Tail recursion`。

```c
int search(int data)
{
  // some conditions..
  
  return search(data); // call itself.
}
```

提到 Tail recursion，很多人常用的例子是透過將 Recursion 改寫成 Tail recursion 的方式，來改善 **Find Fibonacci Number** 的效能。回想當初我們在初學習實作 Fibonacci Number 所使用的 Recursion 寫法：

```c
int recursion_fib(int n)
{
  if (n <= 1) 
    return n;
  return  recursion_fib(n - 1) + recursion_fib(n - 2);
}
```

可以改寫成 `Tail recursion` 版本：

```c
int tail_recursion_fib(int n, int left, int right)
{
  if (n == 0)
  {
    return left;
  }
  return tail_fib(n-1, right, left + right);
}
```

透過以上改寫，我們就能避免掉因為 Recursion 造成的 [Fibonacci Trees](http://www-inst.eecs.berkeley.edu/~cs61bl/r//cur/trees/fibonacci-tree.html)，進而提升執行效率（[Fibonacci with recursion and tail recursion 效率比較](https://hackmd.io/@sysprog/rJ8BOjGGl?type=view#Fibonacci-sequence)）。

不過，單就上面我們無法看出 `Tail recursion` 的優勢，以上面提到的 Fibonacci Number 為例，將 Recursion 改成 Tail recursion，因為減少一個自身的 function 運算，避免 Recursion 發散，本來就對效能有所提升。

## Tail recursion Optimization

之所以採用 `Tail recursion` 寫法的最大原因，是因為很多語言可以透過 compile 將 `Tail recursion` 的效能最佳化，以避免因為執行到 function 而需要為它建立新的 [Stack Frame](https://www.cs.auckland.ac.nz/software/AlgAnim/stacks.html)。

![stack frame](https://www.cs.auckland.ac.nz/software/AlgAnim/fig/stackframe.gif)

舉例:

```c
int tail(int a, int result)
{
  if (a == 0)
  {
    return result;
  }
  return tail(a-1, 1 + result);
}
```

![study]({{ site.url }}/assets/images/tail-recursion-2.png)

可以看到在 `Tail recursion Optimization` 部分，只需要把處理過後的數值儲存起來，接著跳轉到同一個 function 最前端重複執行即可。反過來看，如果沒有經過優化的話，那麼就需要耗費 stack 空間為相同的 function 建立新的 stack frame。

## 實驗分析

現在我們就來實驗一下會在有無 `Tail recursion Optimization` 的情況下，會有多少差距。

### 機器
```
Raspberry B3 ARM Cortex-A53 64 bit
```

### test code
```c
int tail(int a, int result)
{
  if (a < 1)
  {
    return result;
  }
  return tail(a-1, 2*1 + result);
}
```

首先，先把 c file compile 成沒有優化過後的 assembly code file：

```
gcc -S tail.c
```

可以看到 assembly code:

```asm
tail_fib:
        stp     x29, x30, [sp, -32]!
        add     x29, sp, 0
        str     w0, [x29, 28]
        str     w1, [x29, 24]
        str     w2, [x29, 20]
        ldr     w0, [x29, 28]
        cmp     w0, 0
        bne     .L2
        ldr     w0, [x29, 24]
        b       .L3
.L2:
        ldr     w0, [x29, 28]
        sub     w3, w0, #1
        ldr     w1, [x29, 24]
        ldr     w0, [x29, 20]
        add     w0, w1, w0
        mov     w2, w0
        ldr     w1, [x29, 20]
        mov     w0, w3
        bl      tail_fib
.L3:
        ldp     x29, x30, [sp], 32
        ret
        .size   tail_fib, .-tail_fib
        .section        .rodata
        .align  3
```

然後簡單把 `bl tail_fib` 改成 goto 到原有 stack frame head 的行為：

```asm
tail_fib:
        stp     x29, x30, [sp, -32]!
        add     x29, sp, 0
	b	.L5
.L5:
        str     w0, [x29, 28]
        str     w1, [x29, 24]
        str     w2, [x29, 20]
        ldr     w0, [x29, 28]
        cmp     w0, 0
        bne     .L2
        ldr     w0, [x29, 24]
        b       .L3
.L2:
        ldr     w0, [x29, 28]
        sub     w3, w0, #1
        ldr     w1, [x29, 24]
        ldr     w0, [x29, 20]
        add     w0, w1, w0
        mov     w2, w0
        ldr     w1, [x29, 20]
        mov     w0, w3
	b	.L5
.L3:
        ldp     x29, x30, [sp], 32
        ret
        .size   tail_fib, .-tail_fib
        .section        .rodata
        .align  3
```

### Result
在 recusion 數量少的情況下，優化後的效率沒有太多變化，不過當到了 recusive function call 多的時候，就會看出兩者之間的時間差距。

![study]({{ site.url }}/assets/images/tail-recursion.png)