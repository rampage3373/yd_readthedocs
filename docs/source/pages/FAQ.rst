=========
常见问题
=========

**投资者需要做哪些以保证性能最优化？**
| 关于延时，你需要关注的就是两个：
| 1. 做好席位优选，这是最重要的
| 2. 保证你调用API的延时在450ns以下，这个延时指调用insertOrder到报单实际出柜台的时间间隔

**为什么在login的时候coredump？**
| 首先，检查产生的coredump文件，程序是否在getIPMacInfo函数出错退出，如果是的话，可能是因为网卡问题导致的。请手工调用getifaddrs(struct ifaddrs \**ifs)查看是否正常。

**为什么会在getIPMacInfo函数中coredump？**  
| 通常是客户的网卡配置出现问题，请客户使用以下代码文件进行调试，我们程序中就是通过这段代码来获取客户的MAC的。

**如何实现多用户登录？**
| 易达系统支持在同一个进程中实例化多少API实例，由于报单的线程即为客户线程本身，因此只需要按需对回报接收线程绑核处理即可。如果策略关心回报速度或者会在回报中下单，建议给每一个API对象的接收线程绑核，并且设置TradingServerTimeout=-1忙查询回报。如果希望节约内核，建议所有API对象绑定在一个核上，并且设置TradingServerTimeout为一个大于0的值。

**如果UDP行情会发生丢包，如何重发？**
| 由于本系统面向的客户的特殊性，在交易机会错失以后，补单是没有意义的，所以本系统不考虑重发

**如果只用交易，不用MD data，是否只有TCPTradingSocketCPUID和TCPTradingCPUID两个线程？绑定两个核就够了？**
| 如果不用MD（ConnectTCPMarketData=no且ReceiveUDPMarketData=no），只有TCPTradingCPUID一个线程，绑定一个核就够了。TCPTradingCPUID是指“Affinity CPU ID for thread to receive TCP trading notifications, -1 indicate no need to set CPU affinity”，也就是说收报单和成交回报线程所需要的CPUID。TCPTradingSocketCPUID是指“Affinity CPU ID for incoming message of TCP trading socket, -1 indicate no need to set CPU affinity. Ignored in windows”，也就是说将socket的接收终端指定到哪个CPUID上，并不需要单独分配一个CPU给它。从Linux kernel的解释上来说，如果将Socket的接收软中断直接指定给接收CPU会提高性能并降低延迟，因此TCPTradingSocketCPUID应该设置为TCPTradingCPUID。实际测试结果表明并没有明显的性能提升，可能的原因是onload bypass stack已经做了优化处理。

**因为onload-latency是busy_loop，如果开启，加上之前两个，是不是要三个？**
| onload提供了kernel bypass的TCP/IP协议栈，这个协议栈有部分还是运行在kernel（主要是驱动程序接收、DMA和发送完毕的中断通知），有部分运行在调用recv和send的用户线程上。运行在kernel的部分一般会以中断的方式运行在0号CPU上（因此在绑定的时候尽可能将0号CPU空出来，否则整个系统的性能会显著下降，因为影响了操作系统）。send调用是由策略线程来发起的，ydAPI中会等待发送成功后返回；recv线程（也就是TCPTradingCPUID）只接收回报信息。开启onload并没有增加新的线程。 

**亲和性是否用Numactl设置总核数就行，比如numactl -C 0,1,2，不需要单独指定线程？**
| 这个问题比较复杂。首先，高速网卡一般安装在直连0号物理CPU（可以有多个核），可以使用lspci –vvv \| grep Solarflare来查看该网卡在哪个物理CPU上，

.. code-block:: console

   65:00.0 Ethernet controller: Solarflare Communications XtremeScale SFC9250 10/25/40/50/100G Ethernet Controller (rev 01)
   Subsystem: Solarflare Communications XtremeScale X2522 10G Network Adapter
   Control: I/O- Mem+ BusMaster+ SpecCycle- MemWINV- VGASnoop- ParErr- Stepping- SERR+ FastB2B- DisINTx+
   Status: Cap+ 66MHz- UDF- FastB2B- ParErr- DEVSEL=fast >TAbort- <TAbort- <MAbort- >SERR- <PERR- INTx-
   Latency: 0, Cache Line Size: 32 bytes
   Interrupt: pin A routed to IRQ 56
   NUMA node: 0

