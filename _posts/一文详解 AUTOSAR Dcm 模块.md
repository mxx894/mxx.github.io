> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.360doc.com](http://www.360doc.com/content/21/1217/08/50578091_1009075619.shtml)

> 一文详解 AUTOSAR Dcm 模块

AUTOSAR Dcm 模块主要负责诊断相关的实现，其又主要分为三个子模块，分别为 DSL、DSD、DSP，接下来分别对其进行分析，以及最后对 Dcm 模块的配置进行梳理。  

**本文福利**：分享报告《**电驱高压平台开发面临的挑战**》，公众号对话框回复【**汽车 ECU 开发 008**】下载。

**_01_****_._**

**Dcm 模块 Dsl 子模块**

首先来看一下 DSL 子模块。  

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_1_20211217082724380)

**正文**

诊断通信管理 (Diagnostic Communication Manager, DCM) 模块作为 AutoSar 诊断模块的重要组成部分，主要负责诊断数据流和管理诊断状态，包括诊断会话、安全状态及诊断服务分配等。

主要功能贯穿汽车的开发生产及售后等过程，如开发过程中 EMC、DV 等实验均可使用诊断服务实现，生产过程中的软件下载更新、ECU 产线 EOL、汽车产线 EOL 等、售后过程中读取 DTC、控制输出调试功能等。

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_2_20211217082724459)

DCM 模块相关的标准主要包括三部分：ISO 14229(UDS，DCM 遵循的主要标准)、ISO 15031(ISO 15031 (1-7)) 及 SAEJ1939(OBD，与 OBD 相关的 $01 -$0A 服务)。

从网络分层角度看，DCM 模块属于上层模块，主要为应用层提供服务。主要包括 5-7 层，包括会话层服务及应用层等，会话层包括服务定时及服务分配等，应用层为具体的服务功能实现。

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_3_20211217082724802)

在 AUTOSAR 架构中，Dcm（Diagnostic Communication Manager）模块位于通信服务层。Dcm 模块是独立于具体的网络的（不依赖于具体的 CAN,Lin,Eth,Flexray 等网络来实现）。PduR 模块为 Dcm 模块提供独立于具体网络的接口。Dcm 模块从 PduR 模块接收诊断信息，Dcm 模块在内部处理和检查诊断消息。作为处理所请求的诊断服务的一部分，Dcm 将与其他 BSW 模块或 SW-Components(通过 RTE) 交互，以获取所请求的数据或执行所请求的命令。诊断服务处理与特定的服务请求强绑定（不同的诊断请求依赖于不同的一个或几个模块来实现）。通常，Dcm 将汇集收集到的信息，并通过 PduR 模块发送回消息。

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_4_20211217082725209)

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_5_20211217082725427)

**Dcm 和 Dem 的交互：**DEM 模块提供了检索与故障内存相关的所有信息的功能，以便 Dcm 模块能够通过从故障内存中读取数据重新响应测试人员的请求，通俗的讲就是 Dcm 能够读取 Dem 记录的 DTC 信息。

**Dcm 和 PduR 的交互：**PduR 模块接收和发送诊断数据。PduR 为 Dcm 模块提供一个与具体通信协议无关的接口。

**Dcm 和 ComM 模块的交互：**Dcm 模块可以指示状态 “活动” 和“非活动”用于诊断通信。Dcm 模块提供了处理通信需求 “完全 / 静默 / 无通信” 的功能。此外，Dcm 模块提供了在 ComM 模块要求时启用和禁用诊断通信的功能。

**SWC 通过和 RTE 接口和 Dcm 交互：**Dcm 模块在完成诊断功能的时候需要通过 RTE 接口来读写 / 函数调用其他 SWC 的数据 / 服务。

**BswM 和 Dcm 模块的交互：**如果 Dcm 的初始化是从引导加载程序跳转的结果，则 Dcm 通知 BswM 应用程序已更新。Dcm 也向 BswM 指示通信模式的改变。

**3.1 Dcm 子模块功能概述**
-------------------

诊断通信管理 (DCM) 主要包括三个子模块：诊断服务层(Diagnostic Service Layer，DSL)、诊断服务调度(Diagnostic Service Dispatcher, DSD)、诊断服务处理(Diagnostic Service Processing, DSP)

Diagnostic Service Layer：确定诊断数据请求和响应的数据流；监控和确保诊断请求和响应的时序，管理诊断状态（特别是诊断会话和安全状态）

Diagnostic Service Dispatcher：接收到的诊断请求转发给数据处理器；当数据处理器触发时，通过 PDUR 传输诊断响应。

Diagnostic Service Processing：处理实际的诊断请求。

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_6_20211217082725740)

**3.2 诊断会话层 (DSL)**
-------------------

DSL 模块主要用于诊断请求的处理及诊断时序的控制等，具体存在几个功能如下：

### **3.2.1 DSL 功能概述**

**1）处理诊断请求：**

收到请求时，PDUR 会将数据诊断请求数据从下层的 Buffer 中 Copy 到诊断的接收 Buffer 中，DSD 会从此 Buffer 中取出数据进行处理。

维持诊断功能在线（保活逻辑）。

**2）处理诊断响应：**

收到发送数据时，PDUR 会将数据诊断请求数据从 DSL 的发送 Buffer 中 Copy 到 PDUR 的发送 Buffer 中，具体后续处理由 CAN 的协议栈进行处理。

保证诊断时序。

支持周期性传输数据。

支持事件型响应（ResponseOnEvent）。

支持分段响应（segmented response）。

支持由应用触发的挂起响应（ResponsePending）。

**3）管理安全等级：**

用于获取或者设置安全等级

**4）会话状态处理：**

管理会话状态（session state）。

非默认会话的激活状态跟踪。

允许修改时序（应用可以通过诊断命令动态修改时序，也就是修改时序参数）。

**5）诊断协议处理：**

处理不同的诊断协议。

管理资源。

**6）通信模式处理：**

处理通信请求（Full-/Silent-/No Communication）。

标识诊断功能是否激活（active/inactive）。

打开 / 关闭（Enabling/disabling）所有类型的诊断传输。

### **3.2.2 将请求从 PduR 模块转发到 DSD 子模块**

带有诊断属性的 PDU 到达 PduR 模块后，PduR 调用 Dcm_StartOfReception, 通知 Dcm 模块接收的数据大小, 并提供第一帧或单帧的数据, 如果数据规模溢出缓冲区大小, 或者请求的服务不可用允许 Dcm 拒绝接收诊断数据。PduR 模块之后调用 Dcm_CopyRxData 函数请求 Dcm 模块拷贝数据到 Dcm buffer。

如果 DCM 模块接收诊断请求（Dcm_StartOfReception, 函数返回成功），PduR 模块就会调用 Dcm_TpRxIndication 通知到 Dcm 模块诊断请求接收完成。

