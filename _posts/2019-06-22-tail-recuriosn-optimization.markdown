---
layout: single
title:  "Tail Recursion in Go"
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
  return tail_recursion_fib(n-1, right, left + right);
}
```

透過以上改寫，我們就能避免掉因為 Recursion 造成的 [Fibonacci Trees](http://www-inst.eecs.berkeley.edu/~cs61bl/r//cur/trees/fibonacci-tree.html)，進而提升執行效率（[Fibonacci with recursion and tail recursion 效率比較](https://hackmd.io/@sysprog/rJ8BOjGGl?type=view#Fibonacci-sequence)）。

不過，單就上面我們無法看出 `Tail recursion` 的優勢，以上面提到的 Fibonacci Number 為例，將 Recursion 改成 Tail recursion，因為減少一個自身的 function 運算，避免 Recursion 發散，本來就對效能有所提升。

## Tail recursion Optimization

之所以採用 `Tail recursion` 寫法的主要原因，是因為很多語言可以透過 compile 將 `Tail recursion` 的效能最佳化，以避免因為執行到 function 而需要為它建立新的 [Stack Frame](https://www.cs.auckland.ac.nz/software/AlgAnim/stacks.html)。

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

## Tail Recursion (Optimization) / Iteration

看到這裡或許你會覺得很奇怪，這樣透過 `Tail recursion Optimization` 所產生的結果不就跟 `iteration` 差不多嗎？

其實 Iteration 與 Tail Recursion 在大部分情況下可以相互轉換。

> All iterative functions can be converted to recursion because iteration is just a special case of recursion **(tail recursion)**. In functional languages like Scheme, iteration is defined as tail recursion.
> (www.ocf.berkeley.edu)

只是差別在於如果沒有經過 Optimization 階段，那麼 Tail Recursion 會需要處理額外的 stack frame，因此比 Iteration 慢。而 Tail recursion Optimization 後，就能有跟 Iteration 相似的表現。

> In functional programming languages, tail call elimination is often guaranteed by the language standard, allowing tail recursion to use a similar amount of memory as an equivalent loop.
> (https://en.wikipedia.org/wiki/Tail_call)

## 使用 `Tail Recursion` 的時機

1. Language 有支援 Tail Recursion Optimization（如果沒有支援，就有可能讓 stack 空間耗盡）。
2. 提升 code 的簡潔與可讀性。
3. 需要操作 recursive data structures 像是 linklist or tree 等
。

## 效能實驗

簡單實驗一下會在有無 `Tail recursion Optimization` 的情況下，會有多少差距。

### 機器
```shell
Raspberry B3 ARM Cortex-A53 64 bit
gcc target: aarch64-linux-gnu
```

### Test code
測試程式則是大家熟悉的 Fibonacci Number
```c
int tail_recursion_fib(int n, int left, int right)
{
  if (n == 0)
  {
    return left;
  }
  return tail_recursion_fib(n-1, right, left + right);
}
```

首先，先把 c file compile 成沒有優化過後的 assembly code file。
（之所以不直接用 -Os 來進行 code 優化，是因為 -Os 會影響很多 assembly code，不好進行評估）

```shell
gcc -S tail.c
```

可以獲得 *.s 的 assembly code file。

```c
tail_fib:
        stp     x29, x30, [sp, -32] // 移動 stack pointer
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
        bl      tail_fib // branch 執行 function
.L3:
        ldp     x29, x30, [sp], 32
        ret
        .size   tail_fib, .-tail_fib
        .section        .rodata
        .align  3
```

然後參照優化行為，把 `bl tail_fib` 改成 goto 到原有 stack frame head：
```c
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
	b	.L5 // 用途類似 x86 的 jmp. 跳至 .L5 避免移動 SP
.L3:
        ldp     x29, x30, [sp], 32
        ret
        .size   tail_fib, .-tail_fib
        .section        .rodata
        .align  3
```

另外也提供  `x86_64-apple-darwin` 的 assembly code，可以很清楚看出 `Tail Recursion Optimization`。

Normal

```c
LBB0_2:
	movl	-8(%rbp), %eax
	subl	$1, %eax
	movl	-16(%rbp), %esi
	movl	-12(%rbp), %ecx
	addl	-16(%rbp), %ecx
	movl	%eax, %edi
	movl	%ecx, %edx
	callq	_tail_fib // call function and then move down SP
	movl	%eax, -4(%rbp)
```

Optimization

```c
LBB0_3:                                 ## =>This Inner Loop Header: Depth=1
	movl	%edx, %esi
	decl	%edi
	movl	%eax, %edx
	addl	%esi, %edx
	incl	%ecx
	movl	%esi, %eax
	jne	LBB0_3 // Use loop
