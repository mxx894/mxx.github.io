> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/186858463?utm_source=wechat_timeline)

**通过[一篇易懂的 CAN 通讯协议指南 1](https://zhuanlan.zhihu.com/p/162708070)，我们知道：**

*   **CAN 总线的 2 种架构 - 高速 CAN 和低速 CAN；**
*   **CAN 协议帧类型 - 数据帧，遥控帧，错误帧，过载帧；**
*   **线与机制，仲裁机制，位定时与同步。**

**以上基础的应用多数体现在硬件处理部分，所以只有少数体现在软件部分。从精通整个 CAN 通讯功能的角度来说，掌握基础还是非常重要的。本文主要是基于实际项目经验，通过 AUROSAR 的相关文档来介绍 CAN 通讯功能实现，这里通讯仅指 CAN 信号的接收与发送，也就是读与写。本系列 CAN 通讯功能实现将分为一篇 CAN 通讯读功能的实现介绍（一篇易懂的 CAN 通讯功能实现指南 2--READ）和一篇 CAN 通讯写功能的实现介绍（一篇易懂的 CAN 通讯功能实现指南 3--WRITE）.**

本篇将介绍 CAN 通讯读功能，在 AUTOSAR 架构涉及的模块主要有 Can Driver, Can Interface, Pdu Router, Com，如下图 1 所示。

![](https://pic1.zhimg.com/v2-47bc44ac40e5ca6e553928454d109784_r.jpg)

以下先通过两个场景来想象下 CAN 通讯读功能。

场景 1：变速箱控制需要发动机的相关信号，比如转速，扭矩。那么这时变速箱控制器（TCU）怎么获取发动机控制器（ECM）的相关信号呢？

![](https://pic4.zhimg.com/v2-84c579458289f871ac30c176edcce417_r.jpg)

场景 2：电机控制器（MCU）需要整车控制器 (VCU) 的相关信号，比如目标转速，目标扭矩等，同样地 MCU 怎么获取 VCU 的相关信号呢？

![](https://pic2.zhimg.com/v2-da8a8207680660fb196966f9b9658731_r.jpg)

当然以上场景都是通过 CAN 通讯实现，具体以场景 1 再进一步解释 TCU 如何获取 ECM 信号（比如说发动机扭矩信号）。我们知道 TCU 控制需要发动机扭矩信号，所以 TCU 会要求 ECM 提供，ECM 将该信号定义在某条报文中，发送到 CAN 总线，然后 TCU 去接收该报文，其大致过程如下：

![](https://pic4.zhimg.com/v2-1fd3ac4f91016505dcf3acb3cdb1c353_r.jpg)

当 TCU 接收到该报文后，也就是 TCU 芯片的相应寄存器就会去存储该报文数据。那么怎么去获取已存储的报文数据供 TCU 控制使用呢？根据 AUTOSAR 架构的分层，我们知道有：BSW（底层），RTE（接口层）和 ASW（应用层）。

这里根据 AUTOSAR CAN 通讯功能的架构定义，TCU 的底层软件先由 Can Driver 访问硬件，即提取相应寄存器的数据；再通知 Can Interface，继而经 Pdu Router 路由到 Com 模块，然后 Com **解包**数据，并存入 Com 的缓存区（Buffer）。

![](https://pic2.zhimg.com/v2-848fba333d7be17898b6a35d71d0c725_r.jpg)

最后 RTE 根据已设定的路径访问 Com 的缓存区，读取该扭矩信号数据，供 TCU 的 ASW 使用。以上就是一个 CAN 读功能的简介，下面将介绍上图的 BSW 部分，会先铺垫一些基础知识，再进入读功能。

**第 1 部分 基础知识**
---------------

为了更好地理解下文的 CAN 通讯功能，先介绍些相关的基础知识。

1.1 Can Driver（驱动层）
-------------------

CAN Driver（CanDrv）是通讯功能模块中的最底层内容，是硬件设备的直接对话接口，访问硬件设备的唯一接口，并为上层（仅为 Can Interface 模块）提供与硬件独立的应用编程接口。这里 CAN 驱动访问硬件的具体表现比如有：设置通道的位速率，位定时参数；CAN 控制器的工作模式；设定数据的存放位置；访问寄存器，读写数据等。

![](https://pic3.zhimg.com/v2-0ffc4f58528eeca12d93eb8bbad7b0d2_r.jpg)

1.2 Can Interface（接口层）
----------------------

Can Interface (CanIf) 模块是驱动层的抽象，即将所有硬件资源抽象化，使硬件与软件彻底分开，从而 CAN 驱动就只需关注，访问，控制相应的硬件控制器即可，CAN 接口层的上层模块不用再考虑硬件的位置，若要控制或使用硬件资源必须要经过 CAN 接口层处理。这就意味着 CAN 接口层在 CAN 驱动层之上，通讯服务层（Com、CanTp、Pdu Router）之下。

![](https://pic4.zhimg.com/v2-2a9ae870fd4ff122091cc7e60c37c25f_r.jpg)

1.3 Hardware Object（硬件对象）
-------------------------

为了介绍 L-PDU（数据链路层的数据协议单元），先了解下硬件对象（Hardware Object），其示意如下图 2。就是说一个 CAN 硬件单元（CAN Hardware Unit）通常包括一个或多个 CAN 控制器，每个 CAN 控制器都含硬件对象。用于接收的硬件对象，简称为 **HRH(**Hardware Receive Handle）; 用于发送的硬件对象，简称为 **HTH**（hardware Transmit Handle）。

![](https://pic4.zhimg.com/v2-c0750419fd19191e4db7b7acc0e2bc6b_r.jpg)

1.4 L-PDU
---------

HRH 和 HTH 的 PDU（因为在数据链路层，故称为 L-PDU）都由 ID，DLC 和 SDU 三部分组成，如下图 3。

![](https://pic4.zhimg.com/v2-5012310e4d9ac4e00b90c2f992eadbcb_r.jpg)

举例：设有一 CAN 控制器，其硬件对象使用情况如上图 3，在这添加数据来具象化，如下表。

![](https://pic1.zhimg.com/v2-98adce0c3915b49e5f9da033567f3ef8_r.jpg)

简单来说，通过该表可知硬件对象编号与 CAN ID 的映射关系，如果想获取 CAN ID 0x300 的数据，那么查找 HRH 为 2 对应的 L-PDU 即可。

1.5 触发模式
--------

为了介绍后续的 CAN 读写功能，在这先简单介绍下读写功能的触发模式。

**（1）中断模式（interrupt mode）**

中断模式下 CanDrv 处理由 CAN 控制器触发的中断，CanIf 是基于事件触发，当事件发生将被通知，相关的 CanIf 服务将被调用。比如设定 CAN 接收为中断模式，那么中断触发后，调用 CanDrv 的接收函数，接收完了 CanDrv 再调用 CanIf 的接收通知服务 (CanIf_RxIndication)。

**（2）轮询模式（polling mode）**

轮询模式下 CanDrv 被调度模块（Scheduler module）触发，执行随后的处理，比如设定 CAN 接收为轮询模式，那么 CAN 接收函数（Can_MainFunction_Read）就会在预定义的时间间隔被周期性地调用，比如**每几 ms 调用一次**。与中断模式同样，接收完了 CanDrv 再调用 CanIf 的接收通知服务（CanIf_RxIndication）。

**（3）混合模式（mixed mode）**

即（1）（2）的混合使用，这里不作展开。

第 2 部分 读过程
----------

2.1 从 CanDrv 到 CanIf
--------------------

![](https://pic4.zhimg.com/v2-e6954b7ab1bc9f3490bd4a53ad3df12b_r.jpg)

在 CanDrv 主要做两件事：**一是访问硬件，即寄存器，提取数据**；**二是数据提取成功后，通知上层 CanIf**。

2.1.1 读功能的触发模式
--------------

1） 轮询模式的 CanDrv 提取数据
--------------------

当轮询触发 Can 读（接收）时，在 CanDrv 和 CanIf 的具体过程如下图所示：

*   第 1 步，BSW 调度层面，EcuM 周期性访问 Can 的接收函数 Can_MainFunction_Read；
*   第 2 步，Can_MainFunction_Read 中检查所有潜在的新接收数据的 Can 控制器；
*   第 3 步，访问与接收硬件对象相关的寄存器，提取数据（L-PDU），即包括 ID, DLC, SDU；
*   第 4 步，当数据提取成功后，调用 CanIf_RxIndication 来通知 CanIf，并将接收的数据（L-PDU）传给 CanIf。

![](https://pic1.zhimg.com/v2-a907b16cd92b544f1603192d3f76d738_r.jpg)![](https://pic2.zhimg.com/v2-9afc3b4dca29f568daca80665745ea51_r.jpg)

通俗地再描述上面过程，比如：一个 10ms 的任务周期中调用 Can_MainFunction_Read，然后在 Can_MainFunction_Read 就会遍历已使用的 Can 控制器，如有数据更新就提取数据，最后数据全部提取完了，则需调用 CanIf_RxIndication 通知 CanIf 及更上层。注意这里提取数据是指读取相关寄存器数据，放入 L-PDU 中，所以详细地实现**需参考芯片手册对寄存器的定义**。

2）中断模式 CanDrv 接收数据
------------------

当中断触发 Can 读（接收）时，在 CanDrv 和 CanIf 的具体步骤如下图所示：

*   第 1 步，硬件层面， Can 控制器表明有一个成功的接收，触发接收中断 Receive_Interrupt；
*   第 2 步，CanDrv 检查满足接收条件，访问与接收硬件对象相关的寄存器，提取数据 (L-PDU)，即包括 ID, DLC, SDU；
*   第 3 步，当数据提取成功，调用 CanIf_RxIndication 来通知 CanIf，将新数据传给 CanIf。
*   第 4 步，CanIf 检测数据正常后调用 User_RxIndication 通知上层，将新数据传给上层。

![](https://pic2.zhimg.com/v2-85e8db116fa566314fc7b78802090d71_r.jpg)

2.1.2 函数说明
----------

对上述所用函数 Can_MainFunction_Read，CanIf_RxIndication 做简单说明，若要具体了解**可自行研读相应的 AUTOSAR 文档（若有直接接触项目，最好以项目现有资料或代码入手）**

**1） CanIf_RxIndication**

![](https://pic4.zhimg.com/v2-dac6015669fa952ec1507152a3e620b7_r.jpg)

**1-1) CanIf_RxIndication 的主要动作**

![](https://pic4.zhimg.com/v2-c18ebe37a4d917f5d0c207bd13f30233_r.jpg)

**1-2) CanIf_RxIndication 的参数**

![](https://pic4.zhimg.com/v2-04bab0bfcdfebd6ae07ccae50db1afa3_r.jpg)![](https://pic4.zhimg.com/v2-ee6c94b973f5a258ae17885b9fdef383_r.jpg)

2 读函数
-----

由于读函数高度依赖于硬件，所以在 CanDrv 文档中未见读函数的定义。关于读函数的细节，其实文档也算基本提及到了，比如根据硬件来提取相应的 Hrh, CanID，Dlc，L-PDU，也有其他内容，比如提取数据时，如何区分数据帧的标准格式与扩展格式，如下所示：

![](https://pic2.zhimg.com/v2-1afced19be2adbe14e0db534d92526a1_r.jpg)

2.2 从 CanIf 到 PduR
------------------

当 CanDrv 通知 CanIf 后，条件满足则 CanIf 通知它的 User，那这里的 User 是指哪些模块呢？从功能类型有：

*   CAN 通讯则是指 PDU Router 模块;（通讯路径：CanDrv--CanIf--PduR--Com）
*   UDS 服务则是指 CANTp 模块;（通讯路径：CanDrv--CanIf--CanTp--PduR--Com）
*   XCP 服务则是指 XCP 模块。（通讯路径：CanDrv--CanIf--XCP)

![](https://pic3.zhimg.com/v2-9071a842e410f6c068d5103d375b654e_r.jpg)

所以这里的 CanIf User 是 PduR，这里从 CanIf 通过调用 PduR_RxIndication 进入 PduR 模块，在 CAN 读过程中 PduR 模块主要是路由到 Com 模块。

![](https://pic3.zhimg.com/v2-42edb1e215452c5295c444cd702b75e6_r.jpg)

关于 PduR_CanIfIndication 的通用格式定义如下：

![](https://pic3.zhimg.com/v2-b7eaca1ff594596c26c303243828e366_r.jpg)

2.3 从 PduR 到 Com
----------------

本文不展开介绍 PduR 模块，就先简单理解为 PduR 的作用就是将使 PduR_CanIfRxIndication 调用 Com_RxIndication，而不是调用其他函数。通过下图的例子来了解下 PduR 是如何路由的。

![](https://pic1.zhimg.com/v2-e7b01441f45c7ac5543d5deb6a6c5b18_r.jpg)

当 Com_RxIndication 被调用，则在 Com 讲对数据解包，并存入 Com 的 buffer。

![](https://pic1.zhimg.com/v2-ad357703c87279ff15ad2e31f84709ac_r.jpg)

关于 Com_RxIndication 的定义如下：

![](https://pic4.zhimg.com/v2-4d71caa6af85cd9e6fef59bb5b32ab37_r.jpg)

将数据保存到 buffer，使用的 Com_CopyRxData 函数的定义如下：

![](https://pic2.zhimg.com/v2-8c87d0df6b0ff229f7ea1bb1008d6ba9_r.jpg)

第 3 部分 ASW 信号接收
---------------

当 CAN 总线数据被存入 Com 的 buffer，ASW（应用层）怎么去接收这些 buffer 里的数据呢？ASW 通过 RTE 接口函数 Com_ReceiveSignal 访问 Com 的 buffer，获取相应信号的数据。

![](https://pic2.zhimg.com/v2-29b78422b907e1d37a62e97e86bb7ac9_r.jpg)

关于 Com_ReceiveSignal 的定义如下：

![](https://pic1.zhimg.com/v2-2465f9178c6f38c83450d40a140ab92c_r.jpg)

简单地说，上述的函数 Com_ReceiveSignal 根据信号 ID 去索引 buffer 的 ID，两者的 ID 均根据报文中信号的排序序号来的，采用工具配置的话，体现在两者通过 Ref 链接方式 mapping。

Reference:
----------

[1] 基于 AUTOSAR 标准的汽车通讯及网络管理技术的设计及实现 [M], 杨永亮

[2] 参照 AUTOSAR 标准的总线通信协议栈的设计与实现 [M], 周海娟

[3] Specification of CAN Driver.pdf

[4] Specification of CAN Interface.pdf

[5] Specification of PDU Router.pdf

[6] Specification of COM.pdf