### **3.2.3 维持诊断功能在线（保活逻辑）**

DSL 子模块需要实现诊断会话在线功能，测试者通过物理寻址方式发送 0x3E 80 诊断请求，DSL 接收到 0x3E 80 的请求后需要维持当前的诊断会话不退出（通过重置超时计数器 S3Server）。

### **3.2.4** **将 DSD 子模块的响应转发到 PduR 模块**

DSD 模块准完成诊断响应后会向 DSL 模块请求发送诊断响应报文。PduR 模块接收到请求后调用调用 PduR_DcmTransmit() 函数发送数据，数据发送后，PduR 模块调用 Dcm_TxConfirmation() 函数通知 Dcm 模块数据发送结果。

### **3.2.5 通用连接处理**

以 CAN 网络上的通用诊断连接处理为例。CAN 网络上的每个 ECU 节点内部都会记忆两个诊断请求 ID，物理寻址请求 ID 和功能寻址请求 ID。

示例：Tester 为连接到 CAN 网络上发送诊断请求的上位机，可发送任意诊断报文。ECU_A 本地配置的物理寻址 ID 为 0x728，功能寻址 ID 为 0x7DF。ECU_B 本地配置的物理寻址 ID 为 0x72E，功能寻址 ID 为 0x7DF。

**物理寻址：1:1 的单播诊断请求**

Tester 发送物理寻址 ID 为 0x728 的诊断请求时只有 ECU_A 会响应。

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_7_2021121708272668)

Tester 发送物理寻址 ID 为 0x72E 的诊断请求时只有 ECU_B 会响应。

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_8_20211217082726240)

**功能寻址：1:n 的广播诊断请求**

Tester 发送功能寻址 ID 为 0x7DF 的诊断请求时 ECU_A 和 ECU_B 都会响应。

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_9_20211217082726396)

### **3.2.6 发送 Busy 响应来保证诊断时序**

上位机发送诊断请求后会要求 ECU 在一个最大响应时间（一般是 50ms）内发出响应报文，但是对于一些请求的处理，ECU 内部会耗时比较长（比如 0x2E 写 NVM 的请求），来不及在规定时间发出响应。这个时候 ECU 可以发送 0x78（Pending）的否定响应吗来通知上位机 ECU 内部繁忙（Busy），需要上位机等待（Pending，一版 Pending 时间可以设置为 1 秒）。上位机收到 0x78（Pending）的否定响应码后就会等待，直到 ECU 发出积极响应或者 Pending 时间超时。这个发送 Bussy 响应来保证诊断时序的功能由 DSL 模块实现。

### **3.2.7 支持周期传输**

UDS 提供的 “ReadDataByPeriodicIdentifier (0x2A)” 服务，用于向由一个或多个 periodicDataIdentifiers 标识的 ECU 请求定期传输数据记录值。实际项目中没有用到过。

### **3.2.8 支持 ROE 传输。**

通过 UDS Service ResponseOnEvent (0x86)，测试人员可以请求 ECU 启动或停止传输由指定事件发起的响应。当注册一个事件进行传输时，测试者也指定相应的服务来响应 (例如: UDS service ReadDataByIdentifier 0x22)。实际项目中没有用到过。

### **3.2.9 支持分段响应**

如果启用 (DcmPagedBufferEnabled=TRUE)， Dcm module 将提供一种机制来发送比配置的和 allocated 诊断缓冲区更大的响应。也就是分段响应机制。实际项目中没有用到过。

### **3.2.10 支持由应用程序触发的响应**

在某些情况下，例如在例程执行的情况下，应用程序需要立即请求一个 NRC 0x78(响应等待)，该请求应立即发送，而不是仅仅在到达响应时间之前 (P2ServerMax 分别为 P2*ServerMax)。

当 Dcm 模块调用一个操作并得到一个错误状态 DCM_E_FORCE_RCRRP 时，DSL 子模块将触发 NRC 0x78(响应等待) 的负响应传输。这个响应需要从一个单独的缓冲区发送，以避免覆盖正在进行的请求处理。

### **3.2.11 管理安全等级**

**安全等级理解：**DCM 模块的每个诊断服务可以配置安全访问等级，一般有无需通过安全访问、需要通过安全等级 1 访问、需要通过安全等级 2 访问三种等级。比如对于 0x2E 下的服务都设定为需要过安全访问等级 1 后，我们在请求 0x2E 的服务时需要通过特定的安全校验后才能通过安全访问等级 1，这样才能得到 0x2E 服务的积极响应。

DLS 子模块应该保存当前激活的安全等级状态。DLS 模块提供两个接口用来设置和获取当前安全等级：

获取当前激活的安全等级: Dcm_GetSecurityLevel

设置新的安全等级: DslInternal_SetSecurityLevel()

DCM 模块初始化完后，安全等级设置为 0x00（DCM_SEC_LEV_LOCKED，使能了安全访问功能）。

如果我们通过安全校验算法后后使得安全等级不为 0，当发生以下任意事件后，安全等级状态将回到初始状态 0x00（DCM_SEC_LEV_LOCKED）：

1）诊断会话（session）从非缺省会话（0x0, default session）切换到非缺省会话，包括从当前非缺省会话切换到本身自己。

2）诊断会话（session）从非缺省会话（0x0, default session）切换到缺省会话。

### **3.2.12 管理会话状态**

会话：session，一般有三种，缺省会话（0x00, Default session）、编程会话（0x01, Programming session）、扩展会话（0x02Extended session）。每种会话下可以进行的诊断服务不同，根据实际需求由 OEM 自己定义。一般来说诊断（UDS）刷写功能需要在编程会话下进行，涉及到 NVM 关键存储数据的写功能需要在扩展会话下进行。根据实际需求可以自己定义会话，比如定义 0x60（EOL session）专门用于 EOL 工厂下线处理（关于 EOL 下线，后面将会由文章具体接收，请关注本公众号后续文章）。

DLS 子模块应该保存当前激活的会话状态。DLS 子模块应该提供两个接口用于获取 / 设置会话状态：

1）获取当前会话状态：Dcm_GetSesCtrlType()

2）设置会话状态：DslInternal_SetSesCtrlType()

DCM 模块初始化完成后，诊断会话进入缺省会话。

### **3.2.13 跟踪激活的非缺省会话**

从缺省会话进入非缺省会话后，S3Server 定时器就会开始计时（只要收到诊断请求报文就会清零），如果定时器超时（S3Server），DLS 模块就会将会话状态切换到缺省会话状态。

### **3.2.14 允许修改时序**

P2ServerMin, P2ServerMax, P2*ServerMin, P2*ServerMax, S3Server 这些参数值将会影响 DCM 模块的诊断响应时序。

P2ServerMin=0, P2*ServerMin=0, S3Server = 5 为固定值。协议参数影响诊断会话层的时序，不会影响到传输层时序。当协议激活时，可以通过以下方法修改其中一些定时参数:

