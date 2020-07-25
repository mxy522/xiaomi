# UFS2.1 boot

## **Description**

1, 在UFS中有两个logical units\(Boot LU A and Boot LU B\),可以用来储存boot code,但是在开机过程中, 只能有一个可以被active, \(此处概念类似于eMMC中的boot partition, 有两个Boot one/two\)

2, 任何一个logical unit可以被configure为"Boot LU A" or "Boot LU B", 但不能多于一个的Boot LU A 或B,

3, 这个logical unit 在boot时被active, mapping到Boot well known logical unit\(W-lUN=30h\), 并被Host读取,

## **Boot Configuration,**

1, 对一个可以boot 的UFS device来讲, 如果Device descriptor中的

bBootEnable

被 set为01h, boot feature就会enabled,

2,

dNumAllocUnits

代表一个unit 可以被分配的unit logical size,

3,

bBootLunID

代表一个logical unit 被作为boot logical unit ID, 可以是"Boot LU A" or "Boot LU B"

4, 一个logical unit通过写

bBootLunEn

attribute, 使得这个unit在boot时被active,

![](D:\work\Ynote\chenbestdone@126.com%281%29\57c75ed36ebe48f089c88a00e27edbef\clipboard.png)

host不能设置bBootLunEn to ‘Reserved’ values,

一旦bBootLunEn被configure, 一个active boot logical unit就会被mapped 到the Boot well known boot logical unit \(W-LUN = 30h\),

下图举例, 一个UFS device有8个logical units, 其中LU1 和LU4分别被configure为"Boot LU A"和"Boot LU B"

另外LU1是active one\(bBootLunEn = 01h\)

![](D:\work\Ynote\chenbestdone@126.com%281%29\f9cbb9ba4f4c4776b0c117939fada6a5\clipboard.png)

## 13.1.3 Initialization and boot code download process

* Partial initialization,

* boot transfer,

* initialization 完成,

### 13.1.3.1 Partial initialization

partial initialization是在上电之后 或HW reset 或EndpointReset之后, 包括整个UFS stack阶段,

当partial initialization结束, Unipro boot sequence应该已经完成,

* 如果device的Descriptor

bDescrAccessEn

被设置为01h,UTP layer应该可以读取device descriptor, 可以执行read command, Test Unit Ready command,

* 如果

bDescrAccessEn

被设置为00h, 只有device的Descriptor可以被accessible.

在UFS protocol stack中, 执行初始化操作, 是在UFS host and UFS device端的每层都会执行.

* a\) Physical Layer\(M-Phy\)
  * 在reset events 之后, physical layer将会从Disabled state 进入Hibern8 state,
* b\) Link Layer\(Unipro\)
  *  host与device端都会被执行,

1\)用DME\_RESET.req primitive来reset Unipro stack,

2\)通过DME\_RESET.cnf\_L primitive来确认,Reset是否完成,

3\)用DME\_ENABLE.req primitive使能enable Unipro stack,

4\)通过DME\_ENABLE.cnf\_L primitive来确认, 使能是否完成,

5\)用DME\_LINKSTARTUP.req primitive来初始化UniPro Link StartUp sequence, Unipro linkup startup 包含一系列的多任务handshakes, 用来在Host与device端之间,建立双向通信initial link通路,

6\)通过DME\_LINKSTARTUP.cnf\_L primitive来确认link startup是否完成.

c\) UFS Transport Layer \(UTP\)

在Device与host端UFS layer初始化末尾,host 应该发送一个Nop out UPIU 来验证device端的UTP layer是否ready,

由于某些因素, device端的UTP还没有被初始化, 因此可能并没有办法发送NOP in UPIU 来迅速迅速响应Nop out UPIU, host会一直等待, 直到收到device端发出的Nop in UPIU,

当host收到NOP in UPIU, host就知道device端的UTP layer已经ready, 可以执行UTP transactions,

d\) link configuration,

host 可以通过Unipro layer的DME primitives来configure Link attribute, 比如Gear, HS series, RX或TX的PWM mode,

e\) Device Descriptor Reading

UFS host此时可以读取device descriptor, \(比如 Device Class/Subclass, Boot enable, Boot LU size 等\),

* 如果bDescrAccessEn被设置为01h, host可以访问Device descriptior,

* 如果bDescrAccessEn被设置为00h, host只可以访问bDescrAccessEn这一个descriptior,

### 13.1.3.2 Boot transfer

以下步骤可以被执行, 只有当bBootEnable 被set,

Boot code download -// 这里可以看出, 将eMMC的boot code读出的步骤,

* UFS host 执行一个TEST UNIT READY 给Boot well known logical unit, 用来验证是否这个latter可以被accessed,

