> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/weixin_45566765/article/details/119082331)

uboot 就是将 start.o 和大量的 built-in.o 链接在一起。

> built-in.o 好像是把所有子目录下的. o 文件进行链接到一起。

链接脚本为 u-boot.lds ，uboot 链接首地址为 0x87800000，裸机的时候也是 - Ttest 来执行链接首地址

![](https://img-blog.csdnimg.cn/6927a3f92a484e6ca270c2da67b05472.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
查找一下这个链接的地址

```
grep -nR "87800000"
```

![](https://img-blog.csdnimg.cn/1a2fae1ba5ab4a7cbe87726a9979c1cf.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
在 mx6_common.h 文件中设置

通过 uboot-lds 可以看到入口地址为_start  
![](https://img-blog.csdnimg.cn/c9fb3e90cb1e482fa1227c1ac3acbf9f.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)

1) 设置 CPU 为管理模式  
2) 关看门狗  
3) 关中断  
4) 设置时钟频率  
5) 关 mmu, 初始化各个 bank

6) 进入 board_init_f() 函数

> 初始化 DDR, 定时器，初始化波特率串口，打印前面暂存在缓冲区的数据 此时 sp 和 gd 是存放在 DDR 上了。而不是内部的 RAM。
> 
> uboot 会将自己重定位到 DRAM 最后面的地址区域，也就是将自己拷贝到 DRAM 最后面的内存区域中。这么做的目的是给 Linux 腾出空间，防止 Linux kernel 覆盖掉 uboot，将 DRAM 前面的区域完整的空出来。在拷贝之前肯定要给 uboot 各部分分配好内存位置和大小，比如 gd 应该存放到哪个位置，malloc 内存池应该存放到哪个位置等等。这些信息都保存在 gd 的成员变量中，因此要对 gd 的这些成员变量做初始化。最终形成 一个完整的内存 “分配图”，在后面重定位 uboot 的时候就会用到这个内存 “分配图”。

(7) 重定位

> relocate_code 代码重定位函数，负责将 uboot 拷贝到新的地方，完成代码拷贝。

8) 清 bss  
9) 跳转到 board_init_r() 函数, 启动流程结束。

> 前面 board_init_f 函数里面会调用一系列的函数来初始化一些外设和 gd 的成员变量。但是  
> board_init_f 并没有初始化所有的外设，还需要做一些后续工作，这些后续工作就是由函数  
> board_init_r(这里的外设更多包括 EMMC，NANDFLASH) 来完成的, 存放于 common/board_r.c。

最后运行的是 run_main_loop，主循环，处理命令。

**从这里开始就是：打断倒计时执行 uboot 命令要么进入内核的操作。**

> uboot 启动以后会进入 3 秒倒计时，如果在 3 秒倒计时结束之前按下按下回车键，那么就会进入 uboot 的命令模式，如果倒计时结束以后都没有按下回车键，那么就会自动启动 Linux 内 核，这个功能就是由 run_main_loop 函数来完成的。

A：如果自然结束：run_command_list 去执行 bootcmd 的命令，保存着默认的启动命令，因此 linux 启动  
B：循环处理输入的命令

> **bootcmd 和 bootargs 的区别**
> 
> bootcmd： 这个参数包含了一些命令，这些命令将在 u-boot 进入主循环后执行 示例：  
> bootcmd=boot_logo;nand read 10000003c0000 300000;bootm 1000000  
> 意思是启动 u-boot 后，执行 boot_logo 显示 logo 信息，然后从 nand flash 中读内核映像到内存，然后启动内核。
> 
> bootargs 这个参数设置要传递给内核的信息，主要用来告诉内核分区信息和根文件系统所在的分区。 示例：  
> root=/dev/mtdblock5 rootfstype=jffs2console=ttyS0,115200 mem=35M  
> mtdparts=nand.0:3840k(u-boot),4096k(kernel),123136k(filesystem)