· UDS Service diagnostics sessioncontrol (0x10)

· UDS 服务 AccessTimingParameter (0x83)

DSL 子模块提供了以下功能来修改定时参数:

· 提供活动定时参数，

· 设置新的定时参数。只允许在发送响应之后激活新的计时值

### **3.2.15 处理不同的诊断协议**

DSL 子模块需要处理不同的诊断协议。实际应用中一般都是使用 DCM_UDS_ON_CAN 协议。

### **3.2.16 管理资源**

由于资源有限，DSL 应该考虑以下几点作为设计提示:

1）在 Dcm 模块中只允许使用和分配一个诊断缓冲区，这个缓冲区随后用于处理诊断请求和响应

2）NRC 0x78 (Response pending) 响应的输出是通过一个单独的缓冲区完成的

3）支持分页缓冲处理

### **3.2.17 通信模式处理**

DCM 模块和 ComM 模块的交互由 DSL 子模块完成。ComM 模块把每个通信通道（channel）的当前通信状态（No-com, Full-com, Silent-com）通知给 Dcm 模块。Dcm 模块的诊断功能将会调用 ComM 模块的接口功能可能会阻止 Ecu 进入 shutdown 模式（ActiveDiagnostic ==’DCM_COMM_ACTIVE’时 ECU 必须保持唤醒状态，也就是不允许 进入 shutdown/sleep）。

应用程序可以调用 Xxx_SetActiveDiagnostic() 接口通知 Dcm 模块 ActiveDiagnostic 的当前状态。

**No Communication****：**

ComM 模块调用 Dcm_ComM_NoComModeEntered 接口通知 Dcm 模块通信关闭。Dcm 模块收到通知后就会 Disable 所有的诊断接收 / 传输。

**Silent Communication**:

ComM 模块调用 Dcm_ComM_SilentComModeEntered 接口通知 Dcm 模块通信静默。Dcm 模块收到通知后就会 Disable 所有的诊断传输。

**Full Communication**:

ComM 模块调用  Dcm_ComM_FullComModeEntered 接口通知 Dcm 模块通信静默。Dcm 模块收到通知后就会 Enable 所有的诊断接收 / 传输。

**Diagnostic Activation State**:

Dcm 将所有网络的内部诊断状态通知 ComM 模块。网络上的诊断状态有两种选择。在 “活动” 诊断状态下，Dcm 正在处理来自该网络的一个或多个诊断请求，或者 Dcm 处于非默认会话中。在 “非活动” 诊断状态下，Dcm 处于默认会话，并且没有在该网络上处理诊断请求。

当一个网络没有正在进行的通信时，Dcm 会将诊断 ac 激活状态设置为'inactive'。当网络上有诊断通信时，Dcm 将诊断状态设置为 “活动”。在任何非默认会话中，诊断状态保持为“活动” 状态。通信状态也可以通过 API Xxx_SetActiveDiagnostic 来控制。

**_02_****_._**

**Dcm 模块子模块 Dsd**

**接下来看一下 Dcm 模块下的 DSD 子模块**。  

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_10_20211217082726537)

**正文**

**3.3 诊断服务调度层 (DSD)**
---------------------

### **3.3.1 DSD 功能概述**

DSD 模块主要用于诊断服务的分配、服务执行环境及条件，会从接收的数据识别请求的服务类型 (0x10、0x27、0x22)，主要功能：

**1）检查诊断服务**：用于检查诊断服务执行的条件，如当前会话模式、诊断服务是否支持、安全访问等级。当判断条件不支持是会相应 0x7F 和具体的 NRC 否定响应码，通过后会由上一层处理服务 (具体处理也可以给出否定响应)

**2）汇总响应数据**：将判断得到的响应数据或者由 DSP 发送的响应数据发送给 DSL，由 DSL 向外发送数据

### **3.3.2 使用场景**

**1）接收诊断请求信息传输积极响应信息**

DSD 模块接收到诊断请求信息后会校验其有效性（session, security）。这种场景下，数据都是有效的，同时会反馈积极响应。DSD 模块会将数据传递给 DSP 子模块下对于的服务处理器。当 DSP 子模块下的服务处理器完成了数据处理后会触发 DSD 模块传输积极响应信息。

如果数据处理器将服务作为请求指示函数的一部分立即处理，则数据处理器可以在此指示函数 (“同步”) 中触发传输。如果处理需要较长的时间 (例如等待 EEPROM 驱动程序)，则数据处理器将延迟某些处理 (“异步”)。DSL 子模块包含响应挂起机制。数据处理器显式地触发传输，但是从数据处理器的上下文中触发的。

一旦接收到请求消息，相应的 DcmPduId 就会被 DSL 子模块阻塞。在此请求的处理过程中，不能再接收其他相同协议类型的请求 (例如，增强的会话可以通过 OBD 会话结束)，直到发送相应的响应消息，并再次释放 DcmPduId。

**2）接收诊断请求信息抑制积极响应**

这是前一个用例的子用例。在 UDS 协议中，可以通过在请求消息中设置一个特殊的位来抑制正响应。这种特殊的抑制处理完全在 DSD 子模块中执行。

**3）接收诊断请求信息抑制否定响应**

功能寻址的场景下，DSD 模块将抑制否定响应码为 0x11,0x12,0x31,0x7E 的否定响应。

**4）接收诊断请求信息传输否定响应信息**

拒绝请求消息并发送否定响应有许多不同的原因。如果诊断请求无效，或者请求在当前会话中不能执行，DSD 子模块将拒绝处理，并返回一个否定响应。

有许多理由拒绝执行一个格式符合的请求 message，例如，如果 ECU 或系统状态不允许执行。在这种情况下，DSP 子模块将触发一个消极的响应，包括 NRC 提供额外的信息，也就是否定响应的原因。

对于由多个参数组成的请求 (例如，一个 UDS Service ReadDataByIdentifier (0x22) 请求有多个需要读取的标识符)，每个参数被单独处理。每个参数都可能返回一个错误。如果至少一个参数被成功处理，这种请求将返回一个正响应。

**5）没有接收诊断请求发送积极响应**

UDS 协议中包含两个服务，一个请求发送多个响应。通常，一个服务用于启用 (和禁用) 另一个服务的事件或时间触发传输，ECU 在没有相应请求的情况下再次发送该服务。这些服务包括:

. 0x2A 服务，周期传输诊断响应服务。

. 0x86 服务，基于事件触发的响应服务。

**6）分段响应**

在诊断协议中，一些服务允许交换大量的数据，例如 UDS Service ReadDTCInformation (0x19) 和 UDS Service TransferData (0x36)。

在传统的方法中，ECU 内部缓冲区必须足够大，以保存最长的数据消息要交换 (最坏的情况)，并在传输开始之前填充完整的缓冲区。

