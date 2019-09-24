---
layout: single
title:  "Implement Memory Allocator"
date:   2019-09-24 08:00:00 +0800
categories: 
  - C
  - Memory Allocator
toc: true
toc_icon: "cog"
---
## 前言

在很多程式語言都會看到 memory allocator，也可以看到陸續發表的 allocator 實作方式，例如 microsoft [mimalloc](https://github.com/microsoft/mimalloc)。與其用看 source code 的方式來了解其原理，倒不如從基本學起，並且從實作過程中了解到為什麼他們要這樣設計 allocator。

## 原因
在程式開始執行時，我們可以預期會有兩塊 default memory space (system stack, heap) 分配，其中 system stack 負責存放 function return address 或是 local variable 等，而 processor 的 stack pointer 會指向 system stack 的初始位置(可能往下長或往上長)。另外的 heap 則是供 dynamic memory allocation 使用。

> Dynamic Memory Allocation: The mechanism by which storage/memory/cells can be allocated to variables during the run time

(圖待補)

### 1. 為什麼一定要有 dynamic memory allocation
為了節省記憶體使用。很多時候，要等到執行當下才能夠確定是否需要此記憶體空間。最簡單的例子就是 array `int char[255];` ，如果一開始就 allocate 很大一塊 array space，結果實際使用卻不需要這麼多，就顯得有些浪費。

既然 dynamic memory allocation 如此重要，那我們就需要一個工具管理這塊 heap 空間，讓這有限的空間可以被有效運用，而這也就是 Memory Allocator 的職責所在。

有效運用這詞感覺很抽象，簡單來說就是如果隨意將 object 配置 在 heap 任意處，一開始可能用得很開心，等到後來會造成 fragmentation 問題，太多散布的小空間導致 size 較大的 object 無法被分配。

另外，要善用記憶體，除了分配在適當的位置之外，還需要妥善地被釋放掉。例如我們在 C 中手動 `free` 掉佔用的記憶體，或著是其他高階語言有提供 garbage collection 機制，都是此類的體現。

### Example for User-Space Memory Allocator

當我們使用特定程式語言開發應用程式時，該語言其實就有對應的 user space memory allocator。舉例來說，C 程式常常使用 malloc，其 default memory allocator 實作就是 `ptmalloc`；而 Go 程式則是 `TCMalloc` 改良版。

## Implement Memory Allocator

### Stack-Based Allocator

以上概述了一些觀念，接下來就實做簡單的 memory allocator 來加深印象。首先最簡單又最常用的莫過於 Stack-Based Allocator。例如 system stack 就是這類型的 allocator。

在一些情況下，我們可以自己實作 stack allocator 來因應自己的應用場景。例如在 `Game Engine Architecture` 這本書中有提到，簡單 single thread 關卡制的遊戲，很適合使用這類 memory allocator。當一個關卡開始時，allocate 該關卡資源，玩家前往下一個關卡，只需要把 stack pointer 退回到初始點，然後 allocate 下一個關卡的資源就好。

這樣聽起來很基本，不過想想，如果這種可預期的 memory allocation 行為，可以使用更單純的 allocator 來處理，對於效能上就能有顯著地提升。

當然 stack allocator 還有更多變化用法，例如 `double-end stack allocator`。一樣是舉遊戲開發為例子:

![](https://i.imgur.com/s8ohZP1.png)

當玩家在破 A 關卡的時候，可以先將壓縮過的關卡 B load 到 stack 的另一端，然後玩家破完 A 關後，就可以直接將關卡 A pop 掉，解壓縮關卡 B 並 allocate 到一端 stack，接著關卡 C 依序處理。

### Advantages
1. 避免 external fragmentation
由於是連續性地 allocate 或 free 記憶體，因此不會出現 external fragmentation 的狀況。

> External fragmentation arises when free memory is separated into small blocks and is interspersed by allocated memory. 

![resource](https://upload.wikimedia.org/wikipedia/commons/thumb/4/4a/External_Fragmentation.svg/900px-External_Fragmentation.svg.png)

至於 internal fragmentation 部分，就要看具體實作方式，如果在 push 階段 allocate 固定 size 但是實際 object 大小不一時，就會產生 internal fragmentation，但是如果這個 allocator 是用在特定情況下，且 memory usage 已妥善計算過，那就可以避免此狀況發生。

2. 實作簡單

### Disadvantages

1. 必須依照特定順序 Free objects。
畢竟是 stack 結構，所以要依照 LIFO 順序來 free 資源。

3. Resource Contention
因為實作方式簡單(當然也可以做到很複雜啦...)，所以無可避免就會有 resource contention 問題，更不用說如果 multi-thread 一起使用，並且同時 free 資源的情況下，很容易就會順序出錯。所以前面才說這是用於簡單線性遊戲，這樣就可以少去這類問題。

### Implementation

最後就來實作 stack allocator。

```c
struct allocate_state {
  struct allocate_state *next;
  char *addr; // record the start address for validation.
  size_t n; // record the size of allocated object.
};

struct stackAllocator {
  char *top_addr;
  char *bottom_addr;
  size_t length;
  struct allocate_state *head;
  pthread_mutex_t mu;
};

// A fake resource for test
struct game_resource {
  int level;
  char name[50];
};

extern bool init_allocator(struct stackAllocator *allocator, size_t length);
extern void *allocate(struct stackAllocator *allocator, size_t n);
extern bool deallocate(struct stackAllocator *allocator, void *addr);
```

#### Init allocator

```c
bool init_allocator(struct stackAllocator *allocator, size_t length)
{
  char *addr;
  allocator->length = length;
  allocator->head = NULL;
  pthread_mutex_init(&allocator->mu, NULL);

  // Allocate a memory pool.
  addr = (char *)mmap(NULL, allocator->length, PROT_READ|PROT_WRITE, MAP_SHARED|MAP_ANONYMOUS, -1, 0);
  
  if (addr == MAP_FAILED) {
    perror("init allocator");
    return false;
  }

  allocator->top_addr = addr;
  allocator->bottom_addr = allocator->top_addr + allocator->length;
  
  return true;
}
```

#### allocate method

```c
void *allocate(struct stackAllocator *allocator, size_t n)
{
  pthread_mutex_lock(&allocator->mu);
  char *temp = allocator->top_addr;
  
  if (temp + n >= allocator->bottom_addr) {
    DEBUG("No more space for allocation");
    temp = NULL;
    goto unlock;
  }
  allocator->top_addr += n;

  size_t size = sizeof(struct allocate_state);
  allocator->bottom_addr -= size;

  struct allocate_state* state = (struct allocate_state *) allocator->bottom_addr;
  state->next = NULL;
  state->addr = temp;
  state->n = n;

  if (allocator->head == NULL)
  {
    allocator->head = state;
  }
  else {
    state->next = allocator->head;
    allocator->head = state;
  }

unlock:
  pthread_mutex_unlock(&allocator->mu);

  return temp;
}
```

#### deallocate method

```c
bool deallocate(struct stackAllocator *allocator, void *addr)
{
  bool result = false;
  pthread_mutex_lock(&allocator->mu);

  if (allocator->head == NULL) {
    DEBUG("allocate state is null");
    goto unlock;
  }
  if (allocator->head->addr != addr) {
    DEBUG("addrsss mismatched: target %p state %p", addr, allocator->head->addr);
    goto unlock;
  }

  allocator->top_addr -= allocator->head->n;
  size_t size = sizeof(struct allocate_state);
  allocator->bottom_addr += size;
  
  allocator->head = allocator->head->next;
  result = true;

unlock:
  pthread_mutex_unlock(&allocator->mu);

  return result;
}
```

## References
1. [Memory – Part 6: Optimizing the FIFO and Stack allocators](https://techtalk.intersec.com/2014/09/memory-part-6-optimizing-the-fifo-and-stack-allocators)
2. [Memory management](https://en.wikipedia.org/wiki/Memory_management)
3. Game Engine Architecture
4. [std::allocator is to allocation what std::vector is to vexation](https://www.youtube.com/watch?v=LIb3L4vKZ7U)
5. [Memory – Part 4: Intersec’s custom allocators](https://techtalk.intersec.com/2013/10/memory-part-4-intersecs-custom-allocators)