| 上例中安装在0号物理CPU上。如果从1号物理CPU上的核去接收和发送，必须通过CPU之间的QPI总线，测试表明会带来1微左右的性能惩罚。如果CPU性能好，主频高，惩罚可能小一点。因此指定CPU编号时需确保调用ydAPI的线程运行在网卡直连的物理CPU的核上。可以通过lscpu获取这些编号信息。其次不能通过numactl来绑定。如果用这种方法绑定，调度程序会在这些CPU之间调度各个进程和线程，如果发生线程在CPU间的切换，性能的惩罚比上述网卡的还大，甚至达到毫秒的级别。应该直接指定CPU编号，而且这个CPU只指定给一个线程使用。第三，接收线程是使用忙等方式，其CPU使用率会达到100%，如果多个线程在同一个CPU核上，其性能会显著下降。特别是如果隔离了CPU，操作系统将不再为这个CPU核提供时间片调度服务，除非正在运行的线程主动放弃，否则其他线程无法获得CPU运行机会，会被饿死。第四，和ydAPI有关的线程尽可能绑定在同一个物理CPU的不同核上。

**如果启动时候用了onload,是否还需要在config改#RunPosition=OnloadLAN？**
| 本质上来说，RunPosition=OnloadLan代表一组参数，实际与onload没有直接关系。

TradingServerTimeout:10
TCPMarketDataTimeout:10
ReceiveUDPMarketData:yes
BlockedUDPMarketData:no
UDPTrading:yes
这些参数中最为关键的就是UDPTrading，因为UDP协议栈简单，理论上其延迟会好一些。如果删除RunPosition，手工将上述参数加上去就行。前面谈到不接收MD，因此TCPMarketDataTimeout:10，BlockedUDPMarketData:no就没有意义了，ReceiveUDPMarketData需要配置为no。

设置亲和性在TCPTradingCPUID设置了之后，运行的时候是不是还需要用taskset开启?比如亲和性设置了3号核，运行的时候还要运行onload -p latency taskset -c 3 ./[Application]
不需要的。taskset只能设置进程级别的CPU绑定，如果进程里面有多个线程（程序化交易大部分如此），操作系统调度程序会在指定的CPU核上调度运行，如果线程数多于CPU核数，则必须共享，即使运行在用户态的线程也可能发生线程切换的情况。线程切换的代价最少是几个微，如果资源紧张，甚至上毫秒。ydAPI调用pthread_setaffinity_np将线程绑定到CPU核上。如果该CPU核被操作系统隔离（/etc/default/grub中指定kernel启动参数isolcpus），那么绑定的线程就不会被其他线程/进程打扰，除了中断外不会发生线程切换。 

是不是尽量不要把程序运行在0号核上？
是的。绑定0#核是大忌。

下面摘自onload的文档，说的是如果开启了-p latency，就进入了spinning模式，这个时候是否是会导致需要额外的资源？
OpenOnload有两种模式。Spinning模式（busy-wait）中，加速进程的每个线程会占用一整个CPU core，始终处于waiting状态。通过htop可以看到该CPU的使用率为100%。spinning模式速度更快，但是要注意CPU core的数量。Interrupt模式中，线程不会占用满一个CPU core，但可以将中断Interrupt放在一个CPU core。interrupt模式也有加速效果，理论上比spinning略差一些。当服务器上总线程数大于CPU core的数量时，interrupt可能优于spinning，需要测试来论证。使用准备（适用于两种模式）Spinning模式使用spinning模式加速应用：onload -p latency taskset -c 3 ./[Application]onload –profile=latency taskset -c 2 ./[Application]-c 参数选择CPU core，也可以选择多个core：-c 2,3。选择core的数量与进程的线程数有关。Interrupt模式使用interrupt模式加速应用：onload ./[Application] 回答：对于HFT而言，CPU100%运行时正常的，也就是busy-wait。这样可以减少两端时间：一是操作系统将线程/进程唤醒的时间，一般需要500ns秒以上；二是该线程在cpu就绪队里排队等待运行的时间。上述两个时间最少大概在2us。因此，Barclays有个HFT曾经说CPU100%是进入HFT世界的基本配置。