ECU 中的 RAM 内存通常是一个关键的资源，特别是在较小的微型计算机中。在一种更节省内存的方法中，只填充部分缓冲区，传输部分缓冲区，然后再填充部分缓冲区，以此类推。这种分页机制只需要大幅减少的内存量，但需要定义良好的缓冲填充反应时间。

用户可以决定是否使用 “线性缓冲区” 或页面缓冲区进行诊断。

### **3.3.3 DSD 模块和其他模块的交互**

DSL 模块接收到诊断数据后，DSD 模块的服务就会被调用，DSD 模块将会执行以下的操作：

1）委托 DSP 子模块或 Dcm 外的外部模块处理请求。

2）跟踪请求处理。

3）将应用程序的响应发送给 DSL 子模块。

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_11_20211217082726802)

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_12_20211217082726974)

### **3.3.4 DSD 模块功能描述**

#### **3.3.4.1 检查诊断服务标识符**

DSL 接收到诊断信息后触发 DSD 模块去校验诊断信息 ID 的有效性。DCM 模块可以配置当前 ECU 支持多个诊断 ID，不过一般情况下，一个 ECU 仅支持一个物理寻址 ID 和一个功能寻址 ID，如果收到的诊断信息中的 ID 信息不是 ECU 配置好的物理 / 功能寻址 ID，则 DCM 返回否定响应码 NRC:0x11。

#### **3.3.4.2 处理正响应抑制响应位**

诊断报文的第二个字节的 bit7 如果置位 1（suppressPosRspMsgIndicationBit = TRUE），DSD 模块不用发送积极正响应（抑制正响应）。

#### **3.3.4.3 校验功能**

DSD 模块将会对一个诊断请求做一下 6 个方面的校验，如果全都通过了才会继续处理诊断请求，否则返回对应的否定响应码。

**1）制造商许可的校验。**实际项目中基本没有这个校验。

**2）SID 的校验。**服务 ID 的校验，也就是是否支持当前接收到的诊断 ID。

**3）诊断会话校验。**在诊断配置阶段，会预先定义每个诊断请求在什么会话下才支持。测试时，只有在支持的诊断会话下才会积极响应对应的诊断请求，否则返回 NRC:0x7E 的否定响应码。所以在诊断请求前，可以先通过 0x10 服务将诊断会话切换到需要的会话状态，同时通过 0x3E 服务维持诊断会话。

**4）安全访问等级的校验。**在诊断配置阶段，会预先定义每个诊断请求在安全等级下才支持。测试时，只有在支持的安全等级下才会积极响应对应的诊断请求，否则返回 NRC:0x33 的否定响应码。所以在诊断请求前，可以先通过 0x27 服务将安全等级切换到需要的安全等级。

**5）供应商许可的校验。**实际项目中基本没有这个校验。

**6）服务 ID 的模式规则状态校验。**如果 DcmDSD 模块引用的 DcmDsdSidTabModeRuleRef 模式规则不支持当前诊断 ID 服务或其子服务，则返回否定响应码。

#### **3.3.4.4 检查格式及是否支持子服务**

DSD 模块将会检查接收到的诊断信息的长度是否大于配置的最小长度，不满足则返回否定响应码 NRC:0x13。DSD 模块将会检查诊断信息的子服务是否支持，如果不支持则返回否定响应码 NRC:0x12。

#### **3.3.4.5 将诊断消息分发到 DSP 子模块**

DSD 子模块应为新接收的诊断服务标识符搜索 DSP 子模块的可执行功能，并调用相应的 DSP 服务解释器。

#### **3.3.4.6 组装积极或者消极响应**

当 DSP 子模块完成了重新请求的诊断服务的执行时，DSD 子模块将组装响应消息。

#### **3.3.4.7 开始传输诊断响应**

DCM 模块完成诊断操作后，DSD 模块通过调用 PduR 模块的信息发送接口将诊断响应（积极响应或者否定响应）发送出去。

DSL 子模块应将收到的来自 PduR 模块的确认转发给 DSD 子模块。

DSD 子模块应通过内部函数 DspInternal_DcmConfirmation() 将确认转发给 DSP 子模块。

在没有发送诊断 (响应) 消息 (抑制响应) 的情况下，DSL 子模块不发送任何响应。在这种情况下，没有数据确认从 DSL 子模块发送到 DSD 子模块，但 DSD 子模块仍将调用内部函数 DspInternal_DcmConfirmation()。

如果 Dcm 已经完全处理了请求，DSD 子模块应该通过调用 DspInternal_DcmConfirmation() 来完成诊断 tic 服务调度程序的一条诊断消息的处理。

DSP 子模块正在等待 DspInternal_DcmConfirmation() 功能的执行。所以它必须被发送，也是在没有数据确认提供。总之，这意味着在下列任何情况下:

. 积极响应

. 否定响应

. 抑制积极响应

. 抑制否定响应

DSD 子模块将通过调用 DspInternal_DcmConfirmation() 来完成。

**_03_****_._**

**Dcm 模块 DSP 子模块**

**最后来看一下 Dcm 模块下的 DSP 子模块**。  

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_13_20211217082727193)

**正文**  

**3.4 诊断服务处理层 (DSP)**
---------------------

### **3.4.1 DSP 功能概述**

DSP 为诊断服务的处理，当接收到 DSD 请求处理诊断服务，服务处理过程：

. 分析收到的请求消息

. 检查格式和是否支持寻址子功能

. 在 DEM、SWC 或其他 BSW 模块上获取数据或执行所需的函数调用

. 组装响应

#### **3.4.1.1 检查格式及是否支持其子功能**

DSP 对请求消息的分析导致格式错误或长度错误时，DSP 子模块将触发 NRC 0x13 (Incorrect message length or invalid format) 的负响应。

#### **3.4.1.2 组装消息**

DSP 子模块通过对包含响应服务标识符的响应消息进行组装，确定响应消息长度。

#### **3.4.1.3 否定响应码的处理**

除非另一个特定的 NRC 被指定，否则当 API 调用执行服务没有返回 OK 时，DSP 子模块应该触发 NRC 0x10 (generalReject) 的负响应。

#### **3.4.1.4 诊断模式声明组**

Dcm 作为诊断模式的模式管理员，管理以下 7 类模式切换:

1). DcmDiagnosticSessionControl (service 0x10)

2). DcmEcuReset (partly service 0x11)

3). DcmSecurityAccess (service 0x27)

4). DcmModeRapidPowerShutDown (partly service 0x11)

5). DcmCommunicationControl_<symbolic name of ComMChannelId>. (service

0x28)

6). DcmControlDTCSetting (service 0x85)

7). DcmResponseOnEvent_<RoeEventID> (service 0x86)

以 DcmSecurityAccess 为例，DCM 声明安全访问相关的模式组:

ModeDeclarationGroup DcmSecurityAccess