[移植 uboot - 分析 uboot 启动流程 (详解)](https://www.cnblogs.com/lifexy/p/8136378.html)

对于重定位：  
一般片内 ROM 会根据程序的头部会有一个链接地址，然后程序  
通过链接脚本 把源地址 目的地址，长度得到在进行重定位，通过链接脚本得到 (至少得到代码段有多长)  
![](https://img-blog.csdnimg.cn/b821cd998b88409aa76e1115717717eb.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
什么是位置无关码？什么是绝对跳转指令以及相对跳转指令  
![](https://img-blog.csdnimg.cn/91039a8ef060437f89171c1df3d15279.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
通过 lds 的链接脚本确定链接地址

```
SECTIONS {
    . = 0x80100000;

    . = ALIGN(4);
    .text      :
    {
      *(.text)
    }

    . = ALIGN(4);
    .rodata : { *(.rodata) }

    . = ALIGN(4);

    data_load_addr = .;
    .data 0x900000 : AT(data_load_addr)
    {
      data_start = . ;
      *(.data) 
      data_end = . ;
    }

    . = ALIGN(4);
    __bss_start = .;
    .bss : { *(.bss) *(.COMMON) }
    __bss_end = .;
}
```

```
.text
.global  _start

_start: 

	/* 设置栈 */
	ldr  sp,=0x80200000

	/* 重定位text, rodata, data段 */
	bl copy_data

	/* 清除bss段 */
	bl clean_bss

	/* 跳转到主函数 */
	// bl main		/* 相对跳转，程序仍在DDR3内存中执行 */
	ldr pc, =main 	/* 绝对跳转，程序在片内RAM中执行 */

halt:
	b  halt
```

实现的从链接脚本中读取的地址

```
/**********************************************************************
 * 函数名称： copy_data
 * 功能描述： 将整个程序(.text, .rodata, .data)从DDR3重定位到片内RAM
 * 输入参数： 无
 * 输出参数： 无
 * 返 回 值： 无
 * 修改日期        版本号        修改人        修改内容
 * -------------------------------------------------
 * 2020/02/20	    V1.0         阿和            创建
 ***********************************************************************/
void copy_data (void)
{
	/* 从链接脚本中获得参数 _start, __bss_start, */
	extern int _load_addr, _start, __bss_start;

	volatile unsigned int *dest = (volatile unsigned int *)&_start;			//_start = 0x900000
	volatile unsigned int *end = (volatile unsigned int *)&__bss_start;		//__bss_start = 0x9xxxxx
	volatile unsigned int *src = (volatile unsigned int *)&_load_addr;		//_load_addr = 0x80100000

	/* 重定位数据 */	
	while (dest < end)
	{
		*dest++ = *src++;
	}
}
		  			 		  						  					  				 	   		  	  	 	  
/**********************************************************************
 * 函数名称： clean_bss
 * 功能描述： 清除.bss段
 * 输入参数： start, end
 * 输出参数： 无
 * 返 回 值： 无
 * 修改日期        版本号        修改人        修改内容
 * -------------------------------------------------
 * 2020/02/20	    V1.0         阿和            创建
 ***********************************************************************/
void clean_bss(void)
{
	/* 从lds文件中获得 __bss_start, __bss_end */
	extern int __bss_end, __bss_start;

	volatile unsigned int *start = (volatile unsigned int *)&__bss_start;
	volatile unsigned int *end = (volatile unsigned int *)&__bss_end;

	while (start <= end)
	{
		*start++ = 0;
	}
}
```

1、uboot 分析之编译体验
===============

*   1、解压缩
    
*   2、打补丁  
    通过 patch 进行打补丁  
    ![](https://img-blog.csdnimg.cn/e3e3b3018c004b169f236d70acc8c638.png)  
    -p1 表示忽略第一个目录  
    ![](https://img-blog.csdnimg.cn/62339622b85640aab512ac74c41c040e.png)
    
*   3、配置  
    make 100ask24x0_config
    
*   4、编译  
    make
    

> uboot 的终极目的就是启动内核：  
> 1、从 Flash 读出内核，放到 SDRAM 内存中  
> 2、启动内核  
> 因此 uboot 要实现的功能  
> 1、能读 Flash + 为了开发方面提供写 Flash 功能 (通过网络 [网卡] 或者串口 [USB] 烧写内核)  
> 2、初始化 SDRAM(在此之前需要关看门狗、初始化时钟、初始化串口)，能够写 SDRAM; 从 Flash 中读取内核到 SDRAM 中  
> 3、启动内核

分析源码的架构最好的方法就是通过 Makefile 文件去分析。

二、uboot 分析之 Makefile 结构分析
=========================

我们需要

*   3、配置  
    make 100ask24x0_config
*   4、编译  
    make

执行 make 100ask24x0_config 命令的时候，通过关键字找到，相当于执行下面这一行。  
![](https://img-blog.csdnimg.cn/86582bbd90ab4d708a3131a106709823.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
找去找一下 MKCONFIG，在 makefile 中有定义  
![](https://img-blog.csdnimg.cn/499d192d7c1a4ef79bd2e9329797d597.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
表示在源码树下的 mkconfig  
因此，我们配置的时候，实际上执行了这样的一个脚本命令。($@表示替换，结果为：100ask24x0)  
![](https://img-blog.csdnimg.cn/f472cf6deb614b58b0075da3339fa1c8.png)  
![](https://img-blog.csdnimg.cn/638062a34143496e9da524955de1bf38.png)

$# 表示参数的个数  
$1 表示第一个参数

![](https://img-blog.csdnimg.cn/9637f0776f2345c0af348df44b8011fc.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)

![](https://img-blog.csdnimg.cn/a09a838d9b284a8eac29d23e95a1d1a0.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
‘>’ 表示新建一个文件  
‘>>’ 表示追加  
可以看到确实有 config.mk 有导入的内容  
![](https://img-blog.csdnimg.cn/ce6e594a5734407883f78c05bbd6ff27.png)  
同时 config.h 也有  
![](https://img-blog.csdnimg.cn/cacc2ab8d73146b6a767195b0090bc75.png)

![](https://img-blog.csdnimg.cn/bee9237451504ab5b280be80252a286e.png)

重点部分：
-----

make 的时候如果不指定目标，就回去执行第一个目标 all  
![](https://img-blog.csdnimg.cn/5016fba1d11d40309978bd4b352ddc65.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)

通过 make 编译后看完整的过程 ，  
![](https://img-blog.csdnimg.cn/6c28690ad0bf4f4b98c031c341487496.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)

分析可以得到：arm-linux-ld 链接命令：  
1、链接脚本 u-boot.lds 是在最前面的  
2、其次是 start.o  
3、再是各种 lib 库

链接脚本 0x00000000 会加上 0x33F80000，会从这个地址开始排放代码段等等![](https://img-blog.csdnimg.cn/3c2d0f5712cc48d0b736e4e90a574f92.png)  
![](https://img-blog.csdnimg.cn/1caa9cb7fefd41ada3f8cd4def5ff670.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/7b80dcba0efa4ec2ba5aa97dc7910a99.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
可以看到整个内存段的排放。  
1、第一个文件 cpu/arm920t/start.S。  
2、链接地址

![](https://img-blog.csdnimg.cn/43047971dc3b4ae58cdc15e258096a59.png)  
board/100ask24x0/config.mk 我们猜测是不是这个 TEXT_BASE？  
我们看一下之前 Makefile 中链接脚本中定义的 LDFLAGS

![](https://img-blog.csdnimg.cn/b542636f45a14c84b3a6813477cf3f70.png)  
在 uboot 根目录下的 config.mk 可以看到  
![](https://img-blog.csdnimg.cn/140b9cf6e6c74b01bd5b0afb787a365e.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
其中 TEXT_BASE 是在 board/100ask24x0/config.mk 下定义的  
![](https://img-blog.csdnimg.cn/d8208e58aa5b40819a5f997a643d9be0.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
对内存的上面空出 512K 的大小，如果你的 uboot 大小超过了 512K 你也可以修改它的大小，之前前面也提到过 uboot 一般放在内存的最上方，当 linux 内核加载的时候后面就会覆盖 uboot 的代码。

3、uboot 分析之源码第一阶段
=================

> 在裸机的时候启动步骤  
> 1、初始化
> 
> *   关看门口
> *   初始时钟
> *   初始化 SDRAM
> 
> 2、把程序从 NAND->SDRAM  
> 3、设置栈 (也就是把 sp 指向某块内存)

因此 uboot 在上面基础上再调用 c 函数从 Flash 读出内核，再启动内核。

具体步骤如下 (第一阶段的全部是硬件部分的初始化)
-------------------------

*   进入 reset
    
*   进入 SVC32 系统模式
    
*   关看门狗
    
*   关中断
    
*   进入初始化  
    ![](https://img-blog.csdnimg.cn/7b38aab84ac34e5d868459230fe4a326.png)  
    关 flash，清 cache  
    ![](https://img-blog.csdnimg.cn/b1a6b21b618a42b2a3b34395e1bd85e9.png)  
    关 MMU，初始化存储控制器，初始化后内存控制器才可以使用
    
*   再设置栈  
    ![](https://img-blog.csdnimg.cn/985a16c54edd49edb7daa040ecfb626e.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
    通过框起来的代码后，内存布局就是这样，最上面是 uboot 的代码，底下存放的自己实现的 malloc 的区域以及 IRQ、FIQ 和 SP 指针指向的地方，各有各的用处。  
    ![](https://img-blog.csdnimg.cn/67d1f7f8b49d4f46913e36ab4d9f3f8a.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
    栈设置好了后才能去调用 C 函数
    
*   接着 block_init 初始化时钟
    
*   重定位：把代码从 Flash 拷贝到 SDRAM 中，读到 SDRAM 哪里去呢，读到链接地址中去  
    ![](https://img-blog.csdnimg.cn/2398384c4316470fb50407dc54ad422f.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)
    
*   清 BSS 段  
    ![](https://img-blog.csdnimg.cn/1d0bf0ca93234efbb54815b60137388b.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)
    
*   调用 start_armboot(C 函数)，从这个函数开始就是第二阶段了 (第二阶段有一些拓展的开发功能例如：烧写 Flash、网卡、USB、串口、启动内核等等)
    

4、uboot 分析之源码第二阶段
=================

> 我们以  
> 1、从 Flash 读出内核  
> 2、启动  
> 为目的来看。

*   进入 start_armboot  
    ![](https://img-blog.csdnimg.cn/64a1c903fed7407eb290a9b2541f1715.png)  
    分配一个 gd 结构体存放在在 128k 的 CFG_GBL_DATA_SIZE 内存中  
    ![](https://img-blog.csdnimg.cn/60c38ec2d6034eddb677fe33404c0a26.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)
*   通过函数指针进行一系列的初始化  
    ![](https://img-blog.csdnimg.cn/2489506509eb42098c54725364157fee.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
    cpu 初始化，单板初始化。。。  
    ![](https://img-blog.csdnimg.cn/71aa7c49b61c426d8791fa6214f6a11d.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)
*   在单板初始化中会初始化一些管脚，同时还有机器 ID 以及启动参数的设置、nor nand 的初始化

> ![](https://img-blog.csdnimg.cn/d7d2c04ec1f348a2b8da283827a545b2.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
> 下一节中我们讲解内核的时候会关注这两个参数为什么会有。
> 
> 再是 NORflash 初始化：NANDflash 可以直接读写 (比如我们往 0 地址写 1234 如果读出来是 1234 那就是 NAND，读出来还是 0 的话就是 NORflash)，而 NorFlash 的写的时候我们就要配置它，我们因为会有一些拓展的功能就是重新烧写 Flash 所以我需要对 NorFlash 读写进行初始化  
> ![](https://img-blog.csdnimg.cn/70f54d87584d4feb919ee3ceb1354673.png)  
> 再是 nand flash 初始化，还有 malloc 初始化  
> ![](https://img-blog.csdnimg.cn/dd13a510fdbf4201afca0ea41fd6b635.png)  
> 现在 CPU 具备可以读可以写的能力了。  
> 再是环境变量的初始化：  
> ![](https://img-blog.csdnimg.cn/67a7d101057d422daf08ec3214820aa2.png)  
> 以下内容就是环境变量  
> ![](https://img-blog.csdnimg.cn/1e9919bc8db041c8aafc1f507266a7d6.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
> 环境变量会先去 Flash 上去找，如果没有设置的话就会去默认的环境变量去看。  
> 网卡、设备、USB 的初始化

*   就这就到了一个 main_loop 死循环的地方  
    ![](https://img-blog.csdnimg.cn/9bfd5b205e734823b0ee32c3a27001d0.png)

以上的步骤我们知道通过 start_armboot：  
执行了 flash_init  
nand_init 一些硬件设备的初始化  
到达 main_loop 的循环中。

在 main_loop 可看到一些环境变量比如：bootdelay  
![](https://img-blog.csdnimg.cn/6fb34b605ac142678b2514b9d606352f.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)

通过获取 bootcmd 环境变量得到启动命令  
![](https://img-blog.csdnimg.cn/bc8674ce520d4fdab0e6af1aeca68e72.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)

后面一段代码是，如果在倒数计时在到达 0 之前，没有输入空格键。就去打印 booting linux 并且执行 run_command，bootcmd 其实就是两个参数，回到我们这一节最开始的地方。  
![](https://img-blog.csdnimg.cn/41608016b80e4a0096e2f7d84e2f77d3.png)

1、我们想要做的就是从 flash 读出内核也就是对应 bootcmd 的第一个指令

```
nand read.jffs2 0x30007FC0 kernel；
```

读到 0x30007FC0(为什么是这个值我们后面再进行分析) 从 kernel 分区进行读；  
2、怎么启动：通过 bootm 0x30007FC0  
前面已经从 NANDflash 把内核读到内存中来，启动内核。

**因此内核的启动依赖于 bootcmd 命令**

如果我们按了空格，就会跳到下面的代码中：

![](https://img-blog.csdnimg.cn/1db01f9a597746129e04dfa3c44b27f8.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)

因此可以分为两类：  
1、启动内核  
s = getenv(“bootcmd”)  
run_command(s,…)  
2、u-boot 界面  
readline(读入串口的数据)  
run_command

5、uboot 启动内核
============

1、我们想要做的就是从 flash 读出内核也就是对应 bootcmd 的第一个指令

```
nand read.jffs2 0x30007FC0 kernel；
```

读到 0x30007FC0(为什么是这个值我们后面再进行分析) 从 kernel 分区进行读；

*   kernel 分区是什么？  
    因为用的是 Flash 没有分区，我们只能通过软件写死分区 ，固定大小区间  
    ![](https://img-blog.csdnimg.cn/b4807037c9aa40588918bdc4f3073cc8.png)  
    所以从 NANDFlash 中读，那从哪里读呢就是从 kernel 中读，然后读到 SDRAM 中  
    ![](https://img-blog.csdnimg.cn/557b3c51785d4f51a9da8764d6e9c13a.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
    其中 kernel 就是代表其实地址，当然我们用
    
    ```
    nand read.jffs2 0x30007FC0 0x00060000 0x00200000；
    ```
    
    效果是一样的，这个就代表从从起始地址 0x00060000 ，长度为 0x00200000，和图中 kenel 分区的偏移 + 大小是一致的。  
    ![](https://img-blog.csdnimg.cn/df14802fc42f4c0ab86edcfc762773e2.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
    用 jffs2 是因为这个文件格式不需要页对齐。

2、怎么启动：通过 bootm 0x30007FC0  
前面已经从 NANDflash 把内核读到内存中来，启动内核 -> 调用 do_bootm。

U-boot 在 Flash 上存的内核叫：uImage  
uImage 就是一个头部加上一个真正的内核。  
这就是头部的结构体  
![](https://img-blog.csdnimg.cn/d00cd201375242f5b379716aa79307c9.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
ih_load 表示加载地址，表示内核要放在哪里  
ih_ep 表示入口地址，运行内核的时候直接跳到这个地址就可以了

你可以随便放内存里面，只要不破坏 uboot 在内存最顶部存放的信息，因为 uImage 的头部已经包含了加载地址以及入口地址的信息

do_bootm 函数做的事情
---------------

读出头部  
![](https://img-blog.csdnimg.cn/b1e8ae250abf447c88c9702c8d1c5b3e.png)  
把数据拷贝到加载地址中去  
![](https://img-blog.csdnimg.cn/2e62074355a841878dc86fc7e595994d.png)

我们真正的加载地址是 0x30008000 那为什么 bootm 的地址是 0x30007fc0  
两个相减后得到的结果是 64，所以大小是 64 字节，刚好是 64 字节的头部。

启动就是调用 do_bootm_linux  
![](https://img-blog.csdnimg.cn/3fa7459213e34308a464e1593d0f8983.png)  
那 do_bootm_linux 也需要做一些事情：  
(1) uboot 告诉内核一些参数 -> 设置启动参数

> 在某个地址 0x30000100，按某种格式保存数据，这个格式成为 TAG  
> ![](https://img-blog.csdnimg.cn/2e635affdb384a50a7fc8b7b2c4639b6.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)![](https://img-blog.csdnimg.cn/1ec2f13bd8864654b0b07a495c9c99f5.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
> ![](https://img-blog.csdnimg.cn/ec0166e89f2543dcadee58c783d42596.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
> setup_start_tag  
> setup_memory_tag  
> setup_commandline_tag  
> setup_end_tag

第一个 setup_start_tag 函数：  
![](https://img-blog.csdnimg.cn/3f1cb480e4464f169d6ce889f01a0912.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)![](https://img-blog.csdnimg.cn/03b5bc784e9846e39db94f80fff9a424.png)  
第二个 setup_memory_tag：![](https://img-blog.csdnimg.cn/ab62e371be0746e38636b29285802f85.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/931e6d35ccc9478082b2545d58f3a91e.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
此时的内存布局，接下来就是 params = tag_next(params)；

到了第三个 setup_commandline_tag:  
![](https://img-blog.csdnimg.cn/29ae86d89b774d1b97eeb28629da8fe4.png)  
来自于 commandline  
![](https://img-blog.csdnimg.cn/46abed4609344a0a9b9ce817209e03a9.png)  
![](https://img-blog.csdnimg.cn/f59ea19dc0b446dfb8dc56a96cd643bc.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
把 bootargs 的参数传入给内核，其中：  
root = 表示根文件系统他位于 flash 的第四个分区  
init = 表示第一个运行的应用程序  
console 表示内核的打印信息，从串口 0 打印出来

第四个 setup_end_tag  
![](https://img-blog.csdnimg.cn/c019744273e8475f96df7e68f5df0edc.png)  
设置 0，0

至此整个 TAG 就填写完了，保存好了后，内核就回到这个地址来读取这些参数  
![](https://img-blog.csdnimg.cn/15ca60bdec7940cdbb123f130cf8c8ba.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/5ed85e8002c34ca4ac85f3a5d9712160.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)  
启动的时候带了 3 个参数，其中后面两个参数在第 4 节中，我们就提到过，这里刚好对应上了，可以看到启动参数的启动地址也就是 0x30000100。同时还要和内核比对机器 ID 是否匹配。  
![](https://img-blog.csdnimg.cn/d7d2c04ec1f348a2b8da283827a545b2.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)

(2) 跳到入口地址启动内核，调用 theKernel

![](https://img-blog.csdnimg.cn/5ffd6166b3c54ebaa531082b72be6b85.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTU2Njc2NQ==,size_16,color_FFFFFF,t_70)