成交回报中的tradetime是什么含义？
报单成交时间，是整数，表示从该交易日开始到成交时间的秒数。每个交易日从17点开始，即17:00:00对应时间戳0。周六凌晨的

如何优选前置
通过在YDInputOrder中设置ConnectionSelectionType和ConnectionID来设置指定方式和指定的前置连接。其中，ConnectionSelectionType有三个选项：

YD_CS_Any=0，不指定交易所席位连接。ydServer将根据现有席位连接情况尽快将报单请求或撤单请求送入交易所。
YD_CS_Fixed=1，指定交易所席位连接。如果该席位连接因流控或断链等原因不可用，ydServer将等待该席位可用后将请求送入交易所。
YD_CS_Prefered=2，指定交易所席位连接。如果该席位连接因流控或断链等原因不可用，ydServer将根据现有其他席位连接情况尽快将报单请求或撤单请求送入交易所。
ConnectionID是从0开始的序号，席位的总数可以通过YDExchange中的ConnectionCount获取

没有前置优选的情况下，配置多席位有什么意义？
席位会随机连接多个前置，可能会分布在不同的前置机上，即便是同一个前置机，他的实际使用体验也可能会不同。因此，即便没有前置优选功能，配置多个席位还是有意义的。投资者可以在下单时，选择某一个席位。

客户端有哪些线程？
使用ydApi的缺省配置文件会启动这些线程：

接收交易回报（也就是TCPTradingCPUID指定的线程），这个线程不能关闭，否则没有交易回报
接收TCP行情的，如果设置ConnectTCPMarketData=no则不创建
接受UDP行情的，如果ReceiveUDPMarketData=no则不则不创建
内部管理线程，主要用于发送心跳，建议设置UDPHeartBeatGap，值从10到50（单位毫秒）之间不等，这个线程无法关闭，但基本不吃CPU，唯一要求就是和客户的策略/发报单线程在同一个物理CPU上以共享Cache，从而实现UDP协议栈的预热。如果客户使用超频机，本身就只有一个物理CPU，就无所谓了。
启动系统时报 /sys/firmware/dmi/tables/smbios_entry_point: Permission denied 以及 /dev/mem: Permission denied 的错误怎么办？
按照监控中心要求，投资者登录柜台时需要收集硬件信息，上述错误是因为收集时调用了dmidecode而没有权限导致的。dmidecode需要root权限，有两种方式可以使得普通用户可以执行该程序：

suid模式：使用root用户增加dmidecode的s位，具体命令为chmod +s /usr/sbin/dmidecode，这是ydApi默认调用dmidecode的方式。
sudo模式：在ydApi的配置文件中增加ReportMethod=1，ydApi会使用sudo dmidecode的方式调用。
登录时遇到YD_ERROR_ClientReportError=35 客户报告错误怎么办？
该错误表明期货公司打开了严格检查穿透式监管验证的模式，即有任何穿透式监管需要的信息采集不全的不允许登录，包括IP地址、MAC地址、设备名、操作系统版本、硬盘序列号、CPU序列号、BIOS序列号。

为了方便投资者排查该问题，易达提供了客户端采集信息收集的小工具，投资者可以运行以下Linux和Windows程序来观察。

返回的格式为：终端类型@信息采集时间@私网IP1@私网IP2@网卡MAC地址1@网卡MAC地址2@设备名@操作系统版本@硬盘序列号@CPU序列号@BIOS序列号。通常会导致采集不全的是硬盘序列号、CPU序列号、BIOS序列号，如果发现信息有缺失的，可以使用下列命令来逐一检查排查。其中，windows的命令需要在PowerShell中执行。