{

{

 DCM_SEC_LEV_LOCKED

DCM_SEC_LEV_1

 ...

DCM_SEC_LEV_63

}

 initialMode = DCM_SEC_LEV_LOCKED

};

ModeSwitchInterface  SchM_Switch_<bsnp>_DcmSecurityAccess

{

 isService = true;

 SecLevel currentMode;

};

#### **3.4.1.5 模式相关的请求执行**

可以根据模式条件限制请求的执行。这使得 Dcm 能够格式化的进行环境检查。

#### **3.4.1.6 Sender/Receiver 通信**

如果配置容器 DcmDspData 的参数 DcmDspDataUsePort 配置为 USE_DATA_SENDER_RECEIVER 或者 USE_DATA_SENDER_RECEIVER_AS_SERVICE 则 DcmDspDataUsePort 为一个 R-Port，需要为这个 R-Port 顶一个 Interface（一个参数，参数的实现数据类型为 DcmDspDataType）。

如果配置容器 DcmDspData 的参数 DcmDspDataUsePort 配置为 USE_DATA_SENDER_RECEIVER 或者 USE_DATA_SENDER_RECEIVER_AS_SERVICE 则 DcmDspDataUsePort 为一个 P-Port，需要为这个 P-Port 顶一个 Interface（一个参数，参数的实现数据类型为 DcmDspDataType）。

#### **3.4.1.7 将 SwDataDefProps 属性从 DEXT 文件传递到 Dcm Service SW-C**

用例: 将 SwDataDefProps 细节如 CompuMethod、DataContraints 和 Units 传递给 Dcm Service SW-C，并使它们在每个 DID DataElement / RoutineControl 信号中可用。有两个可选的工作流程。

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_14_20211217082727474)

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_15_20211217082727802)

#### **3.4.1.8 异步调用行为**

如果一个 Dem 函数返回 DEM_PENDING, Dcm 将在稍后的时间点再次调用该函数。

如果一个诊断任务请求的负面响应数量达到配置参数 eter DcmDslDiagRespMaxNumRespPend 中定义的值，Dcm 模块将停止处理活动的诊断请求，通知应用程序或 BSW(如果这个诊断任务意味着调用一个 SW-C 接口或 BSW 接口)，通过设置 OpStatus 参数，active 端口接口，DCM_CANCEL，并发送一个 NRC 0x10 的否定响应 (一般拒绝)。

如果之前的返回状态是 E_PENDING，那么 Dcm_SetProgConditions API 将在下一个 Dcm 主函数循环中被再次调用。

DCM_E_PENDING 的返回应该做一个重新触发 (例如在下一个 MainFunction 循环中)。

调用 OpStatus 为 DCM_CANCEL 的接口的返回值将被忽略。

### **3.4.2 UDS 服务**

Dcm 模块需要支持以下 UDS 服务

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_16_2021121708272852)

_Note: 具体每个服务的介绍，请关注本公众号的后续文章_

**2.1 F****unction definitions**
--------------------------------

### **2.1.1** **Dcm_Init**

函数原型：void Dcm_Init(const Dcm_ConfigType* ConfigPtr)

参数：

ConfigPtr：指向 Dcm 配置集合的指针（指向 const 数组的指针）

返回值：无

功能描述：根据动态配置初始化 Dcm 模块

### **4.1.2 Dcm_DemTriggerOnDTCStatus**

函数原型：

Std_ReturnType Dcm_DemTriggerOnDTCStatus(uint32 DTC, Dem_UdsStatusByteType DTCStatusOld, Dem_UdsStatusByteType DTCStatusNew)

参数：

DTC：DTC 标识符（代表某个具体的 DTC）

DTCStatusOld：DTC 改变前的状态

DTCStatusNew：DTC 改变后的状态

返回值：总是返回 E_OK

功能描述：UDS 状态字改变后就会调用这个接口，改变 DTC 状态。

### **2.1.2** **Dcm_GetVin**

函数原型：

Std_ReturnType Dcm_GetVin(uint8* Data)

参数：

Data：存放 VIN 码地址的指针

返回值:

E_OK：成功获取 VIN 到 Data 指向的地址空间

E_NOT_OK：在 DoIP 中将使用默认 VIN

功能描述：获取 VIN 码

### **2.1.3** **Dcm_GetSecurityLevel**

函数原型：

Std_ReturnType  Dcm_GetSecurityLevel(Dcm_SecLevelType* SecLevel)

参数：

SecLevel：获取的安全等级值存放到 SecLevel 指针指向的内存。

返回值：总是返回 E_OK

功能描述：获取安全等级值。

### **2.1.4** **Dcm_GetSesCtrlType**

函数原型：

Std_ReturnType Dcm_GetSesCtrlType(Dcm_SesCtrlType* SesCtrlType)

参数：

SesCtrlType：获取的会话状态值存放到 SesCtrlType 指针指向的内存。

返回值：总是返回 E_OK

功能描述：获取会话状态值。

### **4.1.5Dcm_ResetToDefaultSession**

函数原型：

Std_ReturnType Dcm_ResetToDefaultSession(void) 参数：

返回值：总是返回 E_OK

功能描述：将当前会话状态切换到 default 默认状态。

### **4.1.6Dcm_ SetActiveDiagnostic**

函数原型：

Std_ReturnType Dcm_SetActiveDiagnostic(boolean active)

参数：

Active：是否激活诊断功能的标志。

返回值：总是返回 E_OK

功能描述：如果 active == true，DCM 就会调用 ComM_DCM_ActiveDiagnostic()，否则不调用。

**4.2** **Call-back notifications**
-----------------------------------

介绍底层 BSW 模块提供的功能。回调函数的函数原型将在 Dcm_Cbk.h 文件中提供。也就是 Dcm 模块实现这些回调函数，其他模块（ComM, PduR 等）使用这些回调函数。

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_17_20211217082728349)

### **4.2.1Dcm_StartOfReception**

BufReq_ReturnType Dcm_StartOfReception(

PduIdType id,

const PduInfoType* info,

PduLengthType TpSduLength,

PduLengthType* bufferSizePtr

)

功能描述：PduR 模块接收到诊断报文的第一帧（单针 SF, 或者多帧的首帧 FF）时调用，通知 Dcm 模块开始接收诊断报文。

### **4.2.2 Dcm_CopyRxData**

BufReq_ReturnType Dcm_CopyRxData(

PduIdType id,

const PduInfoType* info,

PduLengthType* bufferSizePtr

)

功能描述：PduR 模块接收到多帧的连续帧（CF）时调用，将诊断数据传递到 DCM 模块。

### **4.2.3 Dcm_TpRxIndication**

void Dcm_TpRxIndication(

PduIdType id,

Std_ReturnType result

)

功能描述：在通过 TP API 接收到 I-PDU 后调用。

