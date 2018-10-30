---
layout: post
title:  DSP将flash中的程序复制到sram运行
key:  2018-10-30
tags:
  - DSP
  - Flash
  - Sram
  - 嵌入式
category:  blog
---

![](https://s1.ax1x.com/2018/10/30/i2LMWV.png)


在我们实际的项目当中，有时候仅仅烧写在Flash中的速度跟我们方针的时候是完全不一样的，仿真的时候可以实现的功能可能一烧写进FLASH中就没有了效果，这就需要我们把FLASH中的程序复制到SRAM中运行。<!--more-->在flash中和SRAM之所以不一样，是因为FLSAH的速度实在是有限，一旦遇到一些高速信号，例如高频的PWM波就没辙了，只有把程序复制到SRAM中才能和仿真的时候一样。如果如何烧写不清楚，可以参考上一篇博客 [ DSP烧写进FLASH ](https://renyoulu.com/blog/2018/10/29/DSP_Flash烧写步骤.html)

废话不多说直接上手做。
*注：本文中最后修改好的文件在文末下载链接当中*

## 1.添加DSP28xxx_SectionCopy_nonBIOS.asm到工程目录下

`DSP28xxx_SectionCopy_nonBIOS.asm`中为程序拷贝函数。定义了段名为**copysections**，之后将会在**CMD**文件添加该段。

## 2.修改启动文件DSP2833x_CodeStartBranch.asm

有些时候程序的执行对于速度要求很高，此时就需要将程序整体拷贝到**RAM**中去执行。程序整体拷贝同样需要进行特殊修改和设置。要想详细了解下面设置的原理，可以查找相关资料来详细了解程序启动的过程。
### 1.程序启动的过程一般为：

```mermaid
graph TD;
    code_start-->wd_disable;
    wd_disable-->c_int00-;
    c_int00--->mian（）;
```

为了完成程序的拷贝，需要在C语言初始化之前插入一个程序拷贝的环节，修改之后的流程为：

```mermaid
graph TD;
    code_start-->wd_disable;
    wd_disable-->copy_sections;
    copy_sections--->c_int00-;
    c_int00--->mian（）;
```

为此在`DSP2833x_CodeStartBranch.asm`中将所有的`c_int00`替换为`copy_sections`。如下**2.1**步骤

程序运行后从**FLASH**启动，会调用**code_start**关闭看门狗后通过调用**c_int00**，通过**c_int00**调用**main（）**函数，所以程序从**FLASH**拷贝到**RAM**需要在**c_int00**之前完成。所以将做以下修改：

### 2.1 将程序中的_c_int00修改为 copy_sections
### 2.2 将.text 修改为.sect "wddisable"

**修改解释：**

```bash
WD_DISABLE .set 1 ;set to 1 to disable WD, else set to 0

.ref _c_int00   #（这里不需要使用_c_int00二是需要调用copy_sections函数，故改为.ref copy_sections）

    .global code_start

    .sect "codestart"

code_start:

    .if WD_DISABLE == 1

        LB wd_disable       

    .else

        LB _c_int00 #(这里应该跳转到copy_sections，故应改为LB copy_sections)

.endif

    .if WD_DISABLE == 1

    .text #（.text段将会加载到RAM中运行，这里看门狗程序是需要在拷贝前，也就是FLASH中运行，所以从新定义一个wddisable段，之后将会添加到.CMD文件，所以修改为.sect "wddisable" 之所以这么定义，是因为.text段是需要拷贝到RAM执行的，而在程序拷贝之前首先需要关闭看门狗，从而避免出现错误。如果仍然将程序放在.text段中，则看门狗禁止程序无法执行，会出现错误，因此在FLASH中定义一个新的段，从而让看门狗关闭的程序在FLASH中提前执行）

wd_disable:

    SETC OBJMODE        ;Set OBJMODE for 28x object code

    EALLOW              ;Enable EALLOW protected register access

    MOVZ DP, #7029h>>6  ;Set data page for WDCR register

    MOV @7029h, #0068h  ;Set WDDIS bit in WDCR to disable WD

    EDIS                ;Disable EALLOW protected register access

    LB _c_int00  #（这里关闭看门狗后需要调用拷贝函数在跳转到_C_INT00所以改为LB copy_sections）       

    .endif

.end

```

## 3.修改DSP2833x_SysCtrl.c文件
**注释掉一些程序**

```bash
 #pragma CODE_SECTION(InitFlash, "ramfuncs");

```

程序定义了**ramfuncs**段，当在**FLASH**中运行时，由于速度问题，部分函数必须在**RAM**中运行，这段程序的作用就是声明**IinitFlash** 函数属于**ramfunc**段，需在**RAM**中运行，我们把所有程序加载到**RAM**中运行，故不再需要，注释掉后默认在`.text`段

## 4. 修改DSP2833x_usDelay.asm文件

```bash
.def _DSP28x_usDelay

.sect "ramfuncs"

```bash

改为

```bash
.def _DSP28x_usDelay

.text

```

原理同上

## 5. CMD文件修改

### 5.1 ：删除相关代码
删除类似以下ramfuncs

```bash
ramfuncs            : LOAD = FLASHD, 
                    RUN = RAML0, 
                    LOAD_START(_RamfuncsLoadStart),
                    LOAD_END(_RamfuncsLoadEnd),
                    RUN_START(_RamfuncsRunStart),
                    PAGE = 0
```

**Ramfuncs**段是之前在**FLASH**中运行时需要把部分程序搬移到**RAM**中定义的段，  `_DSP28x_usDelay`函数就定义在该段，现在要把所有程序都搬到**RAM**中，故不再需要。

### 5.2：添加代码
（通常添加在codestart  : > BEGIN,  PAGE = 0）之后

```bash
wddisable    : > FLASHA,    PAGE = 0
copysections        : > FLASHA,    PAGE = 0

```
**Wddisable**与**copysections**是新添加的段，**Wddisable**是之前将`DSP2833x_CodeStartBranch.asm`中关闭看门狗的代码放在了**Wddisable**段，该段需在**FLASH**中运行。**Copysections**是拷贝函数所在的断，需在**Flash**中完成拷贝。

### 5.3：修改.stack 栈 .ebss全局数据、静态数据 .esysmem堆
可以修改他们的存储大小与位置，但必须在低64K地址中即（M0，M1，L4-L7）(L1 -L3受保护的，放代码段的）。例如：

```bash
.stack              : > RAMM1       PAGE = 1
.ebss               : > RAML4       PAGE = 1
.esysmem            : > RAML4       PAGE = 1

```
### 5. 4：修改代码存储位置与运行位置

将`.cinit              : > FLASHA      PAGE = 0`改为

```bash

.cinit : LOAD = FLASHA, PAGE = 0       

                 RUN = RAML0,       PAGE = 0   

                 LOAD_START(_cinit_loadstart),

                 RUN_START(_cinit_runstart),

                  SIZE(_cinit_size)

```
将程序中变量初值和常量加载到**FLASH**运行在**RAM**中

将`.econst             : > FLASHA      PAGE = 0`改为

```bash

.econst :   LOAD = FLASHA,   PAGE = 0       

                 RUN = RAML0,       PAGE = 0        

                 LOAD_START(_econst_loadstart),

                RUN_START(_econst_runstart),

                 SIZE(_econst_size)

```

将`.pinit              : > FLASHA,     PAGE = 0 `改为

```bash

.pinit :   LOAD = FLASHA,   PAGE = 0         

                 RUN = RAML0, PAGE = 0        

                 LOAD_START(_pinit_loadstart),

                 RUN_START(_pinit_runstart),

                 SIZE(_pinit_size)

```
将`.switch             : > FLASHA      PAGE = 0  `修改为

```bash
.switch :   LOAD = FLASHA,   PAGE = 0        

                 RUN = RAML0,       PAGE = 0       

                 LOAD_START(_switch_loadstart),

                 RUN_START(_switch_runstart),

                 SIZE(_switch_size)   

```
增加`.const`段的定义，或者将拷贝函数中`.const`的拷贝注释掉

```bash
.const :   LOAD = FLASHA,   PAGE = 0        

                 RUN = RAML0, PAGE = 0        

                 LOAD_START(_const_loadstart),

                 RUN_START(_const_runstart),

                 SIZE(_const_size)
```

### 5. 5修改.text段

为了保证足够代码空间，修改MEMORY PAGE 0中

```bash
RAML1       : origin = 0x009000, length = 0x001000        

 RAML2       : origin = 0x00A000, length = 0x001000     

 RAML3       : origin = 0x00B000, length = 0x001000  
```
改为

```bash
  RAM_L1L2L3       : origin = 0x009000, length = 0x003000  
```
然后将SECTIONS中

`.text               : > FLASHA      PAGE = 0`改为

```bash
.text :   LOAD = FLASHA,  PAGE = 0        

                 RUN = RAM_L1L2L3, PAGE = 0        

                 LOAD_START(_text_loadstart),

                 RUN_START(_text_runstart),

                 SIZE(_text_size)
```
## 6.rebuild  project 和烧写
方法和debug一样



*备注：本文一共需要需要的文件均已经修改和添加在附件文件下【flash-sram】中*

【百度云下载链接】：[【flash-sram】](https://pan.baidu.com/s/10LwtP3d7IoOeleQKL-Z0pw)
【提取码】： 3xi2