Windwos的硬盘序列号：Get-WmiObject -Query “SELECT SerialNumber FROM Win32_PhysicalMedia”
Windows的CPU序列号：Get-WmiObject -Query “SELECT ProcessorID FROM Win32_Processor”
Windows的BIOS序列号：Get-WmiObject -Query “SELECT SerialNumber FROM Win32_BIOS”
Linux的硬盘序列号：首先通过/bin/lsblk -dn -o TYPE,NAME找到TYPE为disk的设备名NAME，然后调用/sbin/udevadm info –query=all –name=/dev/{NAME}获得该设备的序列号
Linux的CPU序列号：略
Linux的BIOS序列号：/usr/sbin/dmidecode -s system-serial-number
使用裸协议报单能比API报单快多少？
首先，裸协议报单是指用户自己负责报单、撤单的数据包的组装和发送，其性能直接取决于用户的实现，和易达API没有关系。裸协议报单和通过ydApi的UDP报单，在到达柜台后的穿透性能是没有区别的。因此，差异仅仅在于客户端发送端的性能。

其次，易达的API经过长时间的迭代，在我们实验室中（在X10/25和X2522网卡上）测得的性能已经令人满意，我们认为ydAPI发送性能已经能达到普通FPGA的水准。如果客户尝试自己实现裸协议报单，那么请在实现完成并与我们的API的报单性能进行比较后，选择较快的一种作为您生产报单的方式。

我们在新版本中增加了时间戳函数getYDNanoTimestamp，该函数精度高速度快，可以用来测试API的报单速度。具体方法为在insertOrder的前后调用getYDNanoTimestamp并将结果相减，得到的是从开始发送到报单的最后一个字节出现在光纤上的时间差。

我使用的是UDP报单还是TCP报单？
检查配置文件，UDPTrading：指示使用UDP或者TCP发送报单，yes表示使用UDP报单，no表示使用TCP报单。

易达有行情吗？
易达仅提供一档普通行情，目前仅可用于易达服务端和客户端用于计算保证金和盈亏。如在生产交易中需要行情，请联系期货公司接收组播行情。

为什么收不到易达行情？
在配置文件中检查如下配置项：

ConnectTCPMarketData：是否连接ydServer的TCP行情服务。yes表示接收，no表示不接收。
TCPMarketDataCPUID：设置TCP行情接收线程的亲和性。-1代表不需要设置亲和性；否则为CPU/Core的编号。
ReceiveUDPMarketData：是否接收ydServer发送的UDP组播行情。目前所有易达柜台的UDP组播行情功能均是关闭的，请始终保持该参数为no。

**如何获取易达的席位编号？**
客户端程序可使用YDAPI类的getExchange或者getExchangeByID方法获取YDExchange的指针，其ConnectionCount字段中标明了席位连接数量。注意，席位连接从0开始编号，依次递增，直至ConnectionCount-1为止。

**报单编号OrderRef是如何管理的？**
对于易达系统的报单和报价，所有返回的信息中都会填写客户提交时的OrderRef，易达不进行任何唯一性或者单调性检查；对于报价派生的报单，报单中的OrderRef会与报价中的相同；对于非易达系统的报单和报价，或者非本次易达服务器运行时报入的报单和报价，所有返回信息中的OrderRef一律是-1；使用YDClient给出的所有报单和报价，OrderRef一律是0。

**易达提供测试开发环境吗？**
我们提供互联网测试开发环境，具体参考文档《投资者互联网开发环境》.

**易达提供大商所盘中组合持仓？**
支持大商所盘中组合持仓， 可以通过下面几种方式：

- api接口autoCreateCombPosition，每调用一次这个函数，只会搜索一腿组合持仓，找到后向大商所发送组合指令；
- api接口insertCombPositionOrder，支持组合持仓或拆开组合持仓；
- api接口checkAndInsertCombPositionOrder，检查并报送组合持仓报单；
  
提供大商所组合保证金工具，可以从这里下载。