### **4.2.4 Dcm_CopyTxData**

BufReq_ReturnType Dcm_CopyTxData(

PduIdType id,

const PduInfoType* info,

const RetryInfoType* retry,

PduLengthType* availableDataPtr

)

功能描述：调用该函数获取 I-PDU 段 (N-PDU) 的传输数据。

### **4.2.5 Dcm_TpTxConfirmation**

void Dcm_TpTxConfirmation(

PduIdType id,

Std_ReturnType result

)

功能描述：这个函数是在 I-PDU 在其网络上传输后调用的，结果表明传输是否成功。

### **4.2.6 Dcm_TxConfirmation**

void Dcm_TxConfirmation(

PduIdType TxPduId,

Std_ReturnType result

)

功能描述：下层通信接口模块确认 PDU 发送，或者 PDU 发送失败。

### **4.2.7  Dcm_ComM_NoComModeEntered**

void Dcm_ComM_NoComModeEntered(

uint8 NetworkId

)

功能描述：ComM 模块通知 Dcm 模块 NetworkId 标识的网络进入了 COMM_NO_COMMUNICATION 状态。

**4.2.8 Dcm_ComM_SilentComModeEntered**

void Dcm_ComM_SilentComModeEntered(

uint8 NetworkId

)

功能描述：ComM 模块通知 Dcm 模块 NetworkId 标识的网络进入了 COMM_SILENT_COMMUNICATION 状态。

**4.2.8 Dcm_ComM_FullComModeEntered**

void Dcm_ComM_FullComModeEntered(

uint8 NetworkId

)

功能描述：ComM 模块通知 Dcm 模块 NetworkId 标识的网络进入了 COMM_FULL_COMMUNICATION 状态。

**4.3 Callout Definitions**
---------------------------

Callouts 是在 ECU 集成过程中必须添加到 Dcm 的代码片段。大多数 Callouts 的内容是手写的代码，对于一些 Callouts，Dcm 配置工具会生成一个默认实现，由集成器手动编辑。从概念上讲，这些 Callouts 属于 ECU 固件。

由于 Callouts 不是 Dcm 的服务，它们没有分配的服务 ID。注意:Autosar 架构不提供使用物理地址访问 ECU 内存的可能性。这是使用标识内存块的 BlockId 实现的。

根据这一点，Dcm 不能完全支持 ISO14229- 1 服务的实现，这些服务要求物理内存访问。因此，Dcm 定义了 callout 来实现这种内存访问。通过定义 BlockId 和物理内存地址之间的映射，可以简单地实现这个调出实现。

**4.4Scheduled functions**
--------------------------

### **4.4.1** **Dcm_MainFunction**

void Dcm_MainFunction(void)

**4.5Expected interfaces**
--------------------------

### **4****.****5****.1 Mandatory interfaces**

实现模块核心功能所需的所有接口

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_18_20211217082728599)

### **4.5.2 Optional interfaces**

实现模块的一个可选功能所必需的接口

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_19_20211217082728771)

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_20_2021121708272999)

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_21_20211217082729427)

### **4.5.3 Configurable interfaces**

Dcm 配置中定义 Dcm 将使用的实际函数的接口。根据配置的不同，这些功能的实现可以由其他 bsw 模块 (通常是 DEM) 或软件组件 (通过 RTE) 提供。

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_22_20211217082729818)

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_23_20211217082730193)

**_04_****_._**

**DCM 模块配置分析**

**前面对 Dcm 的三个子模块 DSL、DSD、DSP 进行了详细的梳理，接下来梳理一下 Dcm 模块配置**。

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_24_20211217082730474)

**正文**

**6.1 DCM**
-----------

DCM 模块包括 DcmConfigSet 和 DcmGeneral 两个配置容器。

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_25_20211217082730677)

**DcmGeneral**：Dcm 模块的通用配置项。

**DcmConfigSet**：这个容器包含配置参数和支持多个配置集的 DCM 模块的子容器。

### **6.1.1** **DcmGeneral**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_26_20211217082730834)

**DcmDDDIDStorage**：这个配置开关定义了 DDDID 定义是否存储为非易失性。

**DcmDevErrorDetect**：打开或关闭开发错误检测和通知。

**DcmRespondAllRequest**：如果设置为 FALSE, Dcm 将不会响应响应 ID 范围为 0x40 到 0x7F 或 0xC0 到 0xFF(响应 ID) 的诊断请求。

**DcmTaskTime**：允许配置周期周期任务的时间。

**DcmVersionInfoAp**：配置是否是使用版本检测 API。

### **6.1.2** **DcmConfigSet**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_27_2021121708273137)

**DcmDsd****：**DSD 诊断调度子模块配置容器。

**DcmDsl****：**DLS 诊断会话层子模块配置容器。

**DcmDsp****：**DSP 诊断服务处理子模块配置容器。

**DcmPageBufferCfg****：**也缓冲配置容器。

**DcmProcessingConditions****：**模式仲裁配置容器。

#### **6.1.2.1** **DcmDsd**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_28_20211217082731177)

**DcmDsdRequestManufacturerNotificationEnabled****：**对制造商允许启用或禁用请求通知机制。

**DcmDsdRequestSupplierNotificationEnabled**：对供应商允许启用或禁用请求通知机制。

以上两个参数一般都配置位 fasle。也就没有 **DcmDsdServiceRequestManufacturerNotification**,

**DcmDsdServiceRequestSupplierNotification** 这两个配置容器的配置。

**DcmDsdServiceTable**：这个容器包含每个具体诊断服务（0x10, 0x11 等）的配置 (DSD 参数)。

#### 6.1.2.2 **DcmDsdServiceTable**

**DcmDsdSidTabId**：可能有多个 DcmDsdServiceTable，DcmDsdSidTabId 用来标识一个 DcmDsdServiceTable。

**DcmDsdService**：一个具体诊断服务的配置容器。

#### **6.1.2.3 DcmDsdService**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_29_20211217082731349)

**DcmDsdServiceUsed**：配置是否使用该服务。  

**DcmDsdSidTabFnc**：ECU Supplier 特定组件针对特定服务的回调函数。该函数的原型如 <Module>_< diagnostics service > 所述。如果未配置此参数，则服务在 dcm 内部处理。

**DcmDsdSidTabServiceId**：诊断服务 ID。

**DcmDsdSidTabSubfuncAvail**：该服务是否有子服务。

**DcmDsdSidTabModeRuleRe****f**：对控制服务执行的 DcmDspModeRule 的引用。如果没有配置引用，则不进行模式规则检查。

**DcmDsdSidTabSecurityLevelRef**：引用允许在其中执行服务的安全级别。一个服务允许多个引用。

**DcmDsdSidTabSessionLevelRef**：对允许执行服务的会话级别的引用。一个服务允许多个引用。

