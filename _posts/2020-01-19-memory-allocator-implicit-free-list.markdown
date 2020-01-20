---
layout: single
title:  "Implement Memory Allocator - Implicit Free List"
date:   2020-01-19 08:00:00 +0800
categories: 
  - C
toc: true
toc_icon: "cog"
---
## 前言
由[前篇](https://yushuanhsieh.github.io/c/memory%20allocator/stack-memory-allocator/)文章可以知道，使用 stack data structure 是最簡單的 memory allocator 入門寫法，但是卻會造成一些問題，例如不能任意順序 free 等。既然如此，我們就換成 list 來實作 memory allocator，解決使用 stack 實作的問題。

除了實作之外，也會討論到幾個實作 allocator 上遇到的問題，例如 sbrk，以及 minimum alignment。

## Memory allocator 要素

在實作之前，先歸納一下 allocator 需求。 [Computer Systems(book)](https://www.tenlong.com.tw/products/9780134092669) 與 [CS 4400 – Computer Systems](https://my.eng.utah.edu/~cs4400/) 說明幾個 allocator 要素，包含：

1. Handling arbitrary request sequences
2. Aligning blocks (8 bytes on 32-bit and 16 bytes on 64-bit)
3. Not modifying allocated blocks (compaction is not allowed)
4. How does `free` know an allocated block’s size?
5. How is unallocated space represented?
6. How do we keep track of free blocks?
7. How do we choose an appropriate free block?

依照上列條件，可以知道 stack 形態的 allocator 其實並不能滿足基本 allocator 需求(例如：arbitrary request sequences)，因此後續在實作 implicit free list 的時候，也會根據上列來一一檢視是否有達成需求。

## 基本 Implicit Free List

起初概念，將一個連續性記憶體分成好幾個 blocks ，使用 block header 來標示每個 block 的 size 和 allocated 狀態，並且透過 header 的 size 得知道下一個 block 的初始位置，如此一來就可以遍歷所有的 blocks。

### Block format

Block size 應包含 header + payload size (除了最後一個用為終止線的 block 之外)，例如：header struct 是 8 bytes 而要 allocate 的 data size 是 16 bytes，那 block size = 8 + 16(尚未考慮 alignment)。**Block size 數字會影響後續進行合併 (coalesce) 計算**，所以很重要。

![study]({{ site.url }}/assets/images/memory-allocator-1.png)

### Aligning blocks

malloc 應該遵守 alignment 規定：**8 bytes on 32-bit and 16 bytes on 64-bit**。(其原因在下方補充內容中會提及)，所以當我們在計算 block size 時，要加入 alignment 條件，即：

```c
#define HEADER_SIZE 8
#define ALIGNMENT 16
#define ALIGN(size) (((size) + (ALIGNMENT-1)) & ~(ALIGNMENT-1))


size_t block_size(size_t payload_size)
{
  return ALIGN(HEADER_SIZE + payload_size);
}
```

**重點：align 所有 block size，而不是只有 payload size。**

另外一個重點，**malloc return 的 payload address 必須符合 16-byte-aligned**，因此必要時應調整第一個 header 的起始位置，讓 `payload address & 0xF` 等於 0。

e.g.
整個 heap 的起始位置是: 0xc000
第一個 header 的起始位置應為：0xc000 + 8 
如此一來， payload 的起始位置就會為：0xc008 + 8(header size)


### 初始化 allocator

首先，先跟 kernel 索取需要的空間大小 (sbrk or mmap)，然後建立一個 header 包含可使用的 bytes 數量，以及最後一個代表終止 header。

![study]({{ site.url }}/assets/images/memory-allocator-3.png)

假設 header struct size 是 8 byte，那圖示中總共的 heap 需求空間就是 128 + 8 header size。

### Allocate

如果需要一個 size 為 16 bytes 的空間，則會從第一個 block 開始查找有沒有適合的空block，如果有的話，則將該 block 的 `allocated` 改成 1。

![study]({{ site.url }}/assets/images/memory-allocator-4.png)


假如 block 的 size > allocate 所需求的 size + Header size，則會計算出剩餘的 size 大小，並且新建一個 header 在 block 後方位置(將原先的 block 拆分成兩個)。

最後回傳 block address + header size bytes 的memory 位置(payload 的起始位置)。

### Free

當執行 free 的時候，只要將 object address - header size bytes 來找到 header address，並且將 header 的 allocated 設為 0 即可。

![study]({{ site.url }}/assets/images/memory-allocator-5.png)

當然這樣會導致原先一個大 block 被拆分成很多零碎的小 blocks，進而造成 **external fragmentation** 問題。後面會說明將這些因為 free 行為而產生的零碎 blocks 合併(coalesce)的方式。

### Traveling all blocks

從頭開始遍歷所有的 blocks，然後在 block size 為 0 的地方終止。

![study]({{ site.url }}/assets/images/memory-allocator-6.png)

### 檢視

完成了基本版的 implicit free list 後，來檢視一下一開始提及的條件是否有滿足：

#### 1.Handling arbitrary request sequences (O)
從實作方式可以得知 free 和 allocate 不受順序限制。
#### 2.Aligning blocks (8 bytes on 32-bit and 16 bytes on 64-bit) (O)
#### 3.Not modifying allocated blocks (compaction is not allowed)(O)
#### 4.How does `free` know an allocated block’s size?
使用 block header 來得知 size。
#### 5.How is unallocated space represented?
用 block 中的 allocated 來標示此 block 目前狀態。
#### 6.How do we keep track of free blocks?
使用 block chain 結合 header 中的 allocated 來追蹤所有 free blocks。
#### 7.How do we choose an appropriate free block?
透過遍歷所有 blocks 找出適合 size 的 block。

## 改善 Implicit Free List

上面提到基本的 implicit free list 作法，接下來說明幾種可以改善的方向：

### 1. 調整 header 格式

![study]({{ site.url }}/assets/images/memory-allocator-7.png)

原先的 header struct 是

```c
typedef struct {
  size_t size;
  size_t allocated;
} block_header;
```

但是由於 alignment 關係，一開始的幾個 bit 值皆為 0 ，因此可以直接使用這些 bit 來表示 allocated 值即可。(e.g. alignment 為 8 bytes，則 0-2 bits 皆為 0)

### 2. 改善 - Coalesce

在基本版本的 implicit free list 中，有可能因為 allocate 和 free 的連續操作，造成原先大塊的 block 被拆分成小塊的 blocks，進而導致再次 allocate 大 size 的 object 時會找不到適合的 block。

![study]({{ site.url }}/assets/images/memory-allocator-9.png)

為了解決這個問題，在 free object 步驟時，需要適時合併前後空的 block，以回復成較大的 block。

![study]({{ site.url }}/assets/images/memory-allocator-10.png)

但在合併時會有一個問題，從 block header 中可以知道下一個 block 的 size 和 allocated 狀態，但該如何知道上一個 block header 的值呢？我們可以在 **block 尾端加入 footer**，然後即可透過 `(char *)header - FOOTER_SIZE` 的方式來取得前一個 block 的狀態。

![study]({{ site.url }}/assets/images/memory-allocator-8.png)

### 3. 改善 - 當 block 為 unallocated 狀態時才加入 footer

雖然在 block 尾端加入 footer 是一個不錯的辦法，但是卻會減少可以分配的空間，所以我們調整一下方式，改成只有當 block 為 unallocated 狀態時才加入 footer 值，而 footer 與 payload 空間共用，這樣就不會造成浪費。

![study]({{ site.url }}/assets/images/memory-allocator-11.png)

## Advantges

Implicit free list 最大的優點就是易於實作，邏輯相較起來比較簡單。

## Problems

### 花費較多時間在找尋適合的 block

由於 implicit free list 是 allocated 以及 unallocated blocks 混雜在一起，因此在找尋適合大小的 block 時會花費比較多時間。此外，如果我們是使用 first fit 策略(找到第一個有足夠大小的 block 即返回)，那就有可能造成額外的 external fragmenation 問題。但如果改用 best fit策略(找到最適合的大小 block)，那就要遍歷所有的 blocks 進行 size 比較，因此而花費更多尋找時間。

## 實作

實作部分可以參考 [Github allocator-example](https://github.com/YuShuanHsieh/allocator-example/blob/master/free_list/mm.c)，這其實是 [CS 4400 – Computer Systems](https://my.eng.utah.edu/~cs4400/) 的程式作業XD 而實作結合了上面所提到的三個改善方式。可以透過 `make test60` 來進行測試。

```c
#include <stdint.h>
#include <stdio.h> 
#include <unistd.h>
#include <errno.h>
#include <stdbool.h>
#include <stdint.h>
#include <sys/mman.h>

#include "mm.h"

typedef uint64_t header;

#define HEADER_SIZE sizeof(header)
#define ALIGNMENT 16
#define ALIGN(size) (((size) + (ALIGNMENT-1)) & ~(ALIGNMENT-1))
#define BLOCK_SIZE(header_ptr) (*(uint64_t *) header_ptr & ~0xF)
#define BLOCK_ALLOCATED(header_ptr) (*(uint64_t *) header_ptr & 1)
#define BLOCK_FOOTER(header_ptr) ((char *)(header_ptr) + BLOCK_SIZE(header_ptr) - HEADER_SIZE)
#define PREV_BLOCK_ALLOCATED(header_ptr) ((*(uint64_t *) header_ptr & 2) >> 1)
#define NEXT_BLOCK(header_ptr)((char *)(header_ptr) + BLOCK_SIZE(header_ptr))

void *area;

void set_header(void *header, int size, bool allocated, bool prev_allocated)
{
    *(uint64_t *)header = size | allocated | (prev_allocated << 1);
}

void set_footer(void *footer, int size)
{
    *(uint64_t *)footer = size;
}

void mm_init(void *heap, size_t heap_size)
{ 
  area = heap + HEADER_SIZE;
  heap_size = heap_size - 2 * HEADER_SIZE;
  set_header(area, heap_size, false, true);
  set_header(NEXT_BLOCK(area), 0, true, true);

}

void *find_free_block(void *heap, size_t size)
{
    bool allocated;
    size_t block_size;
    void *block = heap;

    while(1)
    {
        allocated = BLOCK_ALLOCATED(block);
        block_size = BLOCK_SIZE(block);

        if (block_size == 0)
        {
            return NULL;
        } else if (allocated == true || block_size < size)
        {
            block = NEXT_BLOCK(block);
        } else {
            return block;
        }
    };
}

void print_fragment(void *heap)
{
    void *head = heap;
    size_t fragment_size;
    bool allocated;

    while(1)
    {  
        fragment_size = BLOCK_SIZE(head);
        allocated = BLOCK_ALLOCATED(head);

        if(fragment_size == 0)
        {
            break;
        }

        printf(" [%zu, %d] ", fragment_size, allocated);
        head = NEXT_BLOCK(head);
    }
    printf("\n");
}

void *mm_malloc(size_t size)
{
  if (size == 0)
  {
    return NULL;
  }

  int aligned_size = ALIGN(size + HEADER_SIZE);
  void *block = find_free_block(area, aligned_size);

  if (block == NULL)
  {
      printf("space is not enough \n");
      return NULL;
  }

  size_t block_size = BLOCK_SIZE(block);
  
  size_t remainder_size = block_size - aligned_size;

  if (remainder_size > ALIGN(HEADER_SIZE))
  {
    set_header(block, aligned_size, true, PREV_BLOCK_ALLOCATED(block));
    set_header(NEXT_BLOCK(block), remainder_size, false, true);
  } else {
    void *next_header = NEXT_BLOCK(block);
    set_header(block, BLOCK_SIZE(block), true, PREV_BLOCK_ALLOCATED(block));
    set_header(next_header, BLOCK_SIZE(next_header), BLOCK_ALLOCATED(next_header), true);
  }

  return (char *)(block) + HEADER_SIZE;
}


void coalesce_blocks(void *block)
{
    size_t size, prev_size, next_size;
    bool prev = PREV_BLOCK_ALLOCATED(block);
    bool next = BLOCK_ALLOCATED(NEXT_BLOCK(block));

    size = BLOCK_SIZE(block);

    if (prev == false && next == false)
    {
      void *prev_footer = (char *)block - HEADER_SIZE;
      prev_size = BLOCK_SIZE(prev_footer);
      next_size = BLOCK_SIZE(NEXT_BLOCK(block));

      void *prev_header = (char *)block - prev_size;
      int new_size = prev_size + next_size + size;
      set_header(prev_header, new_size, false, PREV_BLOCK_ALLOCATED(prev_header));
      set_footer(BLOCK_FOOTER(prev_header), new_size);

      block = prev_header;

    } else if (prev == false)
    {
      void *prev_footer = (char *)block - HEADER_SIZE;
      prev_size = BLOCK_SIZE(prev_footer);
      void *prev_header = (char *)block - prev_size;

      int new_size = prev_size + size;
      set_header(prev_header, new_size, false, PREV_BLOCK_ALLOCATED(prev_header));
      set_footer(BLOCK_FOOTER(prev_header), new_size);

      block = prev_header;

    } else if (next == false)
    {
      next_size = BLOCK_SIZE(NEXT_BLOCK(block));

      int new_size = next_size + size;
      set_header(block, new_size, false, PREV_BLOCK_ALLOCATED(block));
      set_footer(BLOCK_FOOTER(block), new_size);
      
    } else {
    }
}

void mm_free(void *payload)
{
  if (payload == NULL)
  {
      return;
  }
  void *header = (char *)payload - HEADER_SIZE;
  set_header(header, BLOCK_SIZE(header), false, PREV_BLOCK_ALLOCATED(header));
  
  void *footer = BLOCK_FOOTER(header);
  set_footer(footer, BLOCK_SIZE(header));

  void *next = NEXT_BLOCK(header);
  set_header(next, BLOCK_SIZE(next), BLOCK_ALLOCATED(next), false);

  coalesce_blocks(header);
}
```

## Note

### sbrk() is deprecated

網路上會看到很多範例使用 sbrk ，來延伸 heap 大小，不過卻被人詬病是比較不好的做法，原因是延伸後的 memory range 可能會被使用到 (shared memory)。不過這樣的情形是有機率會出現在 virtual memory 有限的 32 bit 架構上，但對於 virtual memory 很充足的 64 bit 架構來說，每個 process 會有各自的 memory space，不太會發生這樣情形。

針對將 sbrk 換成 mmap 的建議，在[A new malloc(3) for OpenBSD](https://www.openbsd.org/papers/eurobsdcon2009/otto-malloc.pdf) 中有提出不錯的看法，而**Predictability** 是其中一個重要因素，使用 mmap 能夠獲得隨意位置的可用 memory，相對於 sbrk 所產生的連續性記憶位置，更能夠降低因連續位置而造成的潛在問題，例如記憶體位置預測攻擊等。

### Minimum alignment of allocation

聽到 `alignment` 這個詞，可能第一個聯想到 [Data structure alignment](https://en.wikipedia.org/wiki/Data_structure_alignment)，由於 CPU 設計，為了讓 CPU 在讀取或寫入 data 時有較好地表現，因此需要 alignment。

而 malloc 也需要 alignment，不過跟上述所提到的 alignment 目的有所不同。以 C libs 的 malloc 來說，64 bits 架構下為 16 bytes alignment，這聽起來有些奇怪，因為依照 Data structure alignment 的概念，64 bits 其實只需要 8 bytes 就好，為何需要到 16 bytes alignment？  

在 [Minimum alignment of allocation across platforms](http://www.erahm.org/2016/03/24/minimum-alignment-of-allocation-across-platforms/) 文章中有提到此問題，文章中提及過去因為 minimum allocation size 而導致的 crash bug，並且查找各大平台的 minimum alignment 需求皆為 16 bytes alignment on 64-bit，其原因主要是有益於執行效率，例如使用到 SSE instructions 就需要 16-byte-aligned data。 