> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/u014650722/article/details/79076352)

**1、设备树的概念**

        在内核源码中，存在大量对板级细节信息描述的代码。这些代码充斥在 / arch/arm/plat-xxx 和 / arch/arm/mach-xxx 目录，对内核而言这些 platform 设备、resource、i2c_board_info、spi_board_info 以及各种硬件的 platform_data 绝大多数纯属垃圾冗余代码。为了解决这一问题，ARM 内核版本 3.x 之后引入了原先在 Power PC 等其他体系架构已经使用的 Flattened Device Tree。  

        开源文档中对设备树的描述是，一种描述硬件资源的数据结构，它通过 bootloader 将硬件资源传给内核，使得内核和硬件资源描述相对独立。

         Device Tree 可以描述的信息包括 CPU 的数量和类别、内存基地址和大小、总线和桥、外设连接、中断控制器和中断使用情况、GPIO 控制器和 GPIO 使用情况、Clock 控制器和 Clock 使用情况。

         另外，设备树对于可热插拔的设备不进行具体描述，它只描述用于控制该热插拔设备的控制器。

         设备树的主要优势：对于同一 SOC 的不同主板，只需更换设备树文件. dtb 即可实现不同主板的无差异支持，而无需更换内核文件。

_（注：要使得 3.x 之后的内核支持使用设备树，除了内核编译时需要打开相对应的选项外，bootloader 也需要支持将设备树的数据结构传给内核。）_