**DcmDsdSubService**：这个容器包含服务的子服务的配置 (DSD 参数)。只有那些服务可以有子服务，这些子服务将 DcmDsdSidTabSubfuncAvail 配置为 TRUE。

#### **6.1.2.4 DcmDsdSubService**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_30_20211217082731569)

**DcmDsdSubServiceFnc**：ECU Supplier 特定组件针对特定服务的回调函数。ISOLAR 工具没有提供这个配置项。

**DcmDsdSubServiceId**：子服务 ID。

**DcmDsdSubServiceUsed**：是否使用该子服务。ISOLAR 工具没有提供这个配置项。

**DcmDsdSidTabModeRuleRe****f**：对控制服务执行的 DcmDspModeRule 的引用。如果没有配置引用，则不进行模式规则检查。

**DcmDsdSidTabSecurityLevelRef**：引用允许在其中执行服务的安全级别。一个服务允许多个引用。

**DcmDsdSidTabSessionLevelRef**：对允许执行服务的会话级别的引用。一个服务允许多个引用。

#### **6.1.2.5 DcmDsl**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_31_20211217082731757)

**DcmDslBuffer**：此容器包含诊断缓冲区的配置。

**DcmDslCallbackDCMRequestService**：每个 DcmDslCallbackDCMRequestService 容器使用 CallbackDCMRequestServices 接口定义一个 R-Port, 应用使用使用接口向 Dcm 模块请求协议更改的权限。

**DcmDslDiagResp**：此容器包含 Dcm 中自动 requestCorrectlyReceivedResponsePending 响应管理的配置。

**DcmDslProtocol**：这个容器包含 Dcm 中使用的诊断协议的配置。

#### **6.1.2.6 DcmDslBuffer**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_32_20211217082731927)

**DcmDslBufferSize**：诊断缓冲区大小 (以字节为单位)。对于线性缓冲区，大小应该和最长的诊断消息 (请求或响应) 一样大。对于分页缓冲区，大小会影响应用程序性能。  

#### **6.1.2.7 DcmDslCallbackDCMRequestService**

一般实际应用中不使用这个配置。

#### **6.1.2.8 DcmDslDiagResp**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_33_2021121708273284)

**DcmDslDiagRespMaxNumRespPend**：一个请求允许的带有响应码 0x78 (requestCorrectlyReceivedResponsePending) 的最大否定响应数。如果 Dcm 达到此限制，将自动发送 0x10 (generalReject) 最终响应，并取消服务处理。

#### **6.1.2.9 DcmDslProtocol**

**DcmDslProtocolRow**：该容器包含 Dcm 中使用的一种特定诊断协议的配置。

#### **6.1.2.10 DcmDslProtocolRow**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_34_20211217082732240)

**DcmDslProtocolID**：正在配置的 DCM DSL 协议的诊断协议类型。一般都是基于 UDS 的 CAN 诊断。  

**DcmDslProtocolMaximumResponseSize**：定义响应消息的最大长度。

**DcmDslProtocolPriority**：协议抢占时使用的协议优先级。高优先级的协议可能会抢占低优先级的协议。数值越低表示协议优先级越高。

**DcmDslProtocolTransType**：仅当协议类型为 DCM_ROE_ON_xxx 时，此参数有效。

**DcmSendRespPendOnTransToBoot**：参数指定 ECU 在转换到引导加载程序之前是否应该发送 NRC 0x78(响应等待)(参数设置为 TRUE)，或者是否应该启动转换而不发送 NRC 0x78(参数设置为 FALSE)。

**DcmTimStrP2ServerAdjust**：该参数用于确保在总线上的诊断响应在到达 P2 之前可用，通过调整当前的 DcmDspSessionP2ServerMax。该参数主要表示由 DCM 发起传输到消息实际传输到总线的时间之间依赖于软件架构的通信延迟。

**DcmTimStrP2StarServerAdjust**：该参数用于保证在到达 P2Star 之前，总线上的诊断响应是可用的，通过调整当前的 DcmDspSessionP2StarServerMax。该参数主要表示由 DCM 发起传输到消息实际传输到总线的时间之间依赖于软件架构的通信延迟。

**DcmDemClientRef**：在 Dem 配置中引用 DemClient。由 Dem 用于区分不同的客户端调用。

**DcmDslProtocolRxBufferRef**：引用已配置的诊断缓冲区，该缓冲区用于接收协议的诊断请求。

**DcmDslProtocolSIDTable**：对用于此协议的诊断请求处理的服务表的引用。

**DcmDslProtocolTxBufferRef**：引用已配置的诊断缓冲区，用于传输协议的诊断响应。

**DcmDslConnection**：这个容器包含一个特定协议的通信通道配置。注意，它允许与多个 Tester 通信，因此可以为一个协议配置多个连接。

#### **6.1.2.11 DcmDslConnection**

**DcmDslMainConnection****：**这个容器包含诊断协议的主连接的配置。

**DcmDslPeriodicTransmission****：**这个容器包含定期传输连接的配置。实际应用中一般使用不到。

**DcmDslResponseOnEvent**：该容器包含 ResponseOnEvent 连接的配置。

#### **6.1.2.12 DcmDslMainConnection**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_35_20211217082732459)

**DcmDslProtocolRxTesterSourceAddr**：测试器源地址唯一地描述了一个客户端，并将在跳转到 Bootloader 接口中使用。对于通用连接 (MetaDataLength>= 1 的 DcmPdus)，不需要此参数。

**DcmDslPeriodicTransmissionConRef**：对用于处理周期性传输事件的周期性传输连接的引用。实际应用中一般很少使用。

**DcmDslProtocolComMChannelRef**：引用接收 DcmDslProtocolRxPdu 和发送 DcmDslProtocolTxPdu 的 ComMChannel。

**DcmDslROEConnectionRef**：引用一个用于处理 ResponseOnEvent 事件的 ResponseOnEvent 连接。实际应用中一般很少使用。

**DcmDslProtocolRx**：此容器包含诊断连接中接收通道的配置参数。

**DcmDslProtocolTx**：此容器包含诊断连接中传输通道的配置参数。

#### **6.1.2.13 DcmDslProtocolRx**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_36_20211217082732646)

**DcmDslProtocolRxAddrType** ：选择接收信道的寻址类型。1:1 通信采用物理寻址，1:1 通信采用功能寻址。

**DcmDslProtocolRxPduRef**：参考 EcuC 中用于此接收通道的 Pdu。

#### **6.1.2.14 DcmDslProtocolTx**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_37_20211217082732818)

**DcmDslProtocolTxPduRef**：参考 EcuC 中用于此传输通道的 Pdu。

#### **6.1.2.15 DcmDsp**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_38_202112170827335)

**DcmDspDDDIDcheckPerSourceDID**: 使用 ReadDataByIdentifier (0x22) 定义对每个源 DIDs 的会话、安全性和模式依赖项的检查。