```

### Result
在 recusion 數量少的情況下，效率沒有太多變化，不過當到了 recusive function call 多的時候，就會看出兩者之間的時間差距。

![study]({{ site.url }}/assets/images/tail-recursion.png)

## Recursion in Go

在說明 Tail Recursion in Go 之前，先來說說 Recursion 在 Go 的表現。回想在 default stack size 8MB 使用 Recursion 都有可能會遇到 stack overflow 情況，更何況是在每個 goroutine init stack size 為 2 KB (go version 1.12.X) 開發。

幸好的是使用 Go 時，遇到 stack 沒空間，那麼 runtime 就會幫 goroutine 擴展 stack size，好讓程式能執行下去。不過相對應的代價是需要處理 grow stack 部分而導致影響執行時間。也因此 Recursion 在 Go 下更要謹慎使用。

## Tail Recursion Optimization in Go

終於進入到正式主題了，那個 Go 到底沒有支援 `Tail Recursion Optimization` 呢？

答案是**目前沒有**(go version 1.12.X)。

之前有蠻多相關的討論，例如：
- [proposal: Go 2: add become statement to support tail calls](https://github.com/golang/go/issues/22624)
- [proposal: cmd/compile: add tail call optimization for self-recursion only](https://github.com/golang/go/issues/16798)

主要原因在於：
1. Make stack traces clean
2. 大部分情況下，tail recursion 可以 rewrite 成 loop，所以必要性不是太高

基於上面兩點，Go lang 維護者覺得目前應該專注在其他更重要的議題上，因此這個功能預計在 go 2.0 也不會做。

當然，你也可以用 gccgo 來 compile go file，由於是採用 gcc compiler，所以透過優化選項就可以編譯成 `Tail Recursion Optimization` 版本。

## 實驗分析

### Test code

一樣使用 `Fibonacci Number` Tail Recursion 寫法為例：

```go
func TailRecursionFib(n, left, right int) int {
	if n == 0 {
		return left
	}
	return TailRecursionFib(n-1, right, left+right)
}
```

### Assembly code with Go compiler

```c
"".TailRecursionFib STEXT size=121 args=0x20 locals=0x28
  0x0000 00000 (main.go:38)	TEXT	"".TailRecursionFib(SB), ABIInternal, $40-32
  0x0000 00000 (main.go:38)	MOVQ	(TLS), CX
  0x0009 00009 (main.go:38)	CMPQ	SP, 16(CX)
  0x000d 00013 (main.go:38)	JLS	114
  0x000f 00015 (main.go:38)	SUBQ	$40, SP
  0x0013 00019 (main.go:38)	MOVQ	BP, 32(SP)
  0x0018 00024 (main.go:38)	LEAQ	32(SP), BP
  0x001d 00029 (main.go:38)	FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
  0x001d 00029 (main.go:38)	FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
  0x001d 00029 (main.go:38)	FUNCDATA	$3, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
  0x001d 00029 (main.go:39)	PCDATA	$2, $0
  0x001d 00029 (main.go:39)	PCDATA	$0, $0
  0x001d 00029 (main.go:39)	MOVQ	"".n+48(SP), AX
  0x0022 00034 (main.go:39)	TESTQ	AX, AX
  0x0025 00037 (main.go:39)	JNE	59
  0x0027 00039 (main.go:40)	MOVQ	"".left+56(SP), AX
  0x002c 00044 (main.go:40)	MOVQ	AX, "".~r3+72(SP)
  0x0031 00049 (main.go:40)	MOVQ	32(SP), BP
  0x0036 00054 (main.go:40)	ADDQ	$40, SP
  0x003a 00058 (main.go:40)	RET
  0x003b 00059 (main.go:42)	DECQ	AX
  0x003e 00062 (main.go:42)	MOVQ	AX, (SP)
  0x0042 00066 (main.go:42)	MOVQ	"".right+64(SP), AX
  0x0047 00071 (main.go:42)	MOVQ	AX, 8(SP)
  0x004c 00076 (main.go:42)	MOVQ	"".left+56(SP), CX
  0x0051 00081 (main.go:42)	ADDQ	CX, AX
  0x0054 00084 (main.go:42)	MOVQ	AX, 16(SP)
  0x0059 00089 (main.go:42)	CALL	"".TailRecursionFib(SB) // Call itself
```

### 效能

執行時，可以明顯看出在 grow stack size 時出現的執行時間高峰，進而降低整體速度。

![study]({{ site.url }}/assets/images/stack-size-comp.png)

### Tail Recursion v.s. Iteration

![study]({{ site.url }}/assets/images/fib-comp.png)

如果是 Go 的話，使用 Iteration 的效能會遠比 Tail Recursion 好。

## References

1. https://en.wikipedia.org/wiki/Tail_call
2. https://www.cs.auckland.ac.nz/software/AlgAnim/stacks.html
3. https://www.ocf.berkeley.edu/~shidi/cs61a/wiki/Iteration_vs._recursion
4. https://www.ardanlabs.com/blog/2013/09/recursion-and-tail-calls-in-go_26.html
5. https://gcc.gnu.org/wiki/SplitStacks