**2、设备树的组成和使用**

        设备树包含 DTC（device tree compiler），DTS（device tree source 和 DTB（device tree blob）。其对应关系如下图所示：

                                                               ![](https://leanote.com/api/file/getImage?fileId=57f737e8ab644106a000aced)

2.1 DTS 和 DTSI(源文件)
-------------------

        .dts 文件是一种 ASCII 文本对 Device Tree 的描述，放置在内核的 / arch/arm/boot/dts 目录。一般而言，一个. dts 文件对应一个 ARM 的 machine。

         由于一个 SOC 可能有多个不同的电路板（  .dts 文件为板级定义， .dtsi 文件为 SoC 级定义），而每个电路板拥有一个 .dts。这些 dts 势必会存在许多共同部分，为了减少代码的冗余，设备树将这些共同部分提炼保存在. dtsi 文件中，供不同的 dts 共同使用。.dtsi 的使用方法，类似于 C 语言的头文件，在 dts 文件中需要进行 include .dtsi 文件。当然，dtsi 本身也支持 include 另一个 dtsi 文件。

2.2 DTC (编译工具)
--------------

        DTC 为编译工具，dtc 编译器可以把 dts 文件编译成为 dtb，也可把 dtb 编译成为 dts 文件。在 3.x 内核版本中，DTC 的源码位于内核的 scripts/dtc 目录，内核选中 CONFIG_OF，编译内核的时候，主机可执行程序 DTC 就会被编译出来。 即 scripts/dtc/Makefile 中

1.  hostprogs-y := dtc
2.  always := $(hostprogs-y) ​

        在内核的 arch/arm/boot/dts/Makefile 中，若选中某种 SOC，则与其对应相关的所有 dtb 文件都将编译出来。在 linux 下，make dtbs 可单独编译 dtb。以下截取了 TEGRA 平台的一部分。

1.  ifeq ($(CONFIG_OF),y)
2.  dtb-$(CONFIG_ARCH_TEGRA) += tegra20-harmony.dtb \
3.  tegra30-beaver.dtb \
4.  tegra114-dalmore.dtb \
5.  tegra124-ardbeg.dtb ​

        在 2.6.x 版本内核中，只在 powerpc 架构下使用了设备树，DTC 的源码位于内核的 arch/powerpc/boot/dtc-src 目录，编译内核后，可将 DTC 编译出来，DTC 编译工具位于 arch/powerpc/boot 目录下。

2.3 DTB (二进制文件)
---------------

       DTC 编译. dts 生成的二进制文件（.dtb），bootloader 在引导内核时，会预先读取. dtb 到内存，进而由内核解析。

        在 2.6.x 版本内核中，在 powerpc 架构下，dtb 文件可以单独进行编译，编译命令格式如下：

dtc [-I input-format] [-O output-format][-o output-filename] [-V output_version] input_filename

参数说明

input-format：

- “dtb”: “blob” format

- “dts”: “source” format.

- “fs” format.

output-format：

- “dtb”: “blob” format

- “dts”: “source” format

- “asm”: assembly language file

output_version：

定义”blob” 的版本，在 dtb 文件的字段中有表示，支持 1　2　3 和 16, 默认是 3, 在 16 版本上有许多特性改变

(1)  Dts 编译生成 dtb

./dtc -I dts -O dtb -o B_dtb.dtb A_dts.dts

把 A_dts.dts 编译生成 B_dtb.dtb

(2)  Dtb 编译生成 dts

./dtc -I dtb -O dts -o A_dts.dts A_dtb.dtb

把 A_dtb.dtb 反编译生成为 A_dts.dts

        在 linux 3.x 内核中，可以使用 make 的方式进行编译。

2.4 Bootloader(boottloader 支持)
------------------------------

    Bootloader 需要将设备树在内存中的地址传给内核。在 ARM 中通过 bootm 或 bootz 命令来进行传递。    

    bootm [kernel_addr] [initrd_address] [dtb_address]，其中 kernel_addr 为内核镜像的地址，initrd 为 initrd 的地址，dtb_address 为 dtb 所在的地址。若 initrd_address 为空，则用 “-” 来代替。

**3、linux 内核对硬件的描述方式**

       在以前的内核版本中：  
1）内核包含了对硬件的全部描述；  
2）bootloader 会加载一个二进制的内核镜像，并执行它，比如 uImage 或者 zImage；  
3）bootloader 会提供一些额外的信息，成为 ATAGS，它的地址会通过 r2 寄存器传给内核；  
    ATAGS 包含了内存大小和地址，kernel command line 等等；  
4）bootloader 会告诉内核加载哪一款 board，通过 r1 寄存器存放的 machine type integer；  
5）U-Boot 的内核启动命令：bootm <kernel img addr>  
6）Barebox 变量：bootm.image (?)  
[![](http://upload.semidata.info/www.eefocus.com/blog/media/201410/331381.jpg)](http://upload.semidata.info/www.eefocus.com/blog/media/201410/331381.jpg)  
  
现今的内核版本使用了 Device Tree：  
1）内核不再包含对硬件的描述，它以二进制的形式单独存储在另外的位置：the device tree blob  
2）bootloader 需要加载两个二进制文件：内核镜像和 DTB  
    内核镜像仍然是 uImage 或者 zImage；  
    DTB 文件在 arch/arm/boot/dts 中，每一个 board 对应一个 dts 文件；  
3）bootloader 通过 r2 寄存器来传递 DTB 地址，通过修改 DTB 可以修改内存信息，kernel command line，以及潜在的其它信息；  
4）不再有 machine type；  
5）U-Boot 的内核启动命令：bootm <kernel img addr> - <dtb addr>  
6）Barebox 变量：bootm.image,bootm.oftree  
[![](http://upload.semidata.info/www.eefocus.com/blog/media/201410/331382.jpg)](http://upload.semidata.info/www.eefocus.com/blog/media/201410/331382.jpg)  
  
        有些 bootloader 不支持 Device Tree，或者有些专门给特定设备写的版本太老了，也不包含。为了解决这个问题，CONFIG_ARM_APPENDED_DTB 被引进。   
    它告诉内核，在紧跟着内核的地址里查找 DTB 文件；  
    由于没有 built-in Makefile rule 来产生这样的内核，因此需要手动操作：  
        cat arch/arm/boot/zImage arch/arm/boot/dts/myboard.dtb > my-zImage  
        mkimage ... -d my-zImage my-uImage  
    (cat 这个命令，还能够直接合并两个 mp3 文件哦！so easy！)  
另外，CONFIG_ARM_ATAG_DTB_COMPAT 选项告诉内核去 bootloader 里面读取 ATAGS，并使用它们升级 DT。  

**4、DTB 加载及解析过程**

![](https://leanote.com/api/file/getImage?fileId=57f737e8ab644106a000acf2)

    先从 uboot 里的 do_bootm 出发，根据之前描述，DTB 在内存中的地址通过 bootm 命令进行传递。在 bootm 中，它会根据所传进来的 DTB 地址，对 DTB 所在内存做一系列操作，为内核解析 DTB 提供保证。上图为对应的函数调用关系图。

    在 do_bootm 中，主要调用函数为 do_bootm_states，第四个参数为 bootm 所要处理的阶段和状态。 

    在 do_bootm_states 中，bootm_start 会对 lmb 进行初始化操作，lmb 所管理的物理内存块有三种方式获取。起始地址，优先级从上往下：

1.   环境变量 “bootm_low”
2.   宏 CONFIG_SYS_SDRAM_BASE（在 tegra124 中为 0x80000000）
3.   gd->bd->bi_dram[0].start

大小：

1.   环境变量 “bootm_size”
2.   gd->bd->bi_dram[0].size

    经过初始化之后，这块内存就归 lmb 所管辖。接着，调用 bootm_find_os 进行 kernel 镜像的相关操作，这里不具体阐述。

    还记得之前讲过 bootm 的三个参数么，第一个参数内核地址已经被 bootm_find_os 处理，而接下来的两个参数会在 bootm_find_other 中执行操作。

    首先，bootm_find_other 根据第二个参数找到 ramdisk 的地址，得到 ramdisk 的镜像；然后根据第三个参数得到 DTB 镜像，同检查 kernel 和 ramdisk 镜像一样，检查 DTB 镜像也会进行一系列的校验工作，如果校验错误，将无法正常启动内核。另外，uboot 在确认 DTB 镜像无误之后，会将该地址保存在环境变量 “fdtaddr” 中。

    接着，uboot 会把 DTB 镜像 reload 一次，使得 DTB 镜像所在的物理内存归 lmb 所管理：    

*   ①boot_fdt_add_mem_rsv_regions 会将原先的内存 DTB 镜像所在的内存置为 reserve，保证该段内存不会被其他非法使用，保证接下来的 reload 数据是正确的；
*   ②boot_relocate_fdt 会在 bootmap 区域中申请一块未被使用的内存，接着将 DTB 镜像内容复制到这块区域（即归 lmb 所管理的区域）

> 注：若环境变量中，指定 “fdt_high” 参数，则会根据该值，调用 lmb_alloc_base 函数来分配 DTB 镜像 reload 的地址空间。若分配失败，则会停止 bootm 操作。因而，不建议设置 fdt_high 参数。

    接下来，do_bootm 会根据内核的类型调用对应的启动函数。与 linux 对应的是 do_bootm_linux。

*   ① boot_prep_linux

        为启动后的 kernel 准备参数

*   ② boot_jump_linux

![](https://leanote.com/api/file/getImage?fileId=57f737e8ab644106a000acf7)

    以上是 boot_jump_linux 的片段代码，可以看出：若使用 DTB，则原先用来存储 ATAG 的寄存器 R2，将会用来存储. dtb 镜像地址。

    boot_jump_linux 最后将调用 kernel_entry，将. dtb 镜像地址传给内核。

    下面我们来看下内核的处理部分：

    在 arch/arm/kernel/head.S 中，有这样一段：

![](https://leanote.com/api/file/getImage?fileId=57f737e8ab644106a000acea)

    _vet_atags 定义在 / arch/arm/kernel/head-common.S 中，它主要对 DTB 镜像做了一个简单的校验。

![](https://leanote.com/api/file/getImage?fileId=57f737e8ab644106a000acef)

    真正解析处理 dbt 的开始部分，是 setup_arch->setup_machine_fdt。这部分的处理在第五部分的 machine_mdesc 中有提及。

![](https://images2015.cnblogs.com/blog/334341/201702/334341-20170215165840816-1184117050.png)

    如图，是 setup_machine_fdt 中的解析过程。

*       解析 chosen 节点将对 boot_command_line 进行初始化。
*       解析根节点的 {size，address} 将对 dt_root_size_cells，dt_root_addr_cells 进行初始化。为之后解析 memory 等其他节点提供依据。
*       解析 memory 节点，将会把节点中描述的内存，加入 memory 的 bank。为之后的内存初始化提供条件。

*       解析设备树在函数 unflatten_device_tree 中完成，它将. dtb 解析成 device_node 结构（第五部分有其定义），并构成单项链表，以供 OF 的 API 接口使用。

下面主要结合代码分析：/drivers/of/fdt.c

 ![](https://images2015.cnblogs.com/blog/334341/201702/334341-20170215165026050-1645051544.png)

 ![](https://leanote.com/api/file/getImage?fileId=57f737e7ab644106a000ace4)

![](https://images2015.cnblogs.com/blog/334341/201702/334341-20170215165254566-653162640.png)

![](https://images2015.cnblogs.com/blog/334341/201702/334341-20170215165327535-1656536967.png)

![](https://images2015.cnblogs.com/blog/334341/201702/334341-20170215165358082-1160572355.png)

![](https://images2015.cnblogs.com/blog/334341/201702/334341-20170215165424660-81438212.png)

![](https://images2015.cnblogs.com/blog/334341/201702/334341-20170215165505004-584434076.png)

总的归纳为：
------

    ① kernel 入口处获取到 uboot 传过来的. dtb 镜像的基地址

    ② 通过 early_init_dt_scan() 函数来获取 kernel 初始化时需要的 bootargs 和 cmd_line 等系统引导参数。

    ③ 调用 unflatten_device_tree 函数来解析 dtb 文件，构建一个由 device_node 结构连接而成的单向链表，并使用全局变量 of_allnodes 保存这个链表的头指针。

    ④ 内核调用 OF 的 API 接口，获取 of_allnodes 链表信息来初始化内核其他子系统、设备等。