true: Dcm 模块应该用一个 ReadDataByIdentifier (0x22) 检查每个源 DIDs 的会话、安全性和模式依赖关系，DID 的范围是 0xF200 到 0xF3FF。

false: Dcm 模块不检查每个源 DIDs 的会话、安全性和模式依赖关系，每个源 DIDs 的 ReadDataByIdentifier (0x22) 的 DID 范围是 0xF200 到 0xF3FF。

**DcmDspMaxDidToRead**: 指示在单个 “ReadDataByIdentifier” 请求中允许的最大 DIDs。

**DcmDspMaxPeriodicDidToRead**: 允许在单个 “ReadDataByPeriodicIdentifier” 请求中读取的最大 periodicDIDs。

**DcmDspPowerDownTime**: 此参数向客户端指示服务器将保持下电顺序的备用顺序的最小时间。

**DcmDspClearDTC**: 该容器包含 Clear DTC 服务的配置。

**DcmDspComControl**: 提供 CommunicationControl 机制的配置。

**DcmDspCommonAuthorization**: 该容器包含多个服务 / 子服务相同的公共授权的配置 (参数)。

**DcmDspControlDTCSetting**: 提供 ControlDTCSetting 机制的配置。

**DcmDspData**: 该容器包含属于 DID 的 Data 的配置 (参数)。

**DcmDspDataInfo**: 这个容器包含一个 Data 的配置 (参数)。

**DcmDspDid**: 这个容器包含 DID 的配置 (参数)。

**DcmDspDidInfo**: 这个容器包含 DID 的 Info 的配置 (参数)。

**DcmDspDidRange**: 这个容器定义 DID 范围。

**DcmDspEcuReset**: 该容器包含 DcmDspEcuReset 服务的配置。

**DcmDspMemory**: 这个容器包含内存访问的配置。

**DcmDspPeriodicTransmission**: 此容器包含定期传输调度程序的配置 (参数)。

**DcmDspPid**: 该容器定义了 PID 对 DCM 的可用性。

**DcmDspRequestControl**: 此容器包含 “请求控制机载系统、测试或组件” 服务 (服务 $08) 的配置 (参数)。

**DcmDspRequestFileTransfer**: 该容器包含 RequestFileTransfer 的配置。该容器仅在配置了 RequestFileTransfer 时才存在。

**DcmDspRoe**: 提供 ResponseOnEvent 机制的配置。

**DcmDspRoutine**: 这个容器包含例程的配置 (参数)>

**DcmDspSecurity**: 该容器包含安全级别配置 (DSP 参数)(每个安全级别) 描述该容器包含 DcmDspSecurityRow 的行。

**DcmDspSession**: 父容器保存单行来配置特定的会话。

#### **6.1.2.16 DcmDspComControlAllChannel**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_39_20211217082733193)

DcmDspAllComMChannelRef: 引用 ComM 通道。

#### **6.1.2.17 DcmDspCommonAuthorization**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_40_20211217082733366)

**DcmDspCommonAuthorizationModeRuleRef**: 引用一个 DcmModeRule。控制此服务 / 子服务的模式规则。如果没有参考，则不进行模态规则检查。

**DcmDspCommonAuthorizationSecurityLevelRef**: 引用 DcmDspSecurityRow 允许控制此服务 / 子服务的安全级别。如果没有引用，则不进行安全级别检查。

**DcmDspCommonAuthorizationSessionRef**: 引用 DcmDspSessionRow 允许控制此服务 / 子服务的会话。如果没有引用，则不应检查会话级别。

#### **6.1.2.18 DcmDspControlDTCSetting**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_41_20211217082733552)

**DcmSupportDTCSettingControlOptionRecord**: 这个配置开关定义请求消息中是否通常支持 DTCSettingControlOptionRecord。  

**DcmDspControlDTCSettingReEnableModeRuleRef**：引用 DcmModeRule。控制 DCM 重新启用 controlDTCsetting 的模式规则。DCM 模块应执行一个 ControlDTCSetting。关闭 (调用 Dem_EnableDTCSetting())，以便不再满足所引用的模式规则。

#### **6.1.2.19 DcmDspData**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_42_20211217082733709)

**DcmDspDataByteSize**: 以字节为单位定义数组长度或变量的最大数组长度。

**DcmDspDataConditionCheckReadFnc**: 这是函数名字（函数指针）。如果读取这个 DID 的时候需要应用去检查一些系统状态，就可以定义这个函数去实现这个功能。

**DcmDspDataConditionCheckReadFncUsed**: 状态检查函数是否被使用。

**DcmDspDataEndianness**: 定义诊断请求或响应消息中属于 DID 的数据的字节顺序

**DcmDspDataFreezeCurrentStateFnc**: 函数名（函数指针），请求应用程序冻结 IOControl 的当前状态。(FreezeCurrentState-function)。

**DcmDspDataGetScalingInfoFnc**: 函数名（函数指针），向应用程序请求 DID 的伸缩信息。(GetScalingInformation-function)。该参数与接口 Xxxx_GetScalingInformation 有关。

**DcmDspDataReadDataLengthFnc**: 函数名（函数指针），向应用程序请求 DID 的数据长度。(ReadDataLength-function)。该参数与接口 Xxx_ReadDataLength 有关。

**DcmDspDataReadEcuSignal**: 由 DCM 读访问某个 ECU 信号的函数名称。(IoHwAb_Dcm_Read < EcuSignalName>function)。仅当 DcmDspDataUsePort==USE_ECU_SIGNAL 有效。

**DcmDspDataReadFnc**: 函数名向应用程序请求 DID 的数据值。(ReadData-function)。该参数与接口 Xxx_ReadData 有关。

**DcmDspDataUsePort**: 定义要访问数据的接口应使用。

#### **6.1.2.20 DcmDspDidInfo**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_43_20211217082733896)

#### **6.1.2.21** **DcmDspDidRead**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_44_20211217082734115)

#### **6.1.2.22 DcmDspDidWrite**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_45_20211217082734302)

#### **6.1.2.23 DcmDspDid**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_46_20211217082734506)

#### **6.1.2.24 DcmDspDidSignal**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_47_20211217082734662)

#### **6.1.2.25 DcmDspRouting**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_48_20211217082734865)

#### **6.1.2.26 DcmDspRequestRoutineResults**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_49_2021121708273552)

#### **6.1.2.27 DcmDspStartRoutine**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_50_20211217082735271)

#### **6.1.2.28 DcmDspStopRoutine**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_51_20211217082735474)

#### **6.1.2.29 DcmDspSecurity**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_52_20211217082735662)

#### **6.1.2.30 DcmDspSecurityRow**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_52_20211217082735662)

#### **6.1.2.31 DcmDspSessionRow**

![](http://image109.360doc.com/DownloadImg/2021/12/1708/236049100_53_20211217082735849)

* * *