如果命令执行成功, UFS host通过SCSI READ来读取Boot well known logical unit,

* 此时,Devce将会发送一个boot code在Upstream link, 在这个阶段, the Boot well known logical unit可以被accessible,

### 13.1.3.3 Initialization completion

当host完成了从Boot well known logical unit读出boot code后, host将会设置fDeviceInit flag为"01h", 这样是为了告诉Device, initialization已经完成,

当device initialization完成时, 将reset fDeviceInit flag, host会不断poll 这个fDeviceInit flag来确认device端是否完成initialization,

当

fDeviceInit

被reset, device就已经准备好接受任何command,

![](D:\work\Ynote\chenbestdone@126.com%281%29\25661714d38c4ece9bf0ec0b0bf0cc24\clipboard.png)

## 13.1.6 与boot相关的configuration,

The following parameters refer to boot capabilities.

* Device Descriptor parameters:

* bBootEnable \(Boot Enable\)

* bDescrAccessEn \(Descriptor Access Enable\)

* bInitPowerMode \(Initial Power Mode\)

* bInitActiveICCLevel \(Initial Active ICC Level\)

* Unit Descriptor parameters for Boot LU A and Boot LU B:

* bLUEnable \(Logical Unit Enable\)

* bBootLunID \(Boot LUN ID\)

* bLUWriteProtect \(Logical Unit Write Protect\)

* bMemoryType \(Memory Type\)

* dNumAllocUnits \(Number of Allocation Units\)

* bDataReliability \(Data Reliability\)

* bLogicalBlockSize \(Logical Block Size\)

* bProvisioningType \(Provisioning Type\)

NOTE These parameters are non volatile and they may be programmed during the system manufacturing phase.

In addition to the parameters mentioned, the following attributes are relevant for device initialization and

boot

* bBootLunEn \(Boot LUN Enable\)
* bRefClkFreq \(Reference Clock Frequency value\)

以下内容来自Unipro Spec 1.6

9.11.2 Boot Procedure

* UniPro boot procedure是跟着一个cold reset, 包括Power-on reset 或一个warm reset,

* boot 过程是DME User通过

DME\_ENABLE.req primitive

来启动boot procedure,

* unipro boot过程是通过控制DME实现, 确保Unipro layer从L1.5到L4,按照顺序被初始化,DME要使能一个layer之前, 该layer下的layer都已经被enable, 比如说L2层不能开始AFC exchange,除非L1.5层已经执行了link startup sequence, PHY是可以正常运作的.

* 当收到上面的

DME\_ENABLE.req primitive

, Unipro 每一layer都可以被DME enable, 通过

&lt;

layer-identifier

&gt;

\_LM\_ENABLE\_LAYER.req primitive, enable后, 每一layer应该执行它们自己的初始化, 每一layer用

&lt;

layer-identifier

&gt;

\_LM\_ENABLE\_LAYER.cnf\_L primitive来告知DME layer是否已经初始化完成.

* 在这个阶段的结尾, Link转为

LinkDown state

, DME将会用

DME\_ENABLE.cnf\_L primitive

通知DME User这个状态,

* DME User将会执行boot sequence,通过DME发送

DME\_LINKSTARTUP.req primitive

.命令转换Link state从

LinkDown state

到

Linkup state,

* 收到

DME\_LINKSTARTUP.req primitive

, DME将通过

&lt;

layer-identifier

&gt;

\_LM\_LINKSTARTUP.req primitive

.启动L1.5, 随后L2, 每层layer都会用

&lt;

layeridentifier

&gt;

\_

LM\_STARTUP.cnf\_L primitive

来告知DME, layer已经完成相应layer的启动程序, Unipro boot进程确保L2不会开始AFC exchange直到L1.5执行了Link startup 并且Phy可以使用.

* 这个阶段的最后, Link 到达LinkUp state, DME通过

DME\_LINKSTARTUP.cnf\_L primitive

告知DME user这个条件.

* 为了能能够通过Unipro stack进行通信, Network 和Transport layer也需要被configured, 在收到DME\_LINKSTARTUP.cnf\_L之后, Network Layer和Transport的一些Attributes要求被set.

* 当收到DME\_LINKSTARTUP.cnf\_L和configuring Network Layer and Transport Layer Attributes, boot process 完成, Link已经准备好data transfer,

![](D:\work\Ynote\chenbestdone@126.com%281%29\1e1563623e5646c0b60679265b588ab2\clipboard.png)

![](D:\work\Ynote\chenbestdone@126.com%281%29\c88cda8b259945a08db2cb6a3da72790\clipboard.png)

![](D:\work\Ynote\chenbestdone@126.com%281%29\d74fa69b63ac42d29ff32c973851681a\clipboard.png)

