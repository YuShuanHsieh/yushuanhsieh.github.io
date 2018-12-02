---
layout: single
title:  "Microprocessor System Lab. - Cross Compiler"
date:   2018-11-04 08:00:00 +0800
categories: MCU
---
## 前言
其實整個課程已經看完了，只是因為寫 blog 需要準備很多資料，畢竟有些部分老師快速帶過，所以生產文章的速度遠不及看課程速度XD 這次要說的是 Cross Compiler，因為如果是安裝像是 `SW4STM32` IDE，它所有 cross compiler 設定都已經備妥妥了，使用者只要按一個蟲蟲鍵便能快速 Debugger，不過實際上它背後執行許多程序，只是因為都被自動配置好了，因此使用者不太需要去處理這些額外的環境設定。

## Cross compiler
當我們在開發 Embedded software 時，通常情況下，都是在自己所使用、性能比較好的 host PC 進行程式碼編寫和 compile，接著把 compile 過後的執行檔燒在開發板上（畢竟開發板的執行效率不如自己手上的電腦好，如果直接使用開發板來 compile，會 compile 到天荒地老吧）。

不過，大部分人使用的電腦都是 x86 系列，如果直接在電腦上使用一般 compiler，就只會編出僅供 x86 辨識的執行檔，而想要 compile 出其他 CPU 可以辨識的執行檔，就必須透過 **cross compiler** 的協助才行。

![img](http://1.bp.blogspot.com/-vcFB6vudSTs/UvIFwsB5zMI/AAAAAAAADw8/sIuPoi6Fkes/s1600/toolchain_definintion.bmp)
*source: http://abhishekmourya.blogspot.com/2014/02/cross-compiling-toolchain.html*

至於 cross compiler 取得方式也很簡單，像是使用 ARM 系列 CPU，就能直接在官網上下載到專屬的 [GNU Embedded Toolchain for Arm](https://developer.arm.com/open-source/gnu-toolchain/gnu-rm/downloads) 來幫你處理這些 cross compile 問題。

如果使用 IDE，我們也可以查看他的 builder 設定，來得知 IDE 是怎樣設定 cross compiler 。以 `SW4STM32` 為例，打開它 project setting，並點選 C/C++ builder，就會看到相關的 compiler 設置。

```
arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16 -DSTM32 -DSTM32F4 -DSTM32F401RETx -DNUCLEO_F401RE -DDEBUG -O0 -g3 -Wall -fmessage-length=0 -ffunction-sections -c
```

### What is arm-none-eabi- ?
從上面可以發現，同樣是使用 gcc，上面的 gcc 加了 `arm-none-eabi-` 這個 prefix，那麼 `arm-none-eabi-` 是什麼呢？簡單來說， cross compiler 有通俗的命名原則。

**arch-vendor-(os-)abi**
- arch is for architecture: arm, mips, x86, i686...
- vendor is tool chain supplier: apple, Codesourcery, Linux, os is for operating system: linux, none (bare metal)
- abi is for application binary interface convention: eabi(embedded application binary interface), gnueabi, gnueabihf

如此一來，只要看到 cross compiler 的命名方式，很快就可以知道它 compile 出來的檔案適合用在怎樣的平台上。

舉例來說，我的 STM32F4 project 最基本就包含 `arm-none-eabi-gcc` 和 `arm-none-eabi-as` (for assembly language)，讓我可以編寫 C 和 assembly 兩種語言的程式碼。

## Config Linker script with cross compiler
通常我們在把 C code file compile 成可以被執行的 binary 執行檔，會經過以下流程，其中很重要的是，我們必須把所需的 object files 透過 Linker 連結在一起，才能 compile 出完整的執行檔。所以在使用 cross compiler 時，也必須告知其 linker file。

```
arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -mfloat-abi=hard -mfpu=fpv4-sp-d16 -T"/my-project/MCU/mcu-example/LinkerScript.ld" -Wl,-Map=output.map -Wl,--gc-sections -lm
```

![imgs](https://images.slideplayer.com/27/9191896/slides/slide_6.jpg)

## 結論
Cross compiler 可以讓我們在開發不同平台的軟體時， compile 出其對應的執行檔案。除了上面例子有提到的 `arm-none-eabi-gcc`，還有其他像是 `arm-none-linux-gnueabi-gcc` 這些不同 compiler，如果之後接觸到 Embedded Linux System，就必須換用到適合 Linux 的 cross compiler 了！

## Reference
- https://slideplayer.com/slide/9191896/
- http://ocw.nctu.edu.tw/course_detail-v.php?bgid=9&gid=0&nid=246&v5=1_MWy4ReHIE