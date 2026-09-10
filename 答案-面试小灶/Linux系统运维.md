# Linux系统运维 — 面试参考答案

> 覆盖题单：Linux 运维热门面试题 100 道（103题）、Shell 脚本编程（23题）、网络运维（26题）、运维流程规范（15题）
> 格式说明：每题先列核心要点，再给一段可直接背诵的口语化回答。命令均为实际可用命令。

## Linux 运维热门面试题 100 道

### 1. Linux 内存中的 Buffer 与 Cache 有什么作用？

**要点：**
- buffer 是**写缓冲**：把零散的小块写入在内存里攒起来，合并后批量落盘，减少磁盘 IO 次数；主要针对磁盘块（block device）
- cache 是**读缓存**：把从磁盘读过的文件内容（页缓存 page cache）留在内存，下次命中直接走内存，不再读盘
- 两者都占用空闲内存，由内核自动管理：应用需要内存时自动回收，不浪费
- `free -h` 中 buff/cache 合并显示；看真正可用内存看 **available** 列，不是 free 列
- `sync` 手动把 buffer 刷盘；`echo 3 > /proc/sys/vm/drop_caches` 仅测试用

**一句话答：** buffer 解决"写得太碎"——零散写先攒在内存再批量落盘；cache 解决"重复读"——读过的文件放内存加速下次访问。两者都吃空闲内存但不是浪费，应用要用时内核自动释放，所以判断内存够不够要看 free 命令里的 available，而不是 free 列。

### 2. Linux 系统 CPU 持续飙高，如何排查？

**要点：**
- `top` 找到高 CPU 进程 → `top -Hp <pid>`（或 `pidstat -t -p <pid> 1`）定位到线程
- 线程号转十六进制：`printf '%x' <tid>`，Java 应用去 jstack 输出里对号；C/C++ 用 `perf top -p <pid>` 看热点函数
- 看 top 里 CPU 花在哪：**us** 高查应用代码（死循环、频繁 GC、正则回溯），**sy** 高查系统调用/锁竞争，**wa** 高转查磁盘 IO，**si** 高查网络软中断
- `strace -p <pid>` 看进程卡在哪个系统调用；陌生进程持续高 CPU 要排查挖矿木马（`lsof -p <pid>` 看连接、`crontab -l` 看定时任务）

**一句话答：** 三步走：top 定位进程，top -Hp 定位线程，线程号转 16 进制去 jstack 或 perf 里对号入座。同时看 CPU 的去向——us 高是应用代码问题，sy 高是系统调用或锁，wa 高其实是磁盘 IO 把 CPU 拖住了，方向别搞反。生产环境还要防一手挖矿木马。

### 3. Linux 服务器如何查看硬件信息？

**要点：**
- CPU：`lscpu`；内存：`free -h`、`dmidecode -t memory`（条数/频率/序列号）
- 磁盘：`lsblk`、`fdisk -l`；磁盘健康：`smartctl -a /dev/sda`
- 整机：`dmidecode`（型号/序列号）；PCI 设备：`lspci`；USB：`lsusb`
- 带外管理：`ipmitool sensor`（不进系统也能看温度硬件状态）

**一句话答：** 常用就五个：lscpu 看 CPU，free 和 dmidecode -t memory 看内存，lsblk 看磁盘布局，smartctl 看磁盘 SMART 健康，dmidecode 看整机型号和序列号。带外问题用 ipmitool。

### 4. 简述 Iptables 四表五链及其作用？

**要点：**
- **四表**（规则类别，优先级从高到低）：**raw**（不对包做连接跟踪）→ **mangle**（修改报文/TTL/标记）→ **nat**（地址转换）→ **filter**（过滤，最常用）
- **五链**（规则挂载点）：**PREROUTING**（路由判断前）、**INPUT**（进本机）、**FORWARD**（经本机转发）、**OUTPUT**（本机发出）、**POSTROUTING**（路由判断后）
- 组合规律：filter 表挂在 INPUT/FORWARD/OUTPUT；nat 表挂在 PREROUTING/OUTPUT/POSTROUTING
- 报文走向：进本机的走 PREROUTING→INPUT；转发的走 PREROUTING→FORWARD→POSTROUTING

**一句话答：** 表是"规则干什么"，链是"规则挂在路径哪个点"。四表按优先级 raw > mangle > nat > filter，日常 90% 用的都是 filter 表；五链对应报文的五个途经点，比如发布内网服务（DNAT）挂在 PREROUTING，内网上网（SNAT/MASQUERADE）挂在 POSTROUTING。

### 5. 简述 RAID0、RAID1、RAID5 和 RAID10 的工作原理及其特点？

**要点：**
- **RAID0** 条带：数据分散写在多块盘，读写最快、容量 100% 利用，但**无冗余**，坏一块全丢；至少 2 块
- **RAID1** 镜像：两两互拷，可坏一半（每组镜像坏 1 块），容量利用率 50%，读性能好；至少 2 块
- **RAID5** 条带+分布式校验：允许坏 1 块，容量 (n-1)，写有校验计算开销，坏盘重建时压力大、期间性能下降明显；至少 3 块
- **RAID10** 先镜像再条带：兼得性能与冗余，每组镜像可各坏 1 块，利用率 50%，成本高；至少 4 块，数据库/高 IO 场景首选
- 对比记忆：追求性能 RAID0、追求省钱 RAID5、追求性能+可靠 RAID10

**一句话答：** RAID0 把数据拆开并行写，快但一块盘坏全部完蛋；RAID1 两块盘互为镜像，利用率一半；RAID5 用分布式校验盘允许坏一块，容量和成本折中但写性能和重建期表现一般；RAID10 先镜像后条带，性能和可靠性最好，利用率 50%，数据库这类重要系统都用它。

### 6. 提示磁盘空间已满，该如何解决？

**要点：**
- 先分清三种"满"：
  1. **block 满**：`df -h` 确认 → `du -sh /* 2>/dev/null | sort -rh` 逐层下钻找大目录/大文件（常见：日志、coredump、临时文件）
  2. **inode 满**：磁盘有空间但写不进 → `df -i` 确认，海量小文件占满 inode（常见：session 文件、缓存碎片、邮件队列）
  3. **已删除未释放**：文件 rm 了空间不还 → `lsof +L1` 或 `lsof | grep deleted`，进程还握着文件句柄；处理：重启对应进程，或 `: > /path/to/log` 清空（truncate）而不是 rm
- 清理正在写入的日志用 `truncate -s 0` 或 `: > file`，**不要直接 rm**（rm 后空间不释放且日志句柄失效）

**一句话答：** 先 df -h 和 df -i 分清是 block 满还是 inode 满：block 满 du 逐层找大文件；inode 满说明小文件泛滥；还有一种隐蔽情况是文件删了但进程还占着句柄、空间不释放，用 lsof +L1 找到进程重启它，或者用 truncate 清空日志——千万别对正在写的日志直接 rm。

### 7. 简述 DNS 解析过程？

**要点：**
- 顺序：浏览器/应用缓存 → 操作系统缓存 → **hosts 文件** → 本地 DNS 服务器（运营商或自建）
- 本地 DNS 没有缓存时，替客户端做**递归**查询：问根域 → 根返回 .com 顶级域地址 → 问 .com → 返回权威服务器地址 → 问权威 → 拿到最终 IP
- 本地 DNS 到各级服务器之间是**迭代**查询；客户端到本地 DNS 是递归查询
- 结果按 TTL 缓存；验证工具：`dig +trace www.example.com`、`nslookup`

**一句话答：** 客户端先查本机缓存和 hosts，没有就把域名丢给本地 DNS 服务器；本地 DNS 递归替你跑腿——先问根服务器要 .com 的地址，再问 .com 要权威服务器地址，最后问权威拿到 IP，一路缓存下来按 TTL 保存后返回。dig +trace 可以把这条链完整复现出来。

### 8. 简述 Linux 系统启动过程？

**要点：**
- **BIOS/UEFI** 上电自检（POST），定位启动设备
- **GRUB2** 引导：Stage1 在 MBR（或 UEFI 直接读 ESP 分区），加载 grub.cfg 选内核
- 加载**内核 vmlinuz + initramfs**（临时根文件系统，加载存储/文件系统驱动）
- 内核初始化硬件、挂载真实根分区，启动 **PID 1 = systemd**
- systemd 按 **default.target**（一般 multi-user.target 或 graphical.target）拉起服务，最后跑 getty/gdm 提供登录

**一句话答：** 固件自检（BIOS/UEFI）→ GRUB 引导选内核 → 内核带着 initramfs 起来、挂载真正的根分区 → 启动一号进程 systemd → systemd 按 default.target 依次拉起服务，最后出登录界面。排障时每一级对应一类问题：引导坏了查 GRUB，服务起不来查 systemd 和 journalctl。

### 9. 请说一下你经常用的 Linux 系统性能分析工具及作用？

**要点：**
- 总入口：`top` / `htop`（进程级全景）、`uptime`（load）
- CPU：`mpstat -P ALL 1`（每核分布）、`pidstat -u 1`（进程级）
- 内存：`free -h`、`vmstat 1`（si/so 看换页）、`pidstat -r 1`
- 磁盘：`iostat -x 1`（%util、await）、`pidstat -d 1`
- 网络：`ss -s`、`sar -n DEV 1`、`tcpdump`
- 历史/持续观测：`sar`（sysstat 全家桶，带历史数据）

**一句话答：** 我的习惯是 top 起手分流——先看是 CPU、内存还是 IO 的问题，再下钻：CPU 用 mpstat 和 pidstat，内存用 free 和 vmstat 看 si/so，磁盘用 iostat -x 看 %util 和 await，网络用 ss 和 sar -n。需要看历史趋势就用 sysstat 的 sar，它的价值是能回看过去每小时的曲线。

### 10. 简述 FTP 工作模式？

**要点：**
- FTP 用两条连接：**控制连接**（21 端口，全程保持）+ **数据连接**（传输文件时建立）
- **主动模式 PORT**：客户端告诉服务端自己的端口，服务端从 **20 端口**主动连客户端建数据连接 → 客户端在 NAT/防火墙后时连不通
- **被动模式 PASV**：服务端开一个随机高位端口，**客户端**主动去连 → 现代网络默认模式，服务端防火墙要放行被动端口范围
- 安全场景用 sftp（SSH 子系统）或 ftps 替代明文 FTP

**一句话答：** FTP 控制走 21 端口，数据另开连接，区别在数据连接谁主动：主动模式服务端拿 20 端口反向连客户端，客户端躲在 NAT 后就废了；被动模式反过来，客户端连服务端开的高位端口，所以实际生产基本都用被动模式。现在传输文件更推荐直接走 sftp。

### 11. 简述 ext4 日志文件系统原理？

**要点：**
- 核心思想：正式写入前，先把本次操作记录写进**日志区（journal）**，崩溃重启后**重放日志**恢复到一致状态，不用全盘 fsck
- 三种模式（mount 选项 data=）：
  - `journal`：元数据+数据都记日志，最安全最慢
  - `ordered`（**默认**）：只记元数据，数据先于元数据落盘
  - `writeback`：只记元数据，数据落盘顺序不保证，最快风险最高
- 写流程：写日志 → 提交（checkpoint）→ 实际写入

**一句话答：** 原理就是"先记账再干活"：文件系统改动先写进日志区，机器断电崩溃后重启时按日志重放，把文件系统恢复到一致状态，避免传统文件系统崩溃后跑几小时 fsck。ext4 默认 data=ordered，只把元数据记日志，平衡了安全和性能。

### 12. Linux 进程有哪几种状态？

**要点：**
- **R** Running：正在 CPU 上跑或在运行队列排队
- **S** Sleeping（可中断）：等事件，收到信号能唤醒
- **D** 不可中断睡眠：通常在等磁盘/存储 IO，**kill -9 都杀不掉**，大量 D 状态 = 存储出问题了
- **T** Stopped：被 SIGSTOP/作业控制暂停
- **Z** Zombie 僵尸：已退出但父进程还没 wait 回收
- 查看：`ps aux` 的 STAT 列；D 状态排查看 iostat 和存储链路

**一句话答：** 常见五种：R 在跑、S 在等（可被信号打断）、D 在等 IO（不可中断，kill -9 也杀不掉，看到大量 D 就该查磁盘了）、T 被暂停、Z 僵尸。面试重点提 D 和 Z：D 是存储卡住的信号，Z 是父进程没回收。

### 13. Linux 进程间有哪些通信方式？

**要点：**
- **管道**（`|`）与命名管道 FIFO：父子进程/同主机，最简单
- **信号** signal：事件通知（kill 本质就是发信号）
- **消息队列**：内核维护的消息链表，按类型读取
- **共享内存**（shm/mmap）：最快的方式，多个进程映射同一块物理内存，需配**信号量**做同步
- **socket**：唯一支持跨主机的本地/网络通用方式（Unix domain socket 用于本机，如 docker.sock）
- **内存映射文件** mmap

**一句话答：** 常见六种：管道、信号、消息队列、共享内存、信号量、socket。要点是共享内存最快但要配信号量防竞争，socket 是唯一能跨机器的。管道和信号算进程间通信的"日用品"，systemd 和容器时代最常见的是 Unix socket。

### 14. Linux 服务器如何调优？

**要点：**
- 原则：**先测量、后调优、再验证**，盲调参数是反模式
- 通用项：文件句柄 `ulimit -n` / `fs.file-max`；端口范围 `net.ipv4.ip_local_port_range`；TIME_WAIT 复用 `net.ipv4.tcp_tw_reuse=1`；连接队列 `net.core.somaxconn`；换页倾向 `vm.swappiness=10`
- CPU/调度：绑核（taskset）、调大线程池而非加机器之前先看瓶颈
- IO：换 SSD、noatime 挂载、IO 调度器选型（SSD 用 none）
- 内存：大页（THP 视应用而定）、数据库关 swap 或调低 swappiness
- 每项改动前后用压测和监控数据对比，写进变更记录

**一句话答：** 我的调优思路是先测后调：用 top/iostat/sar 找到真实瓶颈再动手，不背参数清单。常用抓手就几类——句柄和端口范围、tcp_tw_reuse、somaxconn、swappiness，每一项都对应一个具体症状。改完必须压测对比验证，并记录变更，不然调优就成玄学了。

### 15. 简述 Keepalived 工作原理？

**要点：**
- 基于 **VRRP**（虚拟路由冗余协议，IP 协议号 112）：多台机器组成一个虚拟组，对外提供同一个 **VIP**
- 按配置的 **priority** 选 MASTER，MASTER 周期组播 VRRP 通告（默认 1s，组播地址 224.0.0.18）
- BACKUP 超时收不到通告 → 竞争成为新 MASTER，接管 VIP（ gratuitous ARP 通告 MAC 变化）
- Keepalived = VRRP + **健康检查**（check 对后端探测，失败时降权或切换）+ LVS 管理（ipvs 规则自动生成）
- 典型用法：两台 Nginx/HAProxy 前端 + Keepalived VIP 实现入口高可用

**一句话答：** 多台机器共享一个虚拟 IP，按优先级选一台 MASTER，它每秒组播一次 VRRP 心跳；其余是 BACKUP，心跳超时就抢过来接管 VIP 并发免费 ARP 更新 ARP 表。Keepalived 在 VRRP 之上加了健康检查，所以能配合 Nginx 或 LVS 做真正的入口高可用。

### 16. LVM 解决了什么问题？有什么特点？

**要点：**
- 三层抽象：**PV**（物理卷，初始化的磁盘/分区）→ **VG**（卷组，PV 的资源池）→ **LV**（逻辑卷，从 VG 划出给文件系统用）
- 解决：传统分区划完就死板——空间不够只能挪数据重新分区；LVM 可**在线扩容**：`lvextend -L +10G` + `xfs_growfs`（xfs）或 `resize2fs`（ext4）
- 特点：支持**快照**（lvsnap，备份利器）、跨多块盘组大空间、条带/镜像
- 代价：多一层抽象，坏一块 PV 影响其上 LV（要配 RAID 底座），引导分区一般不放 LVM

**一句话答：** LVM 在磁盘和文件系统之间加了一层可调度的池子：PV 组成 VG，LV 从 VG 里划。核心价值是分区不再是死的——空间不够 lvextend 在线扩，还能做快照做备份。生产上注意它不管冗余，底下还是要垫 RAID。

### 17. 有500台服务器，你该如何管理？

**要点：**
- **标准化**是前提：统一 OS 镜像、初始化脚本（Hostname/时间/源/内核参数）、目录规范
- **自动化**：Ansible（免 agent）批量下发配置和变更，变更走 Git 版本管理（GitOps 思路）
- **可观测**：Prometheus + Grafana 统一监控告警，日志集中 ELK/Loki，不逐台 ssh 查
- **资产与权限**：CMDB 管资产，堡垒机统一入口，权限最小化+操作审计
- **流程**：变更走工单/审批，重大变更灰度+回滚方案；容量规划定期review

**一句话答：** 500 台的核心思路是"把人从重复劳动里拿出来"：镜像和初始化标准化打底，Ansible 批量自动化，Prometheus 和 ELK 集中监控日志，资产进 CMDB、操作走堡垒机留审计。人只处理异常和设计，不逐台登录敲命令。

### 18. /proc 目录有什么作用？

**要点：**
- 内核暴露的**伪文件系统**（内存中，不占磁盘），实时反映内核状态
- `/proc/<pid>/`：每进程信息（cmdline、status、fd、maps）；`/proc/<pid>/fd` 可找回被删文件的句柄
- 系统信息：`/proc/cpuinfo`、`/proc/meminfo`、`/proc/loadavg`、`/proc/net/`（tcp、conntrack）
- 内核参数：`/proc/sys/` 下的文件即 sysctl 参数，如 `/proc/sys/net/ipv4/ip_forward`（`echo 1 >` 开启转发）
- 类似的还有 /sys（sysfs，设备和驱动）

**一句话答：** /proc 是内核开给用户的"实时状态窗口"，全是内存里的伪文件：cat /proc/meminfo 看内存，/proc/数字目录看每个进程，/proc/sys 下就是 sysctl 参数，echo 1 > /proc/sys/net/ipv4/ip_forward 就能开路由转发。它是 top、free 这些工具的数据源。

### 19. top 中的VIRT、RES和SHR分别是什么意思？

**要点：**
- **VIRT**：进程申请的**虚拟地址空间总量**，包含未实际分配、映射文件、内核空间——通常很大，不是问题指标
- **RES**：实际占用的**物理内存**（常驻），排查内存问题主要看这个
- **SHR**：RES 中可与其他进程**共享**的部分（共享库 so、共享内存段）
- 判断内存泄漏看 RES 持续上涨；Java 进程 VIRT 巨大属正常（JVM 预留地址空间）

**一句话答：** VIRT 是申请的虚拟内存总量，看着吓人但不是真占用；RES 才是真实吃掉的物理内存，排查问题看它；SHR 是其中和别的进程共享的部分，比如 so 库只算一份。所以 Java 进程 VIRT 几十个 G 很正常，别被它吓到。

### 20. 你在以往工作中，有遇到过什么深刻的故障吗？

**要点：**（面试话术模板，按真实经历替换）
- 结构：**背景 → 现象 → 定位过程 → 止血 → 根因 → 复盘改进**
- 定位过程要体现方法论：监控/日志缩小范围 → 二分排查 → 找到第一现场（而不是猜）
- 止血和根因分开讲：先恢复业务（回滚/切流），再慢慢查根因
- 复盘要落到机制改进（加监控、改流程、自动化），而不是"下次注意"
- 示例方向：fstab 写错导致启动失败、日志把盘写满、变更未灰度引发雪崩、DNS 故障、证书过期

**一句话答：**（示范框架）讲一个真实故障：比如一次日志分区写满导致服务不可写——现象是接口 500，先 lsof +L1 定位到删除未释放的日志文件并 truncate 止血，根因是 logrotate 配置错误没轮转，改进是把磁盘水位告警提前到 80% 并用 logrotate 校验脚本防止同类问题。重点展示"先止血再根因再机制"的思路。

### 21. 简述你对内核空间和用户空间的理解？

**要点：**
- 虚拟地址空间被划分为**内核空间**（高地址段）和**用户空间**（低地址段），CPU 分 ring0/ring3 两级特权
- 用户态代码不能直接碰硬件/内存/网卡，必须通过**系统调用**（syscall）陷入内核态，由内核代劳
- 目的：**隔离保护**——应用崩溃不拖垮系统，恶意代码被挡在内核外
- 切换有开销（上下文保存、TLB 等），高频 syscall 是性能点之一（io_uring/DPDK 就是为了减少这个）
- top 里 %sy 是内核态时间占比，%us 是用户态

**一句话答：** 系统把虚拟内存劈成两半：应用跑在用户空间、权限受限，要操作硬件必须经系统调用陷入内核空间，由内核代劳。好处是隔离——应用崩了系统不崩。代价是每次切换有开销，所以高并发场景才有人搞 io_uring 这类减少系统调用的技术。

### 22. 现在要上线一个网站，你会如何设计高可用、高并发的架构？

**要点：**
- **入口**：DNS 智能解析/多运营商线路 + CDN 静态资源分流
- **接入层**：LVS（四层扛海量连接）+ Keepalived VIP，后面挂 Nginx 集群（七层/缓存/限流），Nginx 到后端用 keepalive 长连接
- **应用层**：无状态设计（session 放 Redis），可水平扩容；限流（Nginx limit_req / 网关）、熔断降级
- **数据层**：Redis 集群做缓存（缓存穿透/雪崩要有对策），MySQL 一主多从+读写分离，数据量大再分库分表
- **异步**：MQ 削峰填谷，耗时操作异步化
- **兜底**：全链路监控告警、预案（切流/降级/扩容）、容量压测先行

**一句话答：** 从外到内四层套路：CDN 和智能 DNS 把流量先分掉，接入层 LVS+Keepalived 扛连接、Nginx 做七层和限流，应用层坚持无状态好水平扩，状态和压力往后推——热数据进 Redis，持久层主从读写分离，洪峰用 MQ 削峰。配套做好压测、全链路监控和降级预案，高可用不是堆组件而是预案演练出来的。

### 23. Linux 系统出现大量 TIME_WAIT，是什么原因？

**要点：**
- TIME_WAIT 是**主动关闭方**在发出最后一个 ACK 后进入的状态，停留 **2MSL**（Linux 默认每方向 60s，共 60s 计时后销毁，注意单侧 60s）
- 大量 TIME_WAIT = **本机高频主动关闭短连接**：典型如 Nginx 反代后端用了短连接（upstream 没配 keepalive）、php-fpm 连 MySQL 不用连接池、脚本频繁 curl
- 危害：占满**本地端口**（出方向连接 `net.ipv4.ip_local_port_range` 默认约 3 万个）→ `Cannot assign requested address`；内存开销
- **根治**：改长连接——HTTP keepalive、`upstream { keepalive N; }`、数据库连接池
- **缓解**：`net.ipv4.tcp_tw_reuse=1`（配合时间戳，仅对**主动发起**方向有效）、扩大 `ip_local_port_range`
- `net.ipv4.tcp_tw_recycle` 已在内核 4.12 **移除**，NAT 环境会出问题，不要提"开它"当方案

**一句话答：** 大量 TIME_WAIT 说明本机在疯狂主动关闭短连接——最常见是 Nginx 到 upstream、应用到数据库之间没用长连接。它停 60 秒等两个目的：保证最后的 ACK 丢了能重传、让旧报文自然消亡。根治是上长连接和连接池，缓解是开 tcp_tw_reuse 加大端口范围；tcp_tw_recycle 已被内核移除，别再当答案。

### 24. Linux中 inode 和 block 分别是什么？

**要点：**
- **inode**：文件的"身份证"——权限、属主、大小、时间戳、数据块指针，**不含文件名**；文件名是目录项里 inode 号的映射
- **block**：实际存数据的空间单元（ext4 默认 4K）
- inode 总数在格式化时定死 → 会出现 **df -h 有空间但 df -i 满**写不进文件（海量小文件）
- 硬链接 = 多个目录项指向同一 inode（`ls -i` 可见链接数）；删除文件 = inode 链接数减到 0 且无进程占用才真正释放

**一句话答：** 文件分两部分存：inode 存元数据和数据块指针，block 存内容。关键点有三个：文件名不在 inode 里、inode 数量格式化时定死所以会"有空间却写不进"、删除文件要等链接数为零且无进程占用才真正腾出空间。

### 25. 你怎么理解系统负载（load average）？

**要点：**
- 定义：**1/5/15 分钟**内，**可运行状态（R）+ 不可中断睡眠（D）**进程数的指数加权平均（Linux 特有：把 D 算进去，其他 Unix 只算 R）
- 判断：和 **CPU 核数**比——4 核机器 load 4 是满载，8 就是排队严重；同时看趋势（15min 值 > 1min 值说明在恶化）
- **load 高 ≠ CPU 忙**：CPU 空闲但 load 高，说明有大量 D 状态进程卡在存储 IO 上（NFS 掉线、磁盘故障的典型症状）
- 配套命令：`uptime`、`vmstat 1`（r 列=运行队列，b 列=阻塞）、`iostat -x 1`、`ps aux | awk '$8~/D/'` 抓 D 进程

**一句话答：** load 本质是"排队干活的人数"：正在跑的加上排队的，Linux 里还把卡在 IO 的 D 状态进程算进去。数值要和核数对照：4 核 load 8 就是超载一倍。关键技巧是 load 高但 CPU 空闲时别去加 CPU——那是存储卡住了，查 iostat 和 D 状态进程才是正解。

### 26. 如何提升运维工作效率？

**要点：**
- **脚本化**：出现第 3 次的重复操作就写成脚本（Shell/Python）
- **自动化**：Ansible 批量变更、CI/CD 流水线、巡检自动化替代人工巡检
- **标准化**：镜像、目录、命名、文档模板统一，减少沟通和出错成本
- **工具与数据**：监控告警前置发现问题而不是等用户报障；运维知识库沉淀 SOP
- **流程**：变更模板化+审批，减少救火时间；度量指标（MTTR、变更成功率）驱动改进

**一句话答：** 我的思路是分层消除重复：单机重复写脚本，多机重复上 Ansible，高频人工判断做成自动化巡检和自愈。同时把操作沉淀成文档和 SOP，让效率提升可持续——判断标准很简单：同一种操作我做第二次就会想"它值不值得自动化"。

### 27. 如图所示，如何实现用户A访问用户B？

**要点：**
- （原题为拓扑图，此处给通用分析框架）先看 A、B 是否同网段：
  - **同网段**：直接二层互通，靠 ARP 找 MAC
  - **跨网段**：需要**网关/路由**，A 把包发给网关，网关查路由表转发
- 若中间隔 NAT：SNAT 改源地址上网、DNAT/端口映射让外部访问进来
- 若是不同 VPN/内网：走 VPN 隧道或代理（正向代理对客户端保密目的地，反向代理对服务端暴露入口）
- 回答思路：报文从 A 出发 → 网关逐跳路由 → 到达 B；排查用 `ping`/`traceroute`/`tcpdump` 三件套

**一句话答：** 这类题核心看两点：同网段靠 ARP 直接通；跨网段必须过网关逐跳路由。如果中间有 NAT 就讲 SNAT/DNAT 改地址，有防火墙就讲放行策略，最后落一句"实际排查我会 ping 和 traceroute 定位断在哪一跳"。

### 28. 运维工程师如何保障数据安全？

**要点：**
- **备份**：3-2-1 原则（3 份数据、2 种介质、1 份异地）；定期**恢复演练**——没验证过的备份等于没备份
- **权限**：最小权限原则，sudo 细粒度授权，堡垒机+审计，离职即回收
- **传输与存储**：传输走 SSH/TLS，敏感数据（密码/密钥）用 Vault/KMS 管理，落库加密，禁止明文备份
- **防泄漏**：生产数据脱敏后才进测试环境；数据库审计；DLP 敏感操作告警
- **抗破坏**：防勒索（备份离线/不可变存储）、操作前快照、高危命令（rm -rf、drop）走双人复核

**一句话答：** 我把它拆成四层：备份上遵守 3-2-1 并定期做恢复演练，权限上最小化加堡垒机审计，传输存储全程加密、密钥进 Vault，流程上高危操作复核、测试数据脱敏。核心认知是：备份的价值靠恢复演练证明，安全的价值靠"事发时挡住了"证明。

### 29. 什么是僵尸进程？怎么产生、怎么处理？

**要点：**
- 子进程先退出，父进程**没有调用 wait()/waitpid() 收尸**，子进程的进程表项残留 → Z 状态
- 危害：不占内存/CPU 但**占 PID**，大量堆积会耗尽 PID（`kernel.pid_max`）无法创建新进程
- 定位：`ps aux | awk '$8~/Z/'`；`ps -o ppid= -p <zombie_pid>` 找到**父进程**
- 处理：让父进程正常退出（kill 父进程，僵尸被 init/systemd 收养后回收）；根治是改代码（wait 回收或 signal(SIGCHLD, SIG_IGN)）
- 预防：容器/PID 1 场景注意 init 进程要能回收僵尸（tini/dumb-init）

**一句话答：** 子进程死了父进程没 wait 收尸，就成僵尸。它不占资源但占 PID，泛滥了系统就开不了新进程。杀僵尸本身没用，要处理父进程：kill 父进程让 systemd 收养回收，根治得改代码。排查时先 ps 抓 Z，再顺着 ppid 找到罪魁父进程。

### 30. iptables 的 SNAT 和 DNAT 分别用在什么场景？

**要点：**
- **SNAT** 改**源**地址：内网多台机器共享一个公网 IP **上网**；动作 `--to-source`，挂在 **POSTROUTING**；内网机器伪装出口用 MASQUERADE（出口 IP 动态时用）
- **DNAT** 改**目的**地址：把公网 IP:端口**映射到内网服务**（发布内网 Web/SSH）；动作 `--to-destination`，挂在 **PREROUTING**
- DNAT 在路由判断前改目的地址，所以包能被正确路由到内网；SNAT 在出网关前改源地址保证回包走同一路径
- 记忆：改"谁发的"是 SNAT，改"发给谁"是 DNAT

**一句话答：** SNAT 改源 IP，典型场景是公司内网机器共享公网出口上网，挂在 POSTROUTING；DNAT 改目的 IP 和端口，典型场景是把公网请求映射进内网的 Web 服务器，挂在 PREROUTING。一句话记：改"谁发的"用 SNAT，改"发给谁"用 DNAT。

### 31. ext4 与 xfs 文件系统有什么区别？

**要点：**
- **xfs**：高并发大文件 IO 表现好，支持在线扩容**只能扩不能缩**；RHEL7/CentOS7 起的默认文件系统
- **ext4**：成熟稳定，可以扩大也可以缩小（缩需离线），海量小文件场景历来口碑好
- 恢复工具不同：ext4 用 extundelete/debugfs，xfs 用 xfs_undelete；配 LVM 快照是更稳的兜底
- inode 处理：xfs 是动态分配 inode，ext4 格式化时固定 inode 数量

**一句话答：** 都是一线文件系统：xfs 大文件和高并发 IO 强，只能在线扩不能缩，是 CentOS7 后的默认；ext4 更老牌，支持缩容，小文件场景表现好。日常选型跟着发行版默认走 xfs 就对，需要缩容的少数场景才考虑 ext4。

### 32. 什么是 Swap？什么时候会用？

**要点：**
- Swap 是磁盘上的交换分区/文件，物理内存不足时把**不活跃的匿名页**（堆、栈）换出去腾地方
- 触发时机：内存紧张、内核回收策略控制（`vm.swappiness` 0-100，默认 60，值越大越倾向换出）
- 查看：`free -h`、`swapon -s`、`vmstat 1` 的 **si/so** 列（持续非零=在换页，系统卡顿常见原因）
- 生产经验：数据库/延迟敏感服务调低 swappiness（10 甚至 1）；服务器一般配少量 swap 兜底防 OOM；云上也有完全不配的流派
- swap 文件：`dd`/`fallocate` 创建 → `mkswap` → `swapon` → 写 fstab

**一句话答：** Swap 是拿磁盘当内存的溢出区，内存不够时内核按 swappiness 倾向把不活跃页面换出去。什么时候用：作为内存的保险丝防止 OOM 直接杀进程；但换页本身很慢，所以数据库这类服务会把 swappiness 调到 10 以下，vmstat 里 si/so 持续跳动就要警惕内存瓶颈。

### 33. SSH 连接比较慢是什么问题？

**要点：**
- 最常见：**反向 DNS 解析超时** → `/etc/ssh/sshd_config` 设 `UseDNS no`
- **GSSAPI 认证超时** → `GSSAPIAuthentication no`
- 排查顺序：`ssh -v` 看卡在哪一步；卡 "Connecting" 查网络/防火墙，卡在认证前大概率是 DNS/GSSAPI
- 其他：DNS 服务器本身慢（改 /etc/resolv.conf）、密码认证慢（换密钥）、systemd-logind 卡（`systemctl status systemd-logind`）

**一句话答：** 十有八九是 sshd 在做反向 DNS 或 GSSAPI 认证白等超时——sshd_config 里关掉 UseDNS 和 GSSAPIAuthentication 立竿见影。用 ssh -v 看卡在哪一步就能区分：连不上是网络问题，连上后卡半天才出密码提示就是这两个配置。

### 34. 执行 mount 或 df 命令时卡住无响应，可能是什么问题？

**要点：**
- 典型原因：某个 **NFS/网络存储挂载点已经失联**（服务端挂了、网络断了），df 遍历所有挂载点时在死等那个失联的
- D 状态验证：另开终端 `ps aux | grep df`，STAT 是 **D**（不可中断 IO 等待）
- 解法：`umount -l /mnt/xxx`（**lazy 卸载**，不等 IO）或 `umount -f`；根治修网络/存储端
- 预防：NFS 挂载用 `hard`/`soft` + `timeo`、`intr` 等超时参数，或 systemd automount 按需挂载

**一句话答：** 大概率是有个 NFS 之类的网络挂载点失联了，df 要遍历全部挂载点，走到那里就卡死。验证是 ps 看 df 进程是不是 D 状态，解法是 umount -l 懒卸载把失联的挂载点摘掉，df 立刻恢复。NFS 挂载记得带超时参数防复发。

### 35. conntrack 表满会导致什么问题？如何解决？

**要点：**
- conntrack 是内核连接跟踪表（NAT/有状态防火墙依赖它），每条连接占一项，有上限 `nf_conntrack_max`
- 打满现象：`table full, dropping packet` 刷 dmesg，**新建连接被丢**（ping 通但新 TCP 建不了），NAT 网关/K8s 节点高并发时多发
- 排查：`cat /proc/sys/net/netfilter/nf_conntrack_count` vs `nf_conntrack_max`；`conntrack -L | awk '{print $7}' | sort | uniq -c | sort -rn` 看谁占的
- 解决：调大 max（同时注意 hashsize 和内存）、调短超时（`nf_conntrack_tcp_timeout_established` 默认 5 天！）、优化业务让不需要跟踪的流量进 raw 表 NOTRACK

**一句话答：** conntrack 表记着每一条连接的状态，打满后内核直接丢新包，表现为 ping 得通但新连接建不起来。解决三招：调大 nf_conntrack_max、把 established 超时从默认 5 天调短让表快点释放、对纯转发不需要跟踪的流量用 NOTRACK 跳过记录。

### 36. TCP 连接数暴涨，如何定位是正常业务、还是攻击？

**要点：**
- 先画像：`ss -ant state established | awk '{print $4}' | cut -d: -f1 | sort | uniq -c`（按本地端口/服务分组）、按来源 IP 统计 top、按连接状态分布（SYN_RECV 洪水？ESTABLISHED 空闲连接？TIME_WAIT 短连接风暴？）
- **正常业务**特征：来源 IP 分散且符合用户画像、连接落在业务端口、有正常的收发数据（`ss -ti` 看字节计数）
- **攻击**特征：单一/少量来源海量连接、SYN_RECV 堆积（SYN Flood）、连接建立后无数据、来源是机房产段或海外 VPS、伴随带宽/CPU 异常
- 结合业务：和 QPS/在线用户数曲线对照，业务没涨连接暴涨就可疑
- 处置：确认攻击后防火墙/云清洗/限流，保留 pcap 证据（tcpdump -w）

**一句话答：** 我会先给连接画三张像：按来源 IP、按状态、按端口分布，再和业务曲线对照。正常业务连接数和 QPS 同步涨、来源分散、连接上有真实数据传输；攻击则是来源集中、SYN_RECV 异常堆积或者建了连接不发数据。定位是攻击就先限流和封禁止血，同时 tcpdump 留证据。

### 37. iowait 高会导致什么问题？如何解决？

**要点：**
- iowait 是 CPU 空闲但有未完成磁盘 IO 请求的时间占比——**不是 CPU 忙，是 CPU 在等磁盘**
- 危害：进程 D 状态堆积 → load 飙高（Linux 把 D 算进 load）、请求延迟上升、严重时服务超时雪崩
- 定位：`iostat -x 1` 看 %util 和 await；`iotop` 或 `pidstat -d 1` 找出高 IO 进程；`cat /proc/<pid>/io` 看进程读写量
- 解决：优化 IO 模式（日志异步/批量写、加 buffer）、换 SSD/NVMe、读写分离到不同盘、调大缓存、必要时限流保护后端存储

**一句话答：** iowait 高说明 CPU 没活干纯在等磁盘，表现为响应慢、D 状态进程多、load 虚高。排查用 iostat -x 看哪块盘 %util 打满，iotop 找出元凶进程；解决要么优化应用的写盘方式，要么升级存储。注意别看到 iowait 高就去升 CPU——那解决不了问题。

### 38. 数据库服务器 load 100+，能直接重启嘛？

**要点：**
- **不能先重启**：重启销毁现场（内存数据、锁状态、连接信息），根因没找到大概率起回来又复现，且重启数据库有数据风险（crash recovery 时间不可控）
- 先做无损诊断：`uptime`/`vmstat 1` 看负载构成（r 高=CPU 排队，b 高=IO 阻塞）；`iostat -x` 看磁盘是否打满；数据库内部看慢查询/当前执行 SQL/锁等待（如 MySQL `show processlist`）
- 止血优先级：kill 慢查询/大事务 → 限流应用连接 → 切流到从库/备用 → 以上无效且确认内存爆了（OOM 边缘）才考虑受控重启
- 受控重启：先停应用写入 → 数据库正常 shutdown（不是 kill -9）→ 期间保持只读

**一句话答：** 不先重启。load 100+ 先花两分钟看构成：r 队列高是 CPU，b 阻塞高是 IO，再看数据库慢查询和锁——多半一条烂 SQL 就能解释。止血顺序是杀慢查询、应用限流、切流，重启是最后手段，因为既丢现场又不解决根因，还可能触发更长的 crash recovery。

### 39. Linux 各目录有什么作用？

**要点：**
- `/` 根；`/bin` `/sbin` 基础命令（现在多合并到 /usr）；`/boot` 内核与 GRUB；`/dev` 设备文件；`/etc` 配置文件
- `/home` 用户目录；`/root` root 家目录；`/lib` 库；`/media` `/mnt` 挂载点；`/opt` 第三方软件
- `/proc` `/sys` 内核伪文件系统；`/tmp` 临时文件（重启可能清）；`/usr` 应用与资源；`/var` 可变数据（**日志 /var/log、邮件、缓存**——磁盘满高发区）
- 布局标准：FHS（Filesystem Hierarchy Standard）

**一句话答：** 面试抓大放小：/etc 管配置、/var 放日志等可变数据（磁盘满最常出事的地方）、/home 用户数据、/proc 和 /sys 是内核状态窗口、/dev 设备文件、/boot 内核引导。一句话总结：静态的进 /usr，变的进 /var，配置在 /etc。

### 40. 线上一台服务器需要增加硬盘，该怎么操作？

**要点：**
- 确认接口/热插拔支持（SAS/SATA 一般支持热插，NVMe 多数支持）；物理插入后 `lsblk` 确认识别为新盘（如 /dev/sdb）
- 分区（可选）：`fdisk`/`parted`（>2T 用 parted + GPT）；或整盘不分区
- 文件系统：`mkfs.xfs /dev/sdb1`
- 挂载：临时 `mount`；持久化写 `/etc/fstab`（**用 UUID 而不是设备名**：`blkid` 查 UUID，防盘符漂移）→ `mount -a` 验证 fstab 无误
- 若要扩到已有目录：走 LVM（pvcreate → vgextend → lvextend → xfs_growfs）

**一句话答：** 插盘后 lsblk 确认识别，parted 或 fdisk 分区，mkfs 做文件系统，然后挂载并把 UUID 写进 fstab 开机自动挂——一定 mount -a 验证 fstab 没写错，写错服务器会起不来。如果目标是扩容已有目录而不是新目录，就用 LVM 在线扩。

### 41. 某一天突然发现 Linux 系统文件只读，该怎么办？

**要点：**
- 文件系统变只读通常是**内核检测到磁盘错误后的自我保护**（remount-ro），或磁盘掉线/RAID 卡故障
- 先看证据：`dmesg | tail` 找 I/O error / EXT4-fs error；`mount | grep ro` 确认哪个分区
- 处理顺序：备份/导出还读得到的数据 → 卸载分区 `umount` → 检查修复 `fsck -y /dev/sdX`（**必须先卸载**）→ `mount -o remount,rw` 或重启后验证
- 若反复出现：`smartctl` 查盘健康，RAID 阵列看是否有 degraded，必要时换盘
- 数据盘可用 `mount -o remount,rw` 应急恢复写，根分区错误建议走 fsck

**一句话答：** 文件系统突然只读，一般是内核发现磁盘 IO 错误后把分区自动改成只读保护数据。处理三步：dmesg 找到报错确认哪块盘出事，先抢救数据，然后卸载分区跑 fsck 修复再重新挂载。反复发作就要查 SMART 和 RAID 状态，八成是盘要坏了。

### 42. Keepalived 出现脑裂，是什么原因？

**要点：**
- 脑裂 = **心跳断了但双方都活着**，两台都抢 VIP，导致 IP 冲突、流量分裂
- 常见原因：心跳链路故障（VRRP 组播被交换机/防火墙拦截——很多机房默认挡 224.0.0.18）、网卡/交换机问题、iptables 拦 VRRP 协议、优先级配置错误
- 危害：VIP 漂移混乱、ARP 表错乱、重复写后端
- 检测：两台同时抓 `ip addr | grep VIP`；`tcpdump vrrp` 看心跳
- 预防：心跳走独立直连线或专用 VLAN、放行 vrrp 协议、加**冗余心跳**（双链路）、用 arpangel/第三方检测脚本封 ARP

**一句话答：** 脑裂就是心跳断了但双方都没死，于是都认为对方挂了、都去抢 VIP。最常见的根因是心跳链路出问题——组播被交换机挡了、网卡抽风或者防火墙拦了 VRRP 协议。预防靠独立心跳线加协议放行，发现 IP 冲突告警要第一时间两台机器同时看 ip addr。

### 43. 你之前使用过哪些监控系统，都分别监控了什么？

**要点：**
- **Prometheus + Grafana**：指标监控主力——主机（node_exporter：CPU/内存/磁盘/网络）、中间件（mysqld_exporter、redis_exporter）、应用自定义指标、K8s（kube-state-metrics）；告警走 Alertmanager
- **Zabbix**：传统主机/网络设备监控，SNMP 采集交换机路由器，模板成熟
- **ELK/EFK**：日志监控与检索（error 关键字告警）
- **云监控**：阿里云 CloudWatch 类，管云资源
- 分层回答：基础资源层、中间件层、应用层、业务层（订单量等黄金指标）、链路（可选 SkyWalking）

**一句话答：** 主力是 Prometheus+Grafana，node_exporter 采主机、各中间件 exporter 采数据库缓存、Alertmanager 告警；网络设备用 Zabbix 走 SNMP；日志检索和关键字告警用 ELK。我关注的是分层覆盖：资源、中间件、应用、业务四层都要有，缺一层就有盲区。

### 44. 谈谈你对 SRE 理念的理解？

**要点：**
- Google 提出：用**软件工程方法**解决运维问题，而不是人肉 Ops
- 核心概念：**SLI/SLO/SLA**（指标/目标/协议）、**错误预算**（错误预算用完就冻结发布，可靠性优先）、**拥抱风险**（100% 可靠不现实也不经济）
- 实践：自动化优先（消灭 toil 苦力活）、事前演练（混沌工程）、**无责复盘（blameless postmortem）**、值班轮换
- 与传统运维区别：被动救火 → 主动工程化；以 SLO 数据驱动决策

**一句话答：** SRE 的本质是把运维从手工作坊变成工程学科：用 SLO 量化可靠性，用错误预算平衡发布速度和稳定性，用自动化消灭重复劳动，故障后做无责复盘落到机制改进。我认为最实用的两条是错误预算——它让研发和运维不打架，和自动化优先——人的时间应该花在写工具上而不是当人肉重启按钮。

### 45. 当你加入新运维团队时，如何开展工作？

**要点：**
- **先摸底再动手**：要权限、要文档、要架构图——了解业务架构、系统拓扑、监控覆盖、备份现状
- 盘点风险：过期的证书、无监控的服务、无备份的库、没人知道的"祖传"机器
- 熟悉流程：变更怎么走、故障怎么响应、值班怎么排
- 前两周不搞大动作：从文档补全、小优化、接手值班做起，建立信任
- 30-60-90 计划：1 个月熟悉+接管，2 个月能独立处理故障+提改进，3 个月有落地项目

**一句话答：** 我的节奏是先摸底后动手：前两周把架构、监控、备份、变更流程全过一遍，顺手盘点出风险清单——比如没有备份的库和过期证书；然后从小优化和值班接手建立信任，一个月内能独立处理故障，三个月内拿出第一个落地的自动化改进。新人上来就重构是大忌。

### 46. 简述常见的网络 I/O 模型及应用场景？

**要点：**
- **阻塞 IO（BIO）**：read 一直等到数据来；简单，一连接一线程，连接多了线程爆炸
- **非阻塞 IO（NIO）**：轮询检查，没数据立刻返回 EWOULDBLOCK；配合循环使用
- **IO 多路复用**：`select/poll/epoll`——一个线程监听海量 fd，就绪了再处理；**Nginx/Redis/Kafka 的基石**
  - select：fd 上限 1024，每次全量拷贝+遍历
  - epoll：事件驱动，只返回就绪的 fd，支持百万连接
- **信号驱动 IO**：SIGIO 通知，少用
- **异步 IO（AIO/io_uring）**：内核完成拷贝后才通知，真正异步；io_uring 是新一代高性能方案

**一句话答：** 五种模型按"等待时能不能干别的"递进：阻塞最简单但一连接一线程扛不住量；多路复用 epoll 是重点——一个线程盯成千上万个连接，Nginx Redis 都靠它；异步 IO 最理想，内核全包了再通知你，io_uring 正在普及。面试核心讲清 epoll 为什么比 select 强：事件驱动、只返回就绪的、没有 1024 限制。

### 47. Linux 系统误删除文件，如何恢复？

**要点：**
- **第一原则：立刻停止写入**（最好卸载分区或 remount ro），防止数据块被覆盖
- 前提确认：有没有备份/快照（LVM 快照、云盘快照、rsync 副本）——有就直接恢复，成功率 100%
- ext4：`extundelete /dev/sdb1 --restore-file path`（需卸载）；xfs：基本无可靠免费工具，xfs_undelete 可尝试
- 进程仍占用的情况（最好恢复）：`lsof | grep deleted` 找到 fd，从 `/proc/<pid>/fd/<n>` 直接拷回
- 根治：重要数据上备份策略 + rm 别名加提醒/用回收站机制（如 trash-cli）

**一句话答：** 误删后第一步永远是停止写入防覆盖，然后按优先级来：有快照/备份直接还原；文件还被进程占用就从 /proc 的 fd 里拷回来；没有的话 ext4 用 extundelete 试运气，xfs 基本没戏。这事靠恢复不如靠预防——备份加快照才是正解。

### 48. 你觉得 SRE 核心工作有哪些？

**要点：**
- **可靠性工程**：定 SLI/SLO、容量规划、错误预算管理
- **自动化**：写工具消灭 toil（人工重复操作占比要持续下降）
- **可观测性**：指标/日志/链路三大件建设，告警质量优化（降噪、分级）
- **应急响应**：oncall 值班、故障指挥、应急预案与演练（混沌工程）
- **事后工程**：无责复盘、action item 跟踪、发布检查单
- 配合研发：性能优化、架构评审中的可靠性把关

**一句话答：** 五块：定 SLO 和容量规划保证"不出事"，自动化和工具化减少 toil，可观测性建设保证"看得见"，oncall 和演练保证"救得回来"，无责复盘保证"不二犯"。衡量 SRE 做得好不好的硬指标是 toil 占比下降和 MTTR 缩短。

### 49. 你们故障后怎么做复盘？

**要点：**
- **时间线还原**：从第一个信号到完全恢复，精确到分钟：发现→响应→止血→恢复
- **根因分析**：5 Why 追问到系统性原因（代码/流程/监控），停在"人的失误"就是没挖到根
- **无责文化**：对事不对人，否则大家报喜不报忧
- **Action list**：每条改进项有 owner 和 deadline，分类：防发现（监控）、防发生（代码/流程）、防扩大（熔断/限流）、快恢复（自动化）
- **沉淀**：复盘文档入知识库，同类场景更新预案，重要故障向全团队宣讲

**一句话答：** 我们的复盘四步：还原时间线、5 Why 挖根因、改进项落 owner 和期限、知识库沉淀。关键是无责文化——追责会让下次故障被隐瞒；改进项按"防发现、防发生、防扩大、快恢复"四类归档，下次值班的人拿着预案就能处理同类问题。

### 50. 如何建设自动化运维体系？

**要点：**
- 分层推进：
  1. **标准化**是地基：镜像、配置、命名、目录统一（没有标准化，自动化是给混乱加速）
  2. **工具化**：脚本库、自服务工单
  3. **平台化**：CMDB+堡垒机+发布系统+监控，Web 化自助操作
  4. **智能化**（远期）：异常检测、自动扩缩容、自愈
- 优先级：先自动**高频**操作（发布、扩容、巡检），再自动**高危**操作（减少人肉失误）
- 度量：自动化率、toil 占比、变更成功率、MTTR
- 引擎：Ansible/SaltStack（配置）、Jenkins/GitLab CI（流水线）、K8s operator（自愈）

**一句话答：** 我的建设路径是标准化→工具化→平台化→智能化，标准化是地基——没有统一的镜像和配置规范，自动化只会放大混乱。落地顺序先挑高频操作（发布、巡检）再挑高危操作，用自动化率和 MTTR 度量成效，最终目标是人只处理机器处理不了的事。

### 51. 你用过哪些企业虚拟机技术？

**要点：**
- **KVM**：Linux 内核原生虚拟化，主流选择（OpenStack 底座）
- **VMware vSphere/ESXi**：商业标杆，vMotion 在线迁移、HA 集群，传统企业保有量大
- **Xen**：早期公有云（AWS 初期）采用，半虚拟化
- **Proxmox VE**：开源自 KVM+LXC，中小团队常用
- **Hyper-V**：Windows 生态
- 容器补充：Docker/K8s 是**进程级**隔离，和虚拟机互补（安全要求高时虚拟机套容器）

**一句话答：** 生产里主要是 KVM 和 VMware 两派：KVM 开源生态配合 OpenStack 或 Proxmox，VMware vSphere 商业成熟、vMotion 和 HA 体验最好。现在还叠加容器层——虚拟机管隔离安全，容器管密度和交付速度，两者是互补不是替代。

### 52. 简述 KVM 虚拟化的核心实现？

**要点：**
- KVM 是**内核模块**（kvm.ko + 平台模块 kvm_intel/kvm_amd），把 Linux 内核变成 Type-1 hypervisor，每个虚拟机是一个普通进程
- 硬件辅助：CPU 靠 **Intel VT-x / AMD-V**（提供 root/non-root 两种模式），内存靠 **EPT/NPT** 二级地址翻译，没有硬件辅助时退回软件模拟（qemu 软件翻译，慢）
- **QEMU** 负责设备模拟：磁盘、网卡、显卡等 IO 设备
- **virtio** 半虚拟化设备：前端驱动（guest）+ 后端（host），绕过设备模拟大幅提升 IO 性能；virtio-net 配合 vhost-net 把数据面下沉内核
- 管理栈：libvirt（libvirtd + virsh）→ 上层 OpenStack/Proxmox

**一句话答：** KVM 本身是内核模块，借助 CPU 的 VT-x 硬件辅助把每台虚拟机跑成宿主机上的一个进程，内存靠 EPT 硬件翻译；设备 IO 由 QEMU 模拟，但高性能场景用 virtio 半虚拟化驱动——guest 里装前端驱动、宿主机后端直通内核，网络磁盘性能接近物理机。管理上走 libvirt，再往上才是 OpenStack 这类平台。

### 53. KVM 虚拟机有哪些常用的管理方式？

**要点：**
- 命令行：**virsh**（virsh start/console/list/shutdown、virsh dumpxml 导出配置）
- 图形/Web：virt-manager（单机）、**Proxmox VE**、OpenStack（大规模）、oVirt
- 磁盘镜像管理：qemu-img（create/convert/resize，qcow2 vs raw）
- 快照：virsh snapshot-create-as（内部快照）、外部快照（qemu-img snapshot）
- 热迁移：virsh migrate --live（共享存储前提）
- 云管：Terraform/OpenTofu provider 批量建虚机

**一句话答：** 单机日常用 virsh 命令行加 virt-manager 图形；规模化用 Proxmox 或 OpenStack 的 Web 界面。常用操作围绕 qemu-img 管镜像、virsh snapshot 管快照、live migrate 做不中断迁移——迁移的前提是两台宿主机共享存储。

### 54. Linux 系统无法启动，可能会有哪些原因？

**要点：**
- 按启动链排查：**BIOS/硬件**（磁盘掉线、RAID 降级卡住）→ **GRUB**（引导文件损坏、MBR 被覆盖、grub.cfg 错误）→ **内核/initramfs**（内核升级失败、驱动缺失）→ **根分区挂载**（**fstab 写错**最常见、文件系统损坏）→ **systemd**（关键服务失败卡住）
- 现场信息：卡住时的报错文字就是定位入口（GRUB error / Kernel panic / "Give root password for maintenance"= fsck 或 fstab 问题）
- 急救工具：救援模式/单用户模式（rd.break 或 init=/bin/bash）、LiveCD/救援盘挂盘修复
- 预防：改 fstab 用 `nofail` 选项（数据盘）、内核升级保留旧内核可回退、变更前快照

**一句话答：** 沿启动链背：硬件和 RAID、GRUB 引导、内核和 initramfs、根分区挂载、systemd 服务五级，每一级卡住的报错不一样。实战最高频是 fstab 写错和内核升级失败——进救援模式改 fstab 或选旧内核就行，fstab 里给数据盘加 nofail 是预防针。

### 55. Linux 服务器宕机，系统启动后怎么排查原因？

**要点：**
- **第一现场看日志**：
  - `last -x | head`：看宕机时间点（shutdown/crash 记录）
  - `journalctl --since "..." --until "..."` -b -1：上一次启动的日志末尾
  - `/var/log/messages`、`dmesg | grep -i error`
- **硬件级原因**：journal 跟不到断电瞬间 → 看 **IPMI/带外日志（SEL）**、`mcelog`（CPU 内存硬件错误）、kdump 的 **vmcore**（`crash` 工具分析，/var/crash）
- 常见根因：OOM（查 `grep -i "out of memory"`）、内核 panic（配了 kdump 才有 vmcore）、断电/掉电、硬件故障（ECC 报错）、内核 bug
- 预防：配置 kdump+PanicOnOOM 策略、带外监控告警

**一句话答：** 起来后三步：last -x 和 journalctl -b -1 看宕机前最后的日志，grep OOM 和 panic 关键字；journal 断层说明是硬件级断电，就翻 IPMI 带外日志和 mcelog；配了 kdump 的用 crash 分析 vmcore 拿到 panic 时的内核现场。没配 kdump 的下次事故就只能靠猜——所以事后第一件事是把 kdump 配上。

### 56. Linux 文件系统损坏，如何修复？

**要点：**
- **前提：先卸载**（umount 或救援模式，根分区用 LiveCD/救援环境），带电 fsck 会造成二次损坏
- ext4：`fsck.ext4 -y /dev/sdb1`（-y 自动确认）；xfs：`xfs_repair /dev/sdb1`（严重损坏先 `xfs_repair -L` 清日志——**会丢未落盘数据**，最后手段）
- 修复前有条件先备份：`ddrescue` 镜像整盘再修
- 修复后：mount 验证、跑业务验证数据完整性、查 SMART 找根因（为什么坏：坏盘/断电/内核 bug）
- 预防：定期 fsck 检查、UPS 防断电、RAID+监控

**一句话答：** 核心纪律是先卸载再修：ext4 用 fsck -y，xfs 用 xfs_repair，xfs 超严重时才用 -L 清日志（会丢数据，最后手段）。修完要回答"为什么坏"——SMART 一查如果是坏盘就该换盘，只修文件系统不查根因迟早复发。

### 57. eBPF 是什么？有哪些应用场景？

**要点：**
- eBPF = extended Berkeley Packet Filter：内核里的**安全可编程虚拟机**，允许把经验证器的沙箱代码安全地挂载到内核事件（kprobe/tracepoint/XDP/socket）上执行
- 对比传统：不用改内核/不用写模块/不用重编译——内核版本向前兼容，"内核态的 JavaScript"
- 组成：BPF 程序 + **Verifier**（安全验证）+ JIT 编译 + **BPF Map**（内核/用户态共享数据）
- 场景：
  - 可观测：性能剖析、追踪（**bpftrace**、BCC 工具集、Pyroscope 持续剖析）
  - 网络：Cilium（K8s 网络策略/负载均衡）、XDP 高速丢包/抗 DDoS
  - 安全：运行时检测（Falco、Tetragon）
- 限制：需要较新内核（4.x+，功能随版本增强），Cilium 等对内核版本有要求

**一句话答：** eBPF 是内核里的安全沙箱虚拟机，让一段经过验证器检查的代码挂在内核事件上跑，不用改内核不用重编译——等于给内核装了安全的插件系统。三大场景：可观测（bpftrace、持续剖析）、网络（Cilium、XDP 抗 DDoS）、安全（Falco 运行时检测）。它是近年云原生网络和观测体系的技术底座。

### 58. 你用过哪些基于 eBPF 的排查工具？

**要点：**
- **bpftrace**：一行脚本跟踪内核事件（`bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[comm] = count(); }'` 统计谁在开文件）
- **BCC 工具集**：execsnoop（短命进程）、opensnoop（文件访问）、tcplife（TCP 生命周期）、biolatency（块 IO 延迟分布）、cachestat（页缓存命中）
- **perf**（部分子功能基于 eBPF/tracepoint）：`perf trace`、CPU 火焰图
- 网络类：**Cilium Hubble**（K8s 流量可视）、XDP 工具
- 安全类：Falco/Tetragon（syscall 级入侵检测）

**一句话答：** 排查三件套：bpftrace 写一行脚本跟踪任意内核事件，BCC 的 execsnoop/opensnoop/tcplight 等现成工具查短命进程、文件访问和 TCP 连接生命周期，K8s 里用 Hubble 看服务间流量。相比 strace 那种全量跟踪，eBPF 工具开销低得多，可以安全地跑在生产上。

### 59. 你用 BCC 具体解决过什么线上问题？

**要点：**（面试话术模板，按真实经历替换/改写）
- 结构：现象 → 为什么用 BCC → 工具与输出 → 结论
- 示例方向：
  - **磁盘抖动但 iostat 看不出谁干的** → biolatency 看延迟分布 + biotop 找到高频写的进程 → 发现某应用日志刷盘策略问题
  - **CPU 周期性飙高但 top 平稳** → execsnoop 抓到定时短命进程（监控 agent 反复重启）
  - **连接数异常** → tcpconnect/tcp-life 看连接建立与时长，定位到连接泄漏的模块
- 强调 BCC 的价值：传统工具（top/iostat）粒度不够时，eBPF 能直接回答"是哪个进程、哪个系统调用"

**一句话答：**（示范框架）讲一个真实场景：比如磁盘 util 不高但业务 IO 延迟抖动，iostat 只能看盘看不出进程，我用 BCC 的 biolatency 拿到延迟分布、biotop 锁定高频写的进程，最后发现是应用的日志同步刷盘问题，改成批量异步后抖动消失。核心卖点：eBPF 能把问题定位到"进程+系统调用"级别，传统工具做不到。

### 60. 你在上家公司做过的最有成就感的事情是什么？

**要点：**（面试话术模板，按真实经历替换）
- 选材原则：**结果可量化**（时间/成本/稳定性数字）+ 有技术深度或跨团队协作
- 好素材方向：把人肉发布改成 CI/CD（发布时间从小时到分钟）、建设监控把 MTTD 从用户报障降到主动告警、自动化巡检替代人工、故障复盘推动架构改进
- 讲法：背景痛点 → 我做了什么（突出个人贡献）→ 量化结果 → 带来的机制改变
- 避免空泛："认真负责"不是成就；避免抢功："我们"和"我"分清楚

**一句话答：**（示范框架）挑一件有数字的事讲：比如接手时发布靠人肉 SSH，平均一次 40 分钟且月月出事故；我推动上了 GitLab CI + Ansible 自动化发布，带灰度和一键回滚，发布缩到 5 分钟、变更事故归零。重点展示：你发现问题、设计方案、推动落地的完整闭环，而不是单纯执行。

### 61. Paxos 和 Raft 有什么作用？有哪些应用场景？

**要点：**
- 作用：**分布式一致性算法**——多副本系统在部分节点故障/网络分区下，对同一个值达成一致，保证不丢不乱
- **Paxos**：Leslie Lamport 提出，理论奠基但难以理解和工程实现（Multi-Paxos 用于状态机复制）
- **Raft**：为可理解性设计的等价算法，拆成三个子问题：**Leader 选举、日志复制、安全性**；任期（term）+ 多数派投票
- 应用：**etcd**（Raft，K8s 数据库）、**Consul**（Raft）、TiKV/TiDB（Raft）、ZooKeeper（ZAB，类 Paxos）、Kafka 新版 KRaft 取代 ZooKeeper
- 运维相关性：etcd 集群必须奇数节点（3/5），理解多数派才能理解"为什么两节点集群不如单节点"

**一句话答：** 它们解决的是多副本一致性：一部分节点挂了集群照样对外提供一致的数据。Paxos 是理论源头但难懂，Raft 是工程化翻版，拆成选主、日志复制、安全三件事。运维天天在用：etcd 和 Consul 就是 Raft 实现，所以 K8s 的 etcd 要部署 3 或 5 台奇数节点——多数派存活才能写入。

### 62. MBR 和 GPT 分区有什么区别？

**要点：**
- **MBR**：分区表在磁盘头 512 字节内；最多 **4 个主分区**（扩展分区绕路）、最大支持 **2TB**；无冗余，表坏了盘就丢
- **GPT**：UEFI 时代的分区表；支持**128 个分区**、容量上限 9.4ZB（实际无感）；头和分区表**双份冗余**+CRC 校验，可自我恢复
- 配套启动方式：MBR 配 BIOS + GRUB stage；GPT 配 **UEFI**（需 ESP 分区）
- 实操：>2TB 磁盘必须 GPT（parted）；系统盘从 BIOS 换 UEFI 时注意引导模式匹配

**一句话答：** MBR 是老方案：最多 4 主分区、最大 2TB、分区表单份没冗余；GPT 是现代方案：128 分区、容量几乎无上限、分区表双备份可自愈，配套 UEFI 启动。实操记一条：超过 2TB 的盘必须用 parted 走 GPT，fdisk 会出问题。

### 63. 进程和线程有什么关系和区别？

**要点：**
- **进程 = 资源分配单位**：独立地址空间、fd 表、内存；**线程 = CPU 调度单位**：同一进程内多线程**共享**地址空间和资源，各有自己的栈和寄存器
- 开销：进程创建/切换重（页表切换），线程轻；通信：进程要 IPC，线程直接读写共享变量（但要加锁防竞争）
- 稳定性：进程隔离好，一个线程段错误整个进程崩
- Linux 实现：线程就是"共享资源的进程"（task_struct，clone 标志差异），`ps` 里线程以 LWP 显示（`ps -eLf`）
- 场景：Nginx 多进程（隔离好）、Redis 单进程单线程（避免锁）、JVM 多线程

**一句话答：** 进程是资源分配的最小单位，线程是调度的最小单位；同一进程的线程共享内存所以通信便宜但要用锁，进程之间隔离好崩一个不影响别人。Linux 里线程本质上就是共享了地址空间的进程。选型直觉：要隔离和稳定性用多进程（Nginx），要高并发共享状态用多线程或事件驱动。

### 64. 高并发场景下，系统本地端口耗尽如何解决？

**要点：**
- 现象：`Cannot assign requested address`，大量 TIME_WAIT 占着本地端口（本机主动连外，一个目标 IP:端口 对，本地端口只能用一个）
- **根治：改长连接**（HTTP keepalive、连接池）——端口耗尽本质是短连接风暴
- 参数缓解：
  - `net.ipv4.ip_local_port_range = 1024 65535`（扩大可用范围）
  - `net.ipv4.tcp_tw_reuse = 1`（快速复用 TIME_WAIT 端口，客户端方向）
- 架构缓解：加**多个出口 IP**（一个目标最多约 6.4 万连接/IP）、上代理/连接复用层
- 误区：`tcp_tw_recycle` 已移除，别用

**一句话答：** 端口耗尽的根因是高频短连接，每个出站连接占一个本地端口，TIME_WAIT 还要压 60 秒。根治是上长连接和连接池；参数层面扩大 ip_local_port_range 加开 tcp_tw_reuse；再不够就加出口 IP——对同一个目标，每个 IP 有约 6.4 万个端口可用。

### 65. 说一些常用的 Linux 内核参数？

**要点：**（挑理解深刻的讲，每个都要能说清"为什么"）
- **连接/端口**：
  - `net.ipv4.tcp_tw_reuse = 1`：TIME_WAIT 复用（客户端）
  - `net.ipv4.ip_local_port_range = 1024 65535`：出站端口范围
  - `net.core.somaxconn = 65535`：listen 队列上限（Nginx/Redis 高并发必调）
  - `net.ipv4.tcp_max_syn_backlog`：SYN 半连接队列
- **内存**：
  - `vm.swappiness = 10`：少用 swap
  - `vm.overcommit_memory = 1`（Redis 官方建议）
  - `vm.dirty_ratio/dirty_background_ratio`：刷盘阈值
- **文件**：`fs.file-max`、`fs.nr_open`（句柄上限）
- **其他**：`net.ipv4.ip_forward = 1`（路由/NAT）、`net.ipv4.conf.all.rp_filter`（反向路径校验，多网卡环境注意）
- 持久化在 `/etc/sysctl.d/*.conf`，`sysctl -p` 生效；改前 `sysctl -a` 记录基线

**一句话答：** 我按场景背参数：高并发接入调 somaxconn 和 tcp_max_syn_backlog，出站密集调 tcp_tw_reuse 和端口范围，数据库主机调 vm.swappiness 和 overcommit_memory，网关开 ip_forward。每个参数我都要求自己能说清症状和原理——只背数值不懂机制，面试一追问就露馅。

### 66. 在 CDN 平台上刷新了一条资源，如何检测这条资源有没有刷新成功？

**要点：**
- **看响应头**：`curl -sI https://cdn.example.com/path/file.js` 对比刷新前后——CDN 命中一般带 `X-Cache: HIT`（各厂商头名不同：Ali-Swift-Cache、X-Swift-CacheTime 等）
- 刷新原理：URL 刷新=让边缘节点删缓存回源拉新 → 刷新成功后**下一次请求应是 MISS（回源）**，再请求才 HIT（且内容是新的）
- 验证内容新旧：对比 `ETag` / `Last-Modified` / 响应体 hash 与源站是否一致
- 部分厂商有任务查询 API（刷新任务状态：提交/处理中/成功），如阿里云 DescribeRefreshTasks
- 注意传播延迟：刷新指令全网生效有几秒到几分钟，别刚提交就验证；预热（prefetch）是主动拉，刷新（refresh）是失效

**一句话答：** 刷新是让边缘节点把缓存删掉回源取新，所以验证方法是刷新后立刻 curl -I 打这条 URL：第一跳应该是 X-Cache: MISS（回源拉新），再请求变 HIT，同时对比 ETag 和源站一致就确认成功。厂商控制台一般也有刷新任务状态 API 可以直接查。刚提交就验证会误判，要留几分钟传播时间。

### 67. Linux 系统中 GRUB 是什么，有什么作用？

**要点：**
- GRUB（GRand Unified Bootloader）= **引导加载程序**，BIOS/UEFI 之后的启动第二棒
- 作用：定位并加载内核 vmlinuz + initramfs 到内存，传内核参数（如 `root=`、`rw`），移交控制权
- GRUB2 特性：菜单编辑（按 e 编辑启动项）、救援模式、多内核选择（升级失败回退旧内核）、加密
- 配置：`/etc/default/grub`（默认项/超时/参数）→ `grub2-mkconfig -o /boot/grub2/grub.cfg` 生成（Debian 系 update-grub）
- 故障场景：MBR 损坏进 grub rescue → LiveCD `grub-install` 重装

**一句话答：** GRUB 是引导程序，BIOS 自检后由它把内核和 initramfs 加载进内存、带上启动参数然后把控制权交给内核。实用价值是启动菜单：可以选内核版本——内核升级出问题时回退旧内核就靠它，改内核参数也是在 GRUB 菜单里按 e 临时改。

### 68. Linux 文件权限里所有者、所属组、其他用户分别代表什么？

**要点：**
- 每个文件三个身份：**所有者 u**（user，属主）、**所属组 g**（group）、**其他 o**（other）
- 每类身份三种权限：**r**（4，读/列目录）、**w**（2，写/目录内建删改名）、**x**（1，执行/进入目录）
- `ls -l` 的 rwxr-x---：属主读写执行、组读执行、其他人无权
- 特殊位：SUID（u+s，执行时借用属主身份如 passwd）、SGID（目录继承组）、Sticky（o+t，/tmp 只能删自己的）
- 命令：`chmod`（数字/符号）、`chown user:group file`、`chgrp`
- 注意：目录的 x 是"进入"，没有 x 只有 r 只能看文件名列表进不去

**一句话答：** 权限模型是三维乘三行：属主、组、其他三类身份，每类有读写执行三种权限，数字 421 相加。两个易错点：目录的 x 是进入权限不是执行；/tmp 的 sticky 位是"各人只能删各人的文件"。实际管理要配合 chown 和最小权限原则，能不 777 就不 777。

### 69. Linux 中 umask 的作用是什么？

**要点：**
- umask = 新建文件/目录时**默认权限的"减法"掩码**：文件基始 666、目录 777，减去 umask 位
- umask 022 → 新文件 644（666-022）、新目录 755（777-022）；umask 077 → 文件 600、目录 700（私有）
- 注意：文件基始就是 666（不带 x）——umask 再怎么设，新建文件默认也不会有执行位
- 查看/设置：`umask`、`umask 027`（当前会话）；持久化在 `/etc/profile`、`~/.bashrc`
- 与 chmod 区别：umask 管"出生默认值"，chmod 管"修改已有权限"

**一句话答：** umask 决定新文件的出生权限：文件的基始权限 666、目录 777，减去掩码就是默认权限，比如 022 出来就是 644 和 755。安全敏感机器会设 077 让新文件默认只有自己能读。记一个细节：文件基始没有执行位，所以 umask 管不出可执行文件。

### 70. 日志报错 Could not resolve host 域名解析失败，如何解决？

**要点：**
- 逐步定位：
  1. `ping IP` 通不通——**网络层**问题还是纯解析问题
  2. `cat /etc/resolv.conf` 看 nameserver 是否有效（云主机常见：resolv.conf 被覆盖/清空）
  3. `nslookup 域名` 或 `dig @8.8.8.8 域名`——**指定 DNS 测试**区分本机配置问题还是 DNS 服务问题
  4. `getent hosts 域名` 走系统完整解析链（含 hosts/nsswitch）
- 常见根因：resolv.conf 错误、DNS 服务故障、内网域名没配私有 zone、nsswitch.conf 顺序错、业务用了过期的 DNS 缓存
- 修复后用 `dig +short` 验证；云环境确认 VPC DNS（如阿里云 100.100.2.136）未被破坏

**一句话答：** 先分层：ping IP 通说明网络没问题、纯解析挂了；然后 dig 指定公共 DNS 测一下——能解析说明本机 resolv.conf 配置坏了，不能解析说明上游 DNS 或该域名本身的问题。云主机还要留意 resolv.conf 被网络服务覆盖这种经典坑。

### 71. Linux 应用进程莫名被杀掉，可能是什么原因？

**要点：**
- **OOM Killer**（最常见）：内存耗尽内核挑进程杀 → `dmesg | grep -i "killed process"`、`journalctl -k | grep -i oom` 能看到现场
- **systemd**：服务配置 Restart 策略、LimitNPROC/MemoryMax 触发 cgroup 限制（`MemoryMax=` 超限被杀，journalctl -u 看得到）
- **cgroup/容器**：容器内存超 limit 被 OOM 杀（`docker inspect` 看 OOMKilled 字段，K8s 看 pod 状态 OOMKilled）
- **人为/脚本**：killall/pkill 误伤、crontab 清理脚本、supervisor 拉起策略
- **安全组件**：杀毒/EDR（Linux 上如安全组的 HIDS）判定恶意进程拦截
- 排查链：dmesg → journalctl -u → cgroup 事件 → 审计日志（auditd）确认谁发的信号

**一句话答：** 按概率排：先 dmesg 查 OOM Killer 现场——内存爆了内核杀的会有完整记录；再查 systemd 或 cgroup 的资源限制，容器里就是超 memory limit 被杀；都不是就查有没有清理脚本误杀和安全软件拦截，auditd 能告诉你信号是谁发的。

### 72. crontab 定时任务无法执行，有哪些常见原因？

**要点：**
- **环境变量**：cron 环境极简（PATH 只有 /usr/bin:/bin），脚本里命令写全路径或脚本开头 export PATH；**没有登录 shell 的 profile**
- **路径问题**：脚本里用相对路径，cron 的 cwd 是用户家目录
- **权限**：脚本无执行位、crontab 属主不对（`/var/spool/cron` 权限异常）
- **服务**：crond 没跑（`systemctl status crond`）；`/etc/cron.deny` 拒绝
- **时间**：服务器时区/时间不对；任务表达式写错（分 时 日 月 周）
- **输出**：任务报错被吞——养成 `command > /tmp/xx.log 2>&1` 习惯，先看邮件（/var/spool/mail）
- SELinux：`/var/log/audit/audit.log` 拒绝记录

**一句话答：** 排查口诀：先看 crond 活没活，再看 /var/spool/mail 里的报错——八成是环境变量：cron 的 PATH 和你登录 shell 完全不同，命令要写全路径。其他高频坑：脚本相对路径、没执行位、时区不对。调试期把输出重定向到日志文件，一跑就知道卡在哪。

### 73. 有一台在用的旧服务器需要下线，你会怎么做？

**要点：**
- **盘点依赖**（最重要、最容易被漏）：这台机器被谁依赖——用 `ss -tnp` 看连接来源、nginx/zabbix/跳板配置里 grep 主机名/IP、问业务方确认；DNS/hosts/配置文件里的引用
- **数据处置**：确认数据是否已迁移、备份是否可恢复、敏感数据**擦除**（wipe/ shred，涉密走消磁物理销毁）
- **摘流观察**：从负载均衡/域名摘除 → 观察 1-2 周有没有隐性调用（监控告警兜底）
- **正式下线**：停服务 → 从监控/CMDB/堡垒机/自动化清单移除（防"僵尸资产"继续告警）→ 释放资源 → 文档归档
- 反向操作也重要：**更新所有记录**，别让 CMDB 里还留着"在用"

**一句话答：** 我的关键动作是"摘流观察期"：先盘点谁在依赖它——连接来源、配置引用、DNS 记录，然后从负载均衡摘掉但机器保留一到两周，盯着监控看有没有隐性调用冒出来，确认无人才关机、擦数据、从 CMDB 和监控里移除。直接拔电的下线方式一定会翻车。

### 74. 安装软件提示 LD 动态链接库缺失，该怎样修复？

**要点：**
- 报错形如 `error while loading shared libraries: libxxx.so.1: cannot open shared object file`
- **看缺什么**：`ldd /path/to/binary | grep "not found"`
- **有包就装包**：`yum provides "*/libxxx.so*"` / `apt-file search` 反查哪个包提供该库，装上即可
- **库文件在手但没被找到**：
  1. 放进标准路径或 `/etc/ld.so.conf.d/xxx.conf` 写入自定义路径
  2. `ldconfig` 刷新缓存（**装完第三方库必做**，最常被忘）
- 临时方案：`LD_LIBRARY_PATH=/custom/lib ./app`（不推荐长期用）
- 版本不对：装兼容包或建软链（如 libssl.so.1.1），注意软链是权宜之计

**一句话答：** 先 ldd 看具体缺哪个 so，有 yum provides 或 apt-file 反查出提供这个库的包直接装；库其实在但系统不知道，就把它写进 /etc/ld.so.conf.d 然后 ldconfig 刷新缓存——ldconfig 忘跑是最常见的低级错误。临时验证用 LD_LIBRARY_PATH，长期还是走正规路径。

### 75. 使用 & 把程序放后台运行，没多久就自动中断是什么原因？

**要点：**
- `cmd &` 只是把进程放到**后台作业**，它仍挂在当前 shell 的会话下；**会话结束（ssh 断开/终端关闭）时，内核给该会话的进程发 SIGHUP** → 进程退出
- 关键点：不是"&"丢了，是 **HUP 信号**把它带走
- 正确姿势：
  - `nohup cmd &`：忽略 SIGHUP，输出重定向 nohup.out
  - `setsid cmd`：开新会话脱离终端
  - `tmux`/`screen`：会话托管可重连，**交互式任务首选**
  - 生产服务：直接写 systemd unit，不要手拉进程
- 查证：`ps -o pid,ppid,sid,tty,cmd -p <pid>` 看 TTY 是不是 ?（脱离终端）

**一句话答：** & 只负责放后台，进程还挂在当前终端会话上；SSH 一断，内核向整个会话发 SIGHUP，进程就死了。所以要么 nohup 忽略 HUP，要么 setsid 换会话，交互式任务用 tmux 最稳——而长期跑的服务本来就该写 systemd，不该靠手工后台。

### 76. 进程被 OOM Killer 杀死，该如何排查分析原因？

**要点：**
- **确认现场**：`dmesg -T | grep -i -E "out of memory|oom"` 或 `journalctl -k | grep -i oom`，日志里有完整杀进程记录（谁被杀、当时各进程内存排名）
- **看谁吃了内存**：OOM 日志里的内存快照 + `ps aux --sort=-%mem | head`；`slabtop` 看内核内存；对照 `free -h` 是否真的物理内存+swap 都见底
- **分析根因**：
  - 应用内存泄漏（RES 持续上涨不回落，监控曲线看得见）
  - 突发大请求/大文件读入内存
  - 主机混部互相挤兑（本来给 A 的内存被 B 吃了）
  - `vm.overcommit_memory=0` 下内核宽松承诺内存、真用时爆炸
- **解决**：修泄漏是根治；缓解手段：限制进程内存（systemd MemoryMax/cgroup）、调 overcommit 策略、加内存、`oom_score_adj` 保护关键进程（值为 -1000 最不容易被杀）

**一句话答：** 先 dmesg 拿到 OOM 现场日志，里面有被杀进程和当时全系统的内存排名；然后判断是慢性泄漏（监控看 RES 趋势）还是突发挤兑（混部互相吃内存）。解决分三层：修泄漏是根治，短期用 cgroup 隔离和 oom_score_adj 保护关键进程，实在不够就加内存——注意加了内存不等于修了问题。

### 77. 简述 sudo 提权的工作原理？

**要点：**
- 流程：用户执行 `sudo cmd` → sudo 读 `/etc/sudoers`（或 /etc/sudoers.d/）→ 匹配 **who where=(as_whom) what** 规则 → 验证**本人密码**（默认，可 NOPASSWD）→ 以目标身份（默认 root）fork 执行 → 记录 syslog/audit
- 配置：`visudo`（语法检查，防写锁死自己）；规则例：`dev ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx`
- **安全机制**：细粒度白名单（只放行必要命令）、时间戳缓存（默认 5 分钟内免密）、日志审计（谁在哪执行了什么）
- 对比 su：su 是"换身份"（拿到目标用户完整环境），sudo 是"借权限"（单命令授权+审计），**企业都推 sudo**
- 高危：`ALL=(ALL) ALL` 等于送出 root；通配符写法要小心（`/usr/bin/vi` 可以 shell 逃逸成 root）

**一句话答：** sudo 读 sudoers 规则，按"谁在哪台机上能用什么身份跑什么命令"匹配，验完本人密码后以目标身份执行并记审计日志。它比 su 安全在三点：命令级白名单授权、有时间戳缓存、每条提权都有日志可查。运维上要警惕把 vi、less 这类能起 shell 的命令放进白名单——等于变相送 root。

### 78. 同为系统 1 号进程，systemd 对比 sysvinit 具备哪些优势？

**要点：**
- **并行启动**：sysvinit 串行按序拉脚本，systemd 按依赖图（Requires/After）**并行**拉服务 → 开机从分钟级到秒级
- **按需启动**：socket/path/dbus 激活——服务收到第一个请求才启动（sshd 场景少但 dbus 类常用）
- **统一管理**：`systemctl status/start/enable` 一套命令替代 service+chkconfig
- **进程追踪**：**cgroup 管理服务进程树**——kill 服务不留孤儿进程（传统 init 杀不干净子进程）
- **日志**：journalctl 结构化日志（配合持久化可查历史启动）
- 其他：定时器替代部分 crontab、快照/回滚 target、unit 依赖关系显式化

**一句话答：** 三个硬优势：并行启动让开机提速一个数量级、cgroup 进程管理让服务停得干干净净不留孤儿、journalctl 和 systemctl 让状态和日志一站式可查。再加上 socket 激活和 timer 这些能力，systemd 把 init 从"启动脚本执行器"升级成了系统服务管理平台。

### 79. 软链接和硬链接有什么区别？

**要点：**
- **硬链接**：目录项直接指向同一 **inode**——同一文件的多名字；`rm` 原名数据还在（链接数>0）；**不能跨文件系统**、**不能对目录**；`ls -l` 第二列链接数
- **软链接**：独立文件，内容是**路径字符串**，指向另一路径；可跨分区、可链目录；**原文件删除后变悬空链接**（红显）
- inode 验证：`ls -li` 硬链接 inode 号相同，软链接不同
- 用法场景：硬链接做防误删快照式备份；软链接做版本切换（/usr/lib 软链、发布 current 指向）

**一句话答：** 硬链接是同一个 inode 的两个名字，删一个另一个照常能用，但受限于不能跨分区不能链目录；软链接是个存路径的新文件，灵活能跨盘能链目录，但原文件没了它就悬空。验证方法 ls -li 看 inode 号，硬链接相同软链接不同。

### 80. systemd 在 Linux 系统中负责哪些核心任务？

**要点：**
- **系统启动**：作为 PID 1，按 target 管理启动流程（sysinit → multi-user → graphical）
- **服务管理**：所有 .service unit 的启动/停止/重启/开机自启，依赖管理
- **进程生命周期**：cgroup 分组、资源限制（CPUQuota/MemoryMax）、守护与重启策略（Restart=）
- **日志**：journald 收集与查询（journalctl）
- **扩展功能**：timer 定时任务、mount/automount 挂载、socket 激活、getty 登录、hostname/timedate 等系统设置（localed/timedated）
- 配套工具链：systemctl / journalctl / loginctl / networkctl

**一句话答：** systemd 是 PID 1 总管家：启动流程编排、服务托管和依赖管理、基于 cgroup 的进程与资源控制、journald 日志，外加 timer 定时、socket 激活这些平台能力。理解它的关键是 cgroup——systemd 是靠 cgroup 而不是 PID 列表来界定"一个服务的全部进程"。

### 81. 如何编写一个生产级 Systemd 服务配置文件？

**要点：**
- 位置：`/etc/systemd/system/xxx.service`（管理员自建）
- **[Unit]**：Description、After/Requires（依赖声明，如 After=network.target）
- **[Service]** 核心字段：
  - `Type=simple`（前台进程）/ `forking`（传统守护进程，配 PIDFile）/ `notify`（支持 sd_notify）
  - `ExecStart` 绝对路径；`ExecReload=/bin/kill -HUP $MAINPID`
  - `Restart=on-failure` + `RestartSec=5`（崩溃自愈，避免 Restart=always 掩护异常）
  - `User=` 非 root 运行
  - `LimitNOFILE=65535`（句柄）
  - 加固：`ProtectSystem=full`、`NoNewPrivileges=true`、`PrivateTmp=true`
- **[Install]**：`WantedBy=multi-user.target`
- 流程：写完 `systemctl daemon-reload` → `start` → `enable`；验证 `systemctl status` + `journalctl -u`
- 生产要点：日志重定向（StandardOutput=journal）、优雅停止（ExecStop）、依赖失败不拉起

**一句话答：** 生产配置我关注四块：Type 选对（前台程序 simple，老式守护进程 forking 加 PIDFile）、自愈策略 Restart=on-failure 加 RestartSec、资源与安全加固（User 非 root、LimitNOFILE、ProtectSystem）、依赖声明 After 和 Requires。写完 daemon-reload 再 enable，最后 journalctl 验证启动日志——配置文件写得好不好，看它崩了能不能自己爬起来。

### 82. 如何在业务高峰期安全地重启服务？

**要点：**
- **评估必要性**：高峰期能不重启就不重启——配置生效优先 `reload`（平滑重载，不断连接：nginx -s reload）而非 `restart`
- **必须重启时**：
  1. 摘流量：从 LB/upstream 摘除本节点，等连接排空（`ss -s` 看ESTABLISHED 降到低位）
  2. 预检查：配置 `nginx -t`、依赖服务健康
  3. 有备节点先切流，确认新实例正常再操作下一个（**滚动重启**，永远留有余量）
  4. 优雅停机：`systemctl restart` 配置好 KillSignal/TimeoutStopSec，让进程处理完存量请求
  5. 回流验证：健康检查通过后逐步回流，盯错误率和延迟
- **兜底**：错峰窗口+审批、失败回滚方案、变更前后监控对比

**一句话答：** 核心思路是"先摘流再动刀"：能用 reload 不用 restart；必须重启就先从负载均衡摘节点、等连接排空，配置预检后重启，健康检查过了再逐步回流，全程滚动保证集群有余量。高峰期变更还必须留好回滚方案——重启命令敲下去之前，先想清楚它起不来怎么办。

### 83. Linux 系统内存持续飙高，如何排查？

**要点：**
- **先分清正常与异常**：Linux 尽量用满内存做 cache（free 里 buff/cache 高不是问题），看 **available** 和**应用进程 RES**
- 找占用者：`ps aux --sort=-%mem | head`；按进程看趋势（监控曲线 RES 斜率）
- 判断模式：
  - **持续单调上涨不回落** → 内存泄漏：strace/valgrind（开发配合）、Java 用 jstat 看 GC（老年代回收不掉）、Go 用 pprof heap
  - **周期性波动** → 缓存策略问题或批量任务
  - **整机高但进程都不高** → 查 `slabtop`（内核 slab：dentry/inode 缓存，海量小文件机器常见）、共享内存段（`ipcs -m`）、TMPFS（`df -h | grep tmpfs`）
- 关联现象：swap 使用（vmstat si/so）、OOM 记录
- 佐证：`pmap -x <pid> | sort -k3 -rn | head` 看进程内存分布（堆/so/共享）

**一句话答：** 三层排查：先排除假象——cache 占用高是正常的，看 available；再找大户——ps 按 RES 排序，重点看曲线是不是单调涨（涨而不落就是泄漏，Java 看 GC、Go 上 pprof）；最后进程都不大但内存吃紧的，查内核 slab 和 tmpfs。pmap 能把单个进程的内存构成摊开看。

### 84. Linux 系统硬盘读写慢，如何排查？

**要点：**
- **确认现象**：`iostat -x 1`——看 %util（设备利用率）、await（请求平均等待 ms）、r/s w/s、大小；对照业务延迟
- **找元凶进程**：`iotop -oPa`（累计写排行）、`pidstat -d 1`
- **分层归因**：
  - 盘本身慢：`smartctl` 健康、是不是 SMR 盘/RAID 降级重建中、云盘是不是超了 IOPS/吞吐规格
  - **文件系统层**：海量小文件、碎片、日志模式；`df -i` 看 inode
  - **调用方式**：大量 sync、O_DIRECT 滥用、随机小 IO 打不满顺序 IO 性能
  - **其他**：swap 在换（vmstat si/so）、DDoS 写日志、备份任务撞高峰
- **验证与解决**：fio 基准测试对照厂商标称；解决方向：改批量/异步写、换 NVMe、读写分离、限流

**一句话答：** iostat -x 定盘：%util 高且 await 大说明这块盘真的忙；iotop 找进程看谁在写；然后归因——盘规格不够（云盘 IOPS 超限）、RAID 重建、还是应用 IO 模式烂（狂 sync 或随机小 IO）。有争议时 fio 跑个基准对数。解决优先改应用的 IO 方式，硬件升级是最后手段。

### 85. 防火墙 firewalld 和 iptables 的关系与区别？

**要点：**
- **共同底座**：CentOS7 的 firewalld 底层**仍然调用 netfilter/iptables（nftables 后端可选）**——不是替代内核机制，而是替代"管理方式"
- **iptables**：静态规则列表，改一条要全表刷；服务重启规则易丢（要 save）；无区域概念
- **firewalld**：**区域（zone）**模型（public/trusted/dmz 按接口套区域）、**动态管理**（改规则不断连接不重载全表）、服务/端口抽象（`firewall-cmd --add-service=http`）、D-Bus 接口
- 命令差异：`iptables -A INPUT -p tcp --dport 80 -j ACCEPT` vs `firewall-cmd --permanent --add-port=80/tcp && firewall-cmd --reload`
- Debian/Ubuntu 方向：**nftables** 是 iptables 的继任者（内核级统一），ufw 是简单前端

**一句话答：** 底层都是 netfilter，区别在管理层：iptables 是静态规则表，firewalld 在上面包了动态管理层——区域模型、改规则不用断连接、按服务名配置。日常选型跟发行版走：CentOS7+ 用 firewall-cmd，Debian 系新版本在向 nftables 迁移。

### 86. /etc/fstab 写错导致系统无法启动，怎么恢复？

**要点：**
- 现象：启动卡在 emergency mode / "Give root password for maintenance"，日志提示 mount 失败
- **恢复路径一（root 密码在手）**：emergency mode 直接获得 shell → `mount -o remount,rw /` → 改 `/etc/fstab` 修正 → `reboot`
- **恢复路径二（进不去 shell）**：救援盘/LiveCD/云控制台救援模式 → 挂载系统盘 → chroot 或直接改 fstab
- **写错的常见点**：UUID 写错（blkid 核对）、文件系统类型写错、挂载点目录不存在、选项拼写
- **预防**：写完 fstab 必 `mount -a` 验证再重启；非关键盘加 `nofail` 选项（挂不上也继续启动）；用 UUID 别用设备名

**一句话答：** 进 emergency mode，根分区默认只读，先 mount -o remount,rw / 改 fstab 保存重启就恢复了；root 密码没有就走救援模式挂盘改。这事的教训是流程性的：fstab 改完必须 mount -a 验证无误才能重启，数据盘记得加 nofail，别让一块盘挂失败卡死整个系统。

### 87. 什么是标准输入、标准输出、标准错误？

**要点：**
- 每个进程默认三个流（POSIX 定义）：**stdin=0**（键盘输入）、**stdout=1**（正常输出，缓冲）、**stderr=2**（错误输出，不缓冲，保证错误及时可见）
- 都默认连终端，可重定向：`cmd < in.txt`、`cmd > out.txt`、`cmd 2> err.txt`
- 管道只传 **stdout**（stderr 不进管道，除非 `2>&1`）
- `2>&1` 含义：让 fd2 指向 fd1 当前指向的地方（顺序敏感：`>f 2>&1` vs `2>&1 >f` 效果不同）

**一句话答：** 进程出生自带三条流：0 是标准输入、1 是标准输出、2 是标准错误，默认都接终端。重定向就是改它们的指向，比如 2>err.log 把错误单独收走。关键细节是管道只走 stdout，所以合并错误要用 2>&1。

### 88. 在命令后面加 2>&1 是什么意思？

**要点：**
- 字面：把 **fd 2（stderr）重定向到 fd 1（stdout）当前指向的位置**——通常目的是让错误信息跟着正常输出走同一个目的地
- 典型组合：`cmd > /tmp/run.log 2>&1`（日志和错误进同一文件）、`cmd 2>&1 | less`（错误也能翻页看）
- **顺序敏感**：`>f 2>&1` 两者都进 f；`2>&1 >f` 只有 stdout 进 f，stderr 仍回终端（1 已改道、2 先抄的旧地址）
- bash 快捷写法：`&> f` 等价 `> f 2>&1`
- cron 调试常用：`* * * * * cmd > /tmp/a.log 2>&1`

**一句话答：** 2>&1 就是把标准错误改道到标准输出正在去的地方，通常为了把正常和错误输出收进同一个文件。必须记顺序：>file 2>&1 才是全进文件，写成 2>&1 >file 错误还是会打到屏幕——因为 2 抄的是 1 改道前的旧地址。

### 89. 你在日常工作中 dd 命令用过哪些场景？

**要点：**
- **制作/恢复镜像**：`dd if=/dev/sdb of=/backup/sdb.img bs=4M status=progress`（整盘备份、换机迁移）；恢复反转 if/of
- **制作启动盘**：`dd if=xxx.iso of=/dev/sdX bs=4M`（U 盘刻录）
- **生成测试文件**：`dd if=/dev/zero of=test.bin bs=1M count=1024`（测磁盘配额、IO 压测前置）
- **测裸盘读写**：`dd if=/dev/zero of=test bs=1M count=1024 oflag=direct`（绕缓存测真实写速；读用 iflag=direct，测前注意对生产的影响）
- **擦除数据**：`dd if=/dev/zero of=/dev/sdX`（下线盘清零）
- 安全须知：**if/of 写反就是数据灾难**（全盘覆盖），执行前 dd if=/dev/disk of=... 双检查；进度用 status=progress

**一句话答：** 高频场景四个：整盘备份迁移、刻启动盘、生成大测试文件、绕缓存的裸盘读写测速。dd 的名声在于 if/of 写反直接覆盖整盘，所以我的习惯是先敲一遍命令自查再回车，一律带 status=progress 看进度，生产盘操作前先用 lsblk 三遍确认目标。

### 90. Linux 里 /dev/null 设备文件是什么？

**要点：**
- 字符设备（黑洞设备）：写入的内容**全部丢弃**，读取返回 **EOF**
- 典型用法：
  - `cmd > /dev/null 2>&1`：丢弃全部输出（cron 静默）
  - `cmd 2> /dev/null`：只留正常输出
  - `< /dev/null`：给程序"空输入"防止它等 stdin
- 用途延伸：测试程序写性能（写 /dev/null 排除磁盘因素）、安全清空文件 `> /dev/null` 不行，清文件是 `: > file`
- 对应的还有 /dev/zero（读出无限 0）、/dev/random、/dev/urandom

**一句话答：** /dev/null 是内核的黑洞设备：写进去的都丢掉，读它只会得到 EOF。日常两个用途——丢弃不需要的输出（> /dev/null 2>&1），和切断 stdin 防程序卡在等输入。同族还有 /dev/zero 和 /dev/urandom，分别是无限零和随机数源。

### 91. ss 和 netstat 同样查看端口连接，ss 相比 netstat 优势在哪？

**要点：**
- **数据源**：netstat 遍历 **/proc/net**（逐个读 procfs 文件再拼装，连接多时极慢）；ss 直接用 **netlink 套接字向内核要**，内核态一次返回
- **性能**：十万级连接时 netstat 可能卡几十秒，ss 秒出——高并发机器必用 ss
- **信息更全**：`ss -ti` 能看每条 TCP 的 RTT、cwnd、拥塞算法等内核 TCP 状态；`ss -s` 汇总统计
- **过滤更强**：`ss -tlnp 'sport = :80'`、`ss -o state established '( dport = :443 )'` 原生过滤语法
- 现状：netstat 属于被废弃的 net-tools 包（ifconfig 同命），ss 是官方继任

**一句话答：** 核心差异是数据源：netstat 读 /proc 文件慢慢拼，ss 走 netlink 直接问内核，连接量大时速度差一个数量级。ss 的输出也更专业——ss -ti 能看每条连接的 RTT 和拥塞窗口，还支持类 SQL 的过滤表达式。新机器上 net-tools 已经不预装了，ss 是事实标准。

### 92. Linux 两台服务器之间数据同步如何实现？

**要点：**
- **一次性/手动**：`scp`、`rsync -avz --progress`（断点续传、增量、保留属性，首选）
- **定时增量**：rsync + **crontab**（简单可靠）；rsync + inotify（`inotifywait -m` 监听文件变化触发实时同步，秒级）
- **实时双活**：**lsyncd**（inotify+rsync 封装）、**DRBD**（块设备级实时复制，数据库底层）
- **共享存储替代同步**：NFS 挂载（多台读同一份）、对象存储（OSS/S3）
- 大规模文件分发：BitTorrent 类（P2P 加速，如数据处理场景）
- 选型逻辑：分钟级容忍 cron+rsync，秒级要 inotify/lsyncd，块级一致 DRBD，多读场景直接上共享存储

**一句话答：** 按实时性选型：一次性的 scp/rsync；分钟级增量 rsync 配 crontab；秒级实时用 lsyncd——inotify 监听文件变化触发 rsync；块设备级的一致性复制是 DRBD。如果多台机器都要读，其实更好的思路是别同步、直接挂 NFS 或对象存储。

### 93. Linux 排查 8080 端口被哪个进程占用，常用什么命令？

**要点：**
- `ss -tlnp | grep 8080`：直接给端口对应 PID/进程名（首选）
- `lsof -i :8080`：同样效果，输出更详细（用户、fd）
- `fuser 8080/tcp`：只报 PID，配合 `ps -fp $(fuser 8080/tcp 2>/dev/null)` 
- 找到后 `ps -fp <pid>` 看完整命令行、`systemctl status <pid>` 反查是否 systemd 托管
- 处理：确认归属后 kill / systemctl stop，改应用端口，或换端口部署

**一句话答：** ss -tlnp 或 lsof -i :8080 一条命令出结果：监听的 PID 和进程名直接给。拿到 PID 后 ps -fp 看完整命令行确认归属，再决定停服务还是改端口——杀进程前先搞清楚是谁，别把同事的服务干掉。

### 94. Linux 一个进程最多能创建多少线程？

**要点：**
- 受四重限制：
  1. `ulimit -u`（max user processes，单用户进程+线程总数）
  2. `ulimit -v` 虚拟内存 / 每线程栈 `ulimit -s`（默认 8MB——线程多时虚拟内存先爆）
  3. `kernel.pid_max`（默认 32768 或 4194304，全系统 PID 上限，线程也占 PID）
  4. `kernel.threads-max`（系统级线程数上限，与内存挂钩）
- 查看：`cat /proc/sys/kernel/threads-max`、`ulimit -a`；实测报错 `EAGAIN Resource temporarily unavailable` 即触限
- 调整：limits.conf/limits.d 持久化、`sysctl pid_max=`；但**线程数本身是坏味道**——该优化模型（线程池/事件驱动）而不是堆线程

**一句话答：** 理论上没有固定数，卡在四层限制上：用户的 ulimit -u、每线程 8MB 默认栈带来的虚拟内存压力、系统 pid_max 和 threads-max。报 Resource temporarily unavailable 就是撞墙了。但工程答案更重要：线程几千上万说明模型该换了——上线程池或事件驱动，加限制只是止痛。

### 95. Linux 怎么限制某个目录可用磁盘容量？

**要点：**
- 标准答案：**quota 磁盘配额**，但 quota 基于**用户/组**，"限目录"有两种落地：
  1. **独立分区/LVM + 挂载**：把目录单独成一个文件系统，天然封顶（最常用最可靠）——LVM 上 lvcreate 固定大小，满了只影响该目录
  2. **镜像文件 loop 挂载**：`dd` 造固定大小镜像 → `mkfs` → `mount -o loop`（无需新分区，虚拟机/容器常用）
  3. 传统 quota 对目录不适用；**project quota**（XFS 支持，针对目录配额）是正解之一
- 应用层：TMPFS 限 size（`mount -t tmpfs -o size=2G`）

**一句话答：** 目录本身没法直接限，常见三条路：给这个目录单独划一个固定大小的 LVM 逻辑卷挂上去，物理封顶；或者 dd 造个镜像文件 loop 挂载，不用重新分区；正经的文件系统级方案是 XFS 的 project quota，按目录树配额。临时目录还可以用 tmpfs 的 size 参数。

### 96. df 和 du 显示空间不一致是什么情况？

**要点：**
- **原因一：已删除但被进程占用**（最常见）——du 看不到文件（目录项没了），df 按文件系统算还在占 → `lsof +L1` 确认，重启进程或 truncate 释放
- **原因二：文件系统预留块**：ext4 默认预留 5% 给 root，df 可用数会比实际小（tune2fs -m 2 可调）
- **原因三：挂载点覆盖**：目录挂载了新盘，但底下藏着旧文件——du 看不见被覆盖层；排查：`mount | grep` 对照，或单用户模式不挂载时 du
- **原因四**：稀疏文件（du 按实际块算，df 按已分配算）、硬链接重复计数方向的不一致
- 定位顺序：`lsof +L1` → `tune2fs -l` 看预留 → 核对挂载点

**一句话答：** 不一致八成是"删了但没释放"——文件被进程占着句柄，du 已看不见而 df 还在算，lsof +L1 一查便知。其他可能：ext4 默认预留 5% 空间给 root 造成偏差，以及挂载点下藏着被覆盖的旧文件。排查就按这三个方向走。

### 97. 简述 Linux 读取和写入文件的流程？

**要点：**
- 读：进程 `read()` 系统调用 → 先查**页缓存**（命中直接返回）→ 未命中向块层发 IO 请求 → 文件系统定位 inode→block 映射 → 通用块层/IO 调度 → 磁盘驱动 → 磁盘 → 数据回填页缓存 → 拷贝到用户缓冲区
- 写：`write()` → 写入页缓存即返回（**延迟写**）→ 内核后台回写线程（writeback，受 dirty_ratio 控制）批量刷盘；`fsync()` 强制落盘
- VFS 是中间层：同一套系统调用适配 ext4/xfs/NFS 等不同后端
- 性能要点：读靠缓存命中，写靠批量回写；数据库绕开这层用 O_DIRECT

**一句话答：** 读的路径：先查页缓存，命中直接回；没命中经 VFS 和文件系统定位到数据块，走块层驱动从磁盘读上来缓存再给用户。写的路径：写进页缓存就返回——这叫延迟写，内核按 dirty 阈值后台批量刷盘，要可靠落盘必须 fsync。这套缓存机制就是 buffer/cache 大小波动的来源。

### 98. Linux 中 CPU 是怎么工作的？

**要点：**
- CPU 按时钟节拍执行指令：**取指 → 译码 → 执行**的流水线循环
- **多任务**：靠内核**时间片轮转调度**（CFS 完全公平调度器）——每个任务跑几毫秒被时钟中断打断，切换上下文，宏观上"同时"
- **特权分级**：用户态（ring3）跑应用，内核态（ring0）跑内核；系统调用触发模式切换
- **中断**：硬件事件（网卡来包、磁盘完成）打断 CPU 当前工作转去处理——软中断 ksoftirqd 在 top 里 %si 可见
- 多核：每核独立流水线，共享内存 + 缓存一致性协议（MESI）

**一句话答：** 单颗 CPU 看似并行实则轮流：调度器按时间片切换任务，切换时保存恢复上下文。它有两种工作模式——用户态跑应用、内核态处理系统调用和中断；top 里的 us/sy/wa/si 分别对应这几种时间的占比。理解这些就能把 CPU 指标和真实工作对应起来。

### 99. 如果你提了一个 Bug，开发不认为是 Bug，你会怎么处理？

**要点：**
- **先自证**：确认现象可复现、影响面清楚、有日志/截图/最小复现步骤——很多"Bug 争议"是证据不足
- **对齐标准**：区分 Bug / 需求分歧 / 预期设计（文档里写了的行为）——拿官方文档、历史版本行为、行业惯例佐证
- **数据说话**：用影响范围和业务风险量化（多少用户、什么损失、违反什么 SLA）
- **升级路径**：拉产品/测试共同评审 → 仍无法一致则按团队流程升级到主管/TechLead 裁决——**对事不对人**
- 心态：目标是让产品变好，不是争输赢；真不是 Bug 就大方承认并补知识

**一句话答：** 我会先把自己的证据做扎实：可复现步骤、日志、影响面，很多争议其实是证据不足。然后用数据和文档跟开发对齐"这到底是缺陷还是设计"，拉上测试和产品一起评审；实在定不了就按流程升级到技术负责人裁决，对事不对人。如果最后证明是我的理解错了，我也大方认——目标是把事弄对，不是赢。

### 100. Linux 服务器内存明明够用，为什么还触发了 OOM？

**要点：**
- 常见原因：
  1. **瞬时峰值**：free 是历史快照，OOM 发生在毫秒级峰值——分配大块内存的瞬间（如读大文件、fork 大进程、JVM 扩堆）恰好无物理页可用
  2. **cgroup/容器限额**：容器 memory limit 打满就 OOM，**主机内存充足也没用**（`docker inspect` 看 OOMKilled；K8s 里 limit 太低）
  3. **overcommit 副作用**：内核允许超额承诺内存（vm.overcommit_memory=0），多个进程同时真用起来就超
  4. 32 位进程地址空间限制、NUMA 单节点耗尽（`numactl --hardware`，dmesg 提示 "cpuset/NUMA"）
- 排查：dmesg OOM 记录里的内存明细 → 对照监控曲线找峰值 → 确认是主机级还是 cgroup 级（dmesg 里 "Task in ... killed as a result of memory limit"= 容器限额）

**一句话答：** "内存够用"多半是看 free 的错觉：OOM 是瞬时事件，峰值那一刻可能就是不够——而且如果进程跑在容器里，超过的是 cgroup 的 memory limit，跟主机内存够不够完全无关。dmesg 里 OOM 日志会写明是全局还是 cgroup 限额触发，对照监控曲线基本都能对出峰值来源。

### 101. Linux 目录下有 100 万个文件，如何快速删除？

**要点：**
- 慢的原因：`rm *` 或 `find | xargs rm` 对**每个文件单独 unlink 系统调用**，且 shell 展开 `*` 会参数列表爆炸（Argument list too long）
- 快的方案（由快到普适）：
  1. **直接删整个目录重建**：`rm -rf dir && mkdir dir`（删目录树本身比逐个 unlink 快得多）——前提是目录可重建、句柄无依赖
  2. **rsync 空目录覆盖**：`rsync -a --delete empty/ dir/`（实践中极快，且目录本体保留）
  3. `find dir -type f -delete`（比 find|xargs rm 省一层进程）
- 期间注意 IO 压力（ionice）、监控先摘告警

**一句话答：** rm 慢是因为一百万次 unlink 系统调用。最快的做法是整个目录删掉重建，或者 rsync 一个空目录加 --delete 过去——实测比逐个删快一个数量级。保目录结构的场景用 find -delete，千万别用 rm * 那种 shell 展开，文件一多直接参数列表爆掉。

### 102. Linux 系统文件句柄泄露怎么排查？

**要点：**
- **确认泄露**：`ls /proc/<pid>/fd | wc -l` 周期采样看是否单调上涨不回落；对监 nginx 的 worker、Java 进程看打开数 vs `ulimit -n`、`fs.file-max`
- **定位泄露类型**：`ls -l /proc/<pid>/fd | awk '{print $NF}' | grep -o '^.*type' | sort | uniq -c | sort -rn`——看 fd 都是什么：
  - 大量 **socket**（`ss -tnp` 对应看连接状态）→ 连接池没释放/下游超时不关闭
  - 大量 **文件/日志** → 打开不关闭
  - 大量 **anonymous/inode**（eventfd、epoll）→ 框架级泄露
- 对嫌疑连接：`ls -l /proc/<pid>/fd | grep socket` 拿 inode 号 → `ss -e | grep <inode>` 定位具体连接（对端 IP 端口）
- 处理：临时 `ulimit -n` 调大 + 重启进程恢复；根治改代码（连接池、try-with-resources、defer close）

**一句话答：** 先周期性数 /proc/PID/fd 确认是否单调涨；然后统计 fd 的构成：全是 socket 就顺着 fd 的 inode 号用 ss 定位到具体连接，看是不是连接池不释放或下游超时不关；全是普通文件就是打开不关闭。线上先调大 ulimit 重启进程止血，根治要开发改代码。

### 103. ~/.bash_profile、~/.bashrc、/etc/profile 作用及加载顺序？

**要点：**
- 登录 shell（ssh 登录、su -）：读 `/etc/profile` → `~/.bash_profile`（若存在；否则试 ~/.bash_login、~/.profile）——bash_profile 内通常**手动 source ~/.bashrc**
- 交互非登录 shell（已登录后新开终端 tab）：只读 `/etc/bash.bashrc` + `~/.bashrc`
- /etc/profile 系统级全局（所有用户），~ 下用户级
- 实用推论：**环境变量给所有会话**写 ~/.bash_profile 或 /etc/profile；**别名/提示符**写 ~/.bashrc；改完 source 生效或重登
- 验证：`bash -l -c 'echo $PATH'` vs `bash -c 'echo $PATH'` 对比差异

**一句话答：** 区别在 shell 类型：登录 shell 走 /etc/profile 再 ~/.bash_profile；非登录的交互 shell 只读 ~/.bashrc。通常 bash_profile 里会 source bashrc 把两边打通。所以系统级变量放 /etc/profile，个人环境变量放 bash_profile，别名提示符放 bashrc——改完记得 source 或重开终端。

## Shell 脚本编程面试题

### 1. 判断 192.168.1.0/24 网段哪些 IP 在线？

**要点：**
- 并发 ping + 后台任务，2 秒超时，统计存活
- 改进方向：fping（`fping -aq -g 192.168.1.0/24` 一行搞定）、nmap -sn

```bash
#!/bin/bash
for i in $(seq 1 254); do
  {
    ip="192.168.1.$i"
    if ping -c1 -W2 "$ip" &>/dev/null; then
      echo "$ip 在线"
    fi
  } &
done
wait
```

**一句话答：** 核心是循环 ping 加并发：每个 IP 用 & 扔后台、-W 设超时，最后 wait 等齐。生产上我更直接用 fping -g 整个网段一行搞定，或 nmap -sn 扫存活。

### 2. 批量创建 100 个系统用户？

**要点：**
- useradd + chpasswd；随机密码用 `/dev/urandom` 或 openssl；密码入库要 chmod 600

```bash
#!/bin/bash
for i in $(seq 1 100); do
  user="user$i"
  if ! id "$user" &>/dev/null; then
    useradd -m "$user"
    pass=$(openssl rand -base64 12)
    echo "$user:$pass" | chpasswd
    echo "$user:$pass" >> /root/newusers.txt
  fi
done
chmod 600 /root/newusers.txt
```

**一句话答：** 循环 useradd -m 建用户，随机密码 openssl rand 生成后用 chpasswd 设置，账号密码落文件并 chmod 600。加分点：先 id 判断用户存在避免重复创建。

### 3. 监控多个网站域名可用性？

**要点：**
- curl -s -o /dev/null -w '%{http_code}' 取状态码 + --max-time 超时；失败重试告警（mail/钉钉 webhook）
- state 文件防告警风暴（首次失败不报、连续 N 次才报）

```bash
#!/bin/bash
sites="https://www.a.com https://api.b.com"
for url in $sites; do
  code=$(curl -s -o /dev/null -w '%{http_code}' --max-time 5 "$url")
  if [[ "$code" != "200" && "$code" != "30"* ]]; then
    echo "$(date) $url 异常 code=$code" >> /var/log/site_monitor.log
    # 钉钉/邮件告警
  fi
done
```

**一句话答：** 循环用 curl 取 HTTP 状态码，非 2xx/3xx 记日志并发告警。关键细节：--max-time 防止 curl 卡死，告警前做连续失败判断防抖，避免网络抖动一秒就把人吵醒。

### 4. 找出占用 CPU/内存 过高的进程？

**要点：**
- ps 排序：`ps aux --sort=-%cpu | head`、`--sort=-%mem`；head 取前 N
- 配合阈值判断，超阈值记录或告警

```bash
#!/bin/bash
echo "== CPU TOP5 =="
ps aux --sort=-%cpu | head -6
echo "== MEM TOP5 =="
ps aux --sort=-%mem | head -6
# 阈值告警版
cpu=$(ps aux --sort=-%cpu | awk 'NR==2{print int($3)}')
[[ $cpu -gt 80 ]] && echo "CPU告警: $cpu%" | mail -s "cpu" ops@x.com
```

**一句话答：** 一行 ps aux --sort=-%cpu 接 head 就是榜单；要告警就 awk 取第一名的值和阈值比。内存同理换 --sort=-%mem。

### 5. 分析 Nginx 访问日志？

**要点：**
- 常规统计：PV、UV、TOP IP、TOP URL、状态码分布——awk 数组统计

```bash
#!/bin/bash
log=/var/log/nginx/access.log
echo "== TOP10 IP =="
awk '{print $1}' $log | sort | uniq -c | sort -rn | head
echo "== TOP10 URL =="
awk '{print $7}' $log | sort | uniq -c | sort -rn | head
echo "== 状态码分布 =="
awk '{print $9}' $log | sort | uniq -c | sort -rn
echo "== UV =="
awk '{print $1}' $log | sort -u | wc -l
```

**一句话答：** awk 按列取字段加 sort uniq count 三连：第一列是 IP 统计攻击源，第七列 URL 看热点，第九列状态码看错误率。大日志上量后再上 GoAccess 或 ELK，脚本适合快速临时分析。

### 6. 监控多台服务器磁盘利用率？

**要点：**
- ssh 批量执行 df；主机列表数组；阈值判断；免密（ssh-keygen + ssh-copy-id）前提

```bash
#!/bin/bash
hosts="10.0.0.1 10.0.0.2 10.0.0.3"
for h in $hosts; do
  usage=$(ssh -o ConnectTimeout=5 "$h" "df -h / | awk 'NR==2{print int(\$5)}'")
  if [[ $usage -gt 80 ]]; then
    echo "$h 磁盘 ${usage}%" >> /tmp/disk_alert.log
  fi
done
```

**一句话答：** 前提是配好免密，循环 ssh 各机取 df 根分区的百分比，超 80 记录告警。细节：ConnectTimeout 防止某台宕机拖死整个脚本，远程命令里的 $5 要转义。规模大了就交给 Zabbix/Prometheus，脚本适合几十台以内。

### 7. 监控 MySQL 主从同步状态？

**要点：**
- 核心两个指标：Slave_IO_Running、Slave_SQL_Running 都为 Yes，且 Seconds_Behind_Master 小

```bash
#!/bin/bash
result=$(mysql -u monitor -p'xxx' -e "show slave status\G" | awk '/Slave_(IO|SQL)_Running/{print $2}')
ok=$(echo "$result" | grep -c Yes)
behind=$(mysql -u monitor -p'xxx' -e "show slave status\G" | awk '/Seconds_Behind_Master/{print $2}')
if [[ $ok -ne 2 || $behind -gt 60 ]]; then
  echo "主从异常: running=$result behind=${behind}s" | mail -s "mysql_rep" ops@x.com
fi
```

**一句话答：** show slave status 里抓三个值：IO 和 SQL 线程必须都是 Yes，Seconds_Behind_Master 超过阈值也告警——两个 Yes 只代表复制链路活着，延迟大照样有问题。

### 8. 自动备份 MySQL 中多个库？

**要点：**
- mysqldump 逐库导出 + gzip；日期命名；保留 N 天（find -mtime +7 删）；全量+binlog 才是完整方案

```bash
#!/bin/bash
user=root; pass='xxx'; dest=/backup/mysql
keep=7
mkdir -p $dest
for db in $(mysql -u$user -p$pass -e "show databases" | awk 'NR>1 && $1!~/^(information_schema|performance_schema|sys)$/'); do
  mysqldump -u$user -p$pass --single-transaction --master-data=2 "$db" | gzip > "$dest/${db}_$(date +%F).sql.gz"
done
find $dest -name "*.sql.gz" -mtime +$keep -delete
```

**一句话答：** show databases 拿列表循环 mysqldump，加 --single-transaction 保证 InnoDB 一致性快照不锁表，管道 gzip 压缩，文件名带日期，最后 find -mtime 删七天前的旧备份。还要强调：备份必须定期做恢复演练，不然等于没备。

### 9. 屏蔽异常频繁访问网站的 IP？

**要点：**
- 日志统计单 IP 访问频率 → 超阈值写入防火墙黑名单
- 注意白名单（自己/CDN/监控）；fail2ban 是成熟替代

```bash
#!/bin/bash
log=/var/log/nginx/access.log
limit=100
whitelist="10.0.0.1 192.168.1.100"
awk '{print $1}' $log | sort | uniq -c | sort -rn | awk -v L=$limit '$1>L{print $2}' | while read ip; do
  [[ " $whitelist " == *" $ip "* ]] && continue
  if ! iptables -C INPUT -s "$ip" -j DROP 2>/dev/null; then
    iptables -I INPUT -s "$ip" -j DROP
    echo "$(date) 封禁 $ip" >> /var/log/block_ip.log
  fi
done
```

**一句话答：** 思路是日志统计单 IP 请求数，超阈值的写进 iptables DROP 规则，先查规则存在防重复封。两个必答细节：白名单保护自己人和 CDN 回源 IP，以及这活有现成的 fail2ban 干得更专业——手动脚本是面试考点，生产用工具。

### 10. 批量修改文件名？

**要点：**
- for + 变量替换/重命名；`rename` 工具一行流；备份式修改（先 dry-run）

```bash
#!/bin/bash
# 把 .txt 改成 .log
for f in *.txt; do
  mv "$f" "${f%.txt}.log"
done
# 加前缀
for f in *.jpg; do
  mv "$f" "backup_$f"
done
# rename 一行流: rename 's/\.txt$/.log/' *.txt
```

**一句话答：** 基本功是 bash 变量替换：${f%.txt}.log 去后缀、${f#pre} 去前缀，配合 for 和 mv。系统有 rename 命令的话一条正则搞定。批量改名的纪律是先 echo dry-run 看一眼再真改。

### 11. 统计指定目录下文件总大小？

**要点：**
- du -sh 目录（人读）；`find -type f -exec stat` 精确统计；按大小排序 top

```bash
#!/bin/bash
dir=${1:-.}
echo "总大小: $(du -sh "$dir" | awk '{print $1}')"
echo "== TOP10 大文件 =="
find "$dir" -type f -exec du -h {} + 2>/dev/null | sort -rh | head -10
```

**一句话答：** 一行 du -sh 就是答案；展开能力是 find -type f 配 du 拿到每个文件大小再 sort -rh 排序出 top 榜——找大文件清理时天天用。

### 12. 监控服务运行状态？

**要点：**
- systemctl is-active 判断；down 则尝试拉起并告警；死亡时间记录

```bash
#!/bin/bash
services=("nginx" "redis" "mysqld")
for s in "${services[@]}"; do
  if ! systemctl is-active --quiet "$s"; then
    echo "$(date) $s 已停止, 尝试拉起" >> /var/log/svc_monitor.log
    systemctl start "$s"
    if systemctl is-active --quiet "$s"; then
      echo "$(date) $s 拉起成功" >> /var/log/svc_monitor.log
    else
      echo "$(date) $s 拉起失败!" >> /var/log/svc_monitor.log
      # 告警
    fi
  fi
done
```

**一句话答：** systemctl is-active --quiet 判断状态，挂了就 start 拉起并把动作写日志，拉不起来升级告警。面试加分：指出这种自愈脚本该配 cron 高频跑，且更正规的做法是 systemd 的 Restart=on-failure 让 systemd 自己干。

### 13. 计算目录中指定文件的总行数？

**要点：**
- find + wc -l；xargs 处理空格文件名（-print0/-0）

```bash
#!/bin/bash
dir=${1:-.}          # 目录
pattern=${2:-*.log}  # 文件模式
total=0
while IFS= read -r -d '' f; do
  n=$(wc -l < "$f")
  total=$((total + n))
done < <(find "$dir" -type f -name "$pattern" -print0)
echo "$dir 下 $pattern 总行数: $total"
```

**一句话答：** find 找出文件逐个 wc -l 累加；细节是 find -print0 配 read -d '' 处理带空格的文件名。快速版直接 find -name '*.log' | xargs wc -l | tail -1。

### 14. 监控系统关键文件完整性？

**要点：**
- 思路：基线 md5/sha256 存档 → 周期对比 → 有变化告警；成熟方案 AIDE/Tripwire

```bash
#!/bin/bash
files="/etc/passwd /etc/shadow /etc/ssh/sshd_config /etc/crontab"
base=/var/lib/.filecheck.md5
if [[ ! -f $base ]]; then
  md5sum $files > $base
  echo "初始化基线"
  exit 0
fi
if ! md5sum -c --quiet $base; then
  echo "$(date) 关键文件变更!" >> /var/log/file_monitor.log
  md5sum $files > $base   # 确认合法变更后更新基线
fi
```

**一句话答：** md5sum 先做基线快照，定时 md5sum -c 对比，变了就告警——告警人工确认是正常变更后再更新基线。生产级用 AIDE，原理相同但带数据库和更全的属性检查。

### 15. 自动发布 Java 项目到 Tomcat？

**要点：**
- 流程：拉包（maven/制品库）→ 备份旧包 → 停服务 → 替换 war → 起服务 → 健康检查 → 失败回滚

```bash
#!/bin/bash
app=myapp; tomcat=/usr/local/tomcat; pkg=$1
[[ -z $pkg ]] && { echo "用法: $0 <war包路径>"; exit 1; }
# 备份
cp $tomcat/webapps/$app.war /backup/$app.war.$(date +%F_%H%M) 2>/dev/null
# 停 -> 替换 -> 起
systemctl stop tomcat
rm -rf $tomcat/webapps/$app $tomcat/webapps/$app.war
cp "$pkg" $tomcat/webapps/$app.war
systemctl start tomcat
# 健康检查
for i in $(seq 1 30); do
  code=$(curl -s -o /dev/null -w '%{http_code}' http://localhost:8080/$app/health)
  [[ $code == 200 ]] && { echo "发布成功"; exit 0; }
  sleep 2
done
echo "健康检查失败, 回滚"
systemctl stop tomcat
cp /backup/$app.war.$(date +%F)_* $tomcat/webapps/$app.war 2>/dev/null
systemctl start tomcat
```

**一句话答：** 骨架是"备份-停-换-起-验"五步：旧包按时间戳备份，停 Tomcat 换 war 再启动，curl 探活循环检查，失败自动用备份回滚。要点强调健康检查和回滚必须有，这也是 CI/CD 流水线里发布步骤的原型。

### 16. 监控目录文件变化并实时同步？

**要点：**
- inotifywait 监听事件 → 触发 rsync；或 lsyncd 现成方案

```bash
#!/bin/bash
src=/data/www/
dst=backup@10.0.0.2::www
inotifywait -mrq -e modify,create,delete,move "$src" | while read line; do
  rsync -az --delete "$src" "$dst"
  echo "$(date) 同步: $line"
done
```

**一句话答：** inotifywait -m 持续监听 modify/create/delete 事件，管道 while 循环里触发 rsync --delete 整体同步。生产我会直接用 lsyncd——它就是这套逻辑的成品封装，带去抖和并发控制。记得配 SSH 免密。

### 17. 一键部署 LNMP 网站环境？

**要点：**
- 要点：判断系统版本用对应包管理器、每步校验失败退出（set -e）、配置文件模板化

```bash
#!/bin/bash
set -e  # 任一步失败立即退出
# nginx
yum install -y epel-release && yum install -y nginx
# php
yum install -y php php-fpm php-mysqlnd
# mysql
yum install -y mariadb-server
# 配置: nginx 转发 php 给 fpm
cat > /etc/nginx/conf.d/lamp.conf <<'EOF'
server {
    listen 80;
    root /var/www/html;
    index index.php;
    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
EOF
systemctl enable --now nginx php-fpm mariadb
echo "LNMP 部署完成"
```

**一句话答：** 脚本要点三条：set -e 让任何一步失败就停，别带着错误继续跑；包管理按系统分支（yum/apt）；服务统一 systemctl enable --now。真正生产不这么装——我会用 Ansible playbook 或 Docker Compose，脚本是理解部署流程的入门。

### 18. 一键采集 Linux 服务器信息？

**要点：**
- 采集维度：主机名/IP/系统版本/内核/CPU/内存/磁盘/网卡/运行时长

```bash
#!/bin/bash
echo "主机名: $(hostname)"
echo "IP: $(hostname -I)"
echo "系统: $(cat /etc/os-release | grep PRETTY | cut -d'"' -f2)"
echo "内核: $(uname -r)"
echo "CPU核数: $(nproc)"
echo "内存: $(free -h | awk '/Mem/{print $2}')"
echo "磁盘: $(lsblk -d -o NAME,SIZE | grep -v loop)"
echo "运行时长: $(uptime -p)"
echo "负载: $(uptime | awk -F'load average:' '{print $2}')"
```

**一句话答：** 就是把 lscpu、free、lsblk、uname 这些一条条组织起来，重点在输出格式化成资产表——配合 ansible setup 模块能批量拿全 Fleet 的信息，脚本版适合单机快速盘点。

### 19. 服务器端口扫描脚本？

**要点：**
- /dev/tcp 伪设备探活（bash 内置免依赖）或 nc -z；超时控制

```bash
#!/bin/bash
host=${1:-127.0.0.1}
ports="22 80 443 3306 6379 8080"
for p in $ports; do
  if timeout 2 bash -c "echo >/dev/tcp/$host/$p" 2>/dev/null; then
    echo "$host:$p 开放"
  fi
done
```

**一句话答：** 用 bash 内置的 /dev/tcp 伪设备加 timeout 探测，免装依赖；nc -zv 也一样。范围扫描换 nmap -p-。强调只扫自己的资产，别去扫别人的网。

### 20. 从 FTP 服务器自动下载文件？

**要点：**
- wget/curl 支持 ftp；lftp 脚本化更强；凭证别硬编码（~/.netrc 或环境变量）

```bash
#!/bin/bash
server=10.0.0.5
user=backup; pass='xxx'
# wget 方式
wget -r -nH --cut-dirs=1 "ftp://$user:$pass@$server/backup/" -P /data/ftp_backup
# lftp 镜像方式(推荐)
lftp -u "$user,$pass" "$server" <<EOF
mirror --continue /backup /data/ftp_backup
bye
EOF
```

**一句话答：** 简单场景 wget 直接拉 ftp 目录，增量断点用 lftp 的 mirror --continue 更稳。密码管理用 ~/.netrc 而不是明文写在命令行里——命令行密码会被 ps 看到也算安全点。

### 21. 批量修改服务器账号密码？

**要点：**
- chpasswd 或 passwd --stdin（RHEL）；随机密码生成与分发；失败重试记录

```bash
#!/bin/bash
hosts="10.0.0.1 10.0.0.2"
user=root
for h in $hosts; do
  newpass=$(openssl rand -base64 16)
  if ssh "$h" "echo '$user:$newpass' | chpasswd"; then
    echo "$h:$newpass" >> /root/passwords_$(date +%F).txt
  else
    echo "$h 修改失败" >> /root/chpass_fail.log
  fi
done
chmod 600 /root/passwords_*.txt
```

**一句话答：** 循环 ssh 用 chpasswd 改密，新密码 openssl 随机生成后落到加密管理的台账并 chmod 600。面试要点是说出风险：静态密码轮转正被密钥+堡垒机替代，密码文件要进 Vault 或密码管理器，不能落明文。

### 22. 解释 $0、$1、$#、$?、$$、$*、$@ 的含义？

**要点：**
- `$0` 脚本名；`$1-$n` 位置参数（$1 第一个参数）；`$#` 参数个数
- `$?` 上一命令退出码（0 成功）；`$$` 当前脚本 PID；`$!` 最近后台任务 PID
- `$*` 所有参数合成**一个**字符串；`$@` 每个参数**独立**字符串（加引号时区别明显："a b" 两个词，$* 变一个，"$@" 仍是两个）

**一句话答：** 前五个背定义：$0 脚本名、$1 起是位置参数、$# 个数、$? 上条命令退出码、$$ 自己的 PID。真正的考点是 $* 和 $@ 的区别：带引号时 "$@" 保持参数边界，"$*" 粘成一个参数——遍历参数永远用 "$@"。

### 23. 单引号 ' '、双引号 " "、反引号 ` 有什么区别？

**要点：**
- **单引号**：完全字面量，里面 `$x` 不展开
- **双引号**：变量展开（"$x"）、命令替换保留，但抑制通配符和单词分割
- **反引号**（及推荐的 `$( )`）：命令替换，取命令输出；嵌套时 $() 可读可递归，反引号不行
- 实战：字符串含变量用双引号；正则/含 $ 的字面串用单引号；命令替换一律 $()

```bash
x="world"
echo 'hello $x'    # hello $x  (不展开)
echo "hello $x"    # hello world (展开)
echo "today: $(date +%F)"   # 命令替换(推荐写法)
```

**一句话答：** 单引号一切原样输出，双引号会展开变量但阻止分词，反引号是命令替换——但现代写法都用 $( ) 因为可读且能嵌套。口诀：要展开用双引号，要字面用单引号，要执行用 $()。

## 网络运维面试题

### 1. 简述 OSI 七层参考模型？

**要点：**
- 自下而上：**物理层**（比特/线缆电平）→ **数据链路层**（帧/MAC/交换机）→ **网络层**（包/IP/路由器）→ **传输层**（段/TCP UDP/端口）→ **会话层**（会话建立管理）→ **表示层**（编码加密压缩）→ **应用层**（HTTP/DNS 接口）
- 记法：物链网传会表应；每层记一个代表设备/协议
- 实战价值：分层定位故障——物理层看灯、链路层 ping 网关、网络层 traceroute、传输层 telnet 端口、应用层看响应

**一句话答：** 七层从下到上：物理、数据链路、网络、传输、会话、表示、应用，分别对应比特、MAC 帧、IP 包、TCP/UDP 段直到 HTTP。面试不用说全，重点是能拿它当排障地图：先看物理层链路通不通，再逐层往上缩小范围。

### 2. 简述 TCP/IP 四层模型？与 OSI 七层模型的区别？

**要点：**
- 四层：**网络接口层**（OSI 的物理+链路）→ **网络层**（IP/ICMP）→ **传输层**（TCP/UDP）→ **应用层**（HTTP/DNS/SSH，合并了会话+表示+应用）
- 区别：OSI 是 ISO 的理论参考模型（先有模型后有协议）；TCP/IP 是事实标准（先有协议后被归纳成模型）——工业界用四层，教学用七层
- 对应关系：四层模型每层都能在七层里找到投影

**一句话答：** TCP/IP 四层是网络接口、网络、传输、应用，把 OSI 上三层合并成应用层、下两层合并成网络接口层。区别一句话：OSI 是理论参考先有模型，TCP/IP 是工程实践先有协议——实际工作按四层思维排查，面试按七层背概念。

### 3. 简述 TCP 三次握手的过程？

**要点：**
- **SYN → SYN+ACK → ACK**：
  1. 客户端发 **SYN**（seq=x）进入 SYN_SENT
  2. 服务端回 **SYN+ACK**（seq=y, ack=x+1）进入 SYN_RCVD
  3. 客户端回 **ACK**（ack=y+1）双方 ESTABLISHED
- **为什么三次不是两次**：两次无法确认"客户端能收到服务端的消息"（服务端不知道自己的 SYN+ACK 是否到达），也无法防止**历史重复 SYN** 建立错误连接（RFC793 提出的失效连接问题）
- 队列：全连接队列（somaxconn/accept queue）+ 半连接队列（syn backlog）；SYN Flood 攻击打的就是半连接
- 抓包验证：`tcpdump -i any port 80 'tcp[tcpflags] & (tcp-syn) != 0'`

**一句话答：** 客户端发 SYN，服务端回 SYN+ACK，客户端再回 ACK，三次之后双方都进入 ESTABLISHED。为什么是三次：前两次只能证明客户端到服务端通，第三次 ACK 才让服务端确认"客户端能收到我的包"，同时防止网络上滞留的旧 SYN 建立错误连接。配套要懂两个队列：半连接队列被 SYN Flood 打爆，全连接队列被 somaxconn 限制——高并发服务必调。

### 4. TCP 有哪些状态，分别是什么意思？

**要点：**
- 建立连接：**LISTEN**（监听）→ **SYN_SENT**（客户端已发 SYN）→ **SYN_RCVD**（服务端收到 SYN）→ **ESTABLISHED**（建立完成）
- 关闭连接：**FIN_WAIT_1/2**（主动方等对方确认/等对方 FIN）→ **CLOSE_WAIT**（被动方收到 FIN，等我方处理完关）→ **LAST_ACK**（被动方发出 FIN 等最后 ACK）→ **TIME_WAIT**（主动方最后等待 2MSL）→ **CLOSED**
- 运维解读：
  - 大量 **CLOSE_WAIT** = 我方代码收到关闭后没调用 close()（连接泄漏，代码 bug）
  - 大量 **TIME_WAIT** = 我方高频主动关闭短连接（架构问题）
  - 大量 **SYN_RECV** = 可能被 SYN Flood 或 accept 处理不过来
- 查看：`ss -ant state established | wc -l`、`ss -ant | awk '{print $1}' | sort | uniq -c`

**一句话答：** 状态机分两段背：建连接 LISTEN、SYN_SENT、SYN_RCVD 到 ESTABLISHED；断连接经过 FIN_WAIT、CLOSE_WAIT、LAST_ACK 到 TIME_WAIT。运维最关注三个：CLOSE_WAIT 堆积是代码没 close 的 bug，TIME_WAIT 堆积是短连接太多，SYN_RECV 堆积是半连接队列被打——这三个状态分布一看，问题方向基本就定了。

### 5. 简述 TCP 四次挥手的过程？

**要点：**
- 主动方 **FIN**（FIN_WAIT_1）→ 被动方 **ACK**（CLOSE_WAIT，主动方到 FIN_WAIT_2）→ 被动方处理完数据发 **FIN**（LAST_ACK）→ 主动方 **ACK**（TIME_WAIT，等 2MSL 后 CLOSED）
- **为什么四次**：TCP 全双工，两个方向各关各的；被动方收到 FIN 时可能还有数据没发完，所以 ACK 和 FIN 通常分开发（若正好没数据可合并，出现三次挥手形态）
- **TIME_WAIT 为什么等 2MSL**：① 保证最后一个 ACK 丢失时能重传接收（被动方会重发 FIN）② 让本连接的旧报文在网络中自然消亡，避免污染新连接
- 大量 FIN_WAIT_2：对端一直不关（应用层没处理完）

**一句话答：** 四次挥手是两个方向各关一次：FIN、ACK、FIN、ACK。多出来一次是因为收到 FIN 的一方可能还有数据要发，ACK 和 FIN 分开。主动关闭方最后等 2MSL 再彻底关闭——一是防最后的 ACK 丢了对方重发 FIN 时没人应答，二是让旧报文自然消亡不污染新连接。这就是 TIME_WAIT 的由来。

### 6. TCP 与 UDP 的主要区别及适用场景？

**要点：**
- **TCP**：面向连接（三次握手）、可靠（确认重传）、有序（序号排序）、字节流（无消息边界）、拥塞控制；开销大延迟高
- **UDP**：无连接、尽力而为不保证可靠有序、面向报文（有边界）、无拥塞控制；快开销小
- 场景：TCP——HTTP/SSH/数据库/文件传输（要可靠）；UDP——DNS（一问一答快）、视频直播/语音（要实时宁可丢帧）、NTP、游戏；QUIC 在 UDP 上重建可靠性（HTTP/3 底座）
- "UDP 不可靠"可补：需要可靠时应用层自己补（KCP/QUIC 思路）

**一句话答：** TCP 是打电话——先建连接、句句确认、按序到达；UDP 是寄明信片——直接发、不保证到。场景分法：要完整正确走 TCP，网页、SSH、数据库；要实时低延迟走 UDP，直播、语音、DNS。现在 HTTP/3 的 QUIC 就是在 UDP 上应用层重建可靠性和加密，鱼和熊掌兼得的尝试。

### 7. SMTP 协议有什么作用？

**要点：**
- **Simple Mail Transfer Protocol**：发送和中转**邮件**的应用层协议，TCP 25 端口（提交 587，SSL 465）
- 流程：MUA（客户端）→ MTA（服务器）逐跳转发，按收件人域名的 **MX 记录**投递
- 命令式交互：HELO/EHLO、MAIL FROM、RCPT TO、DATA
- 运维关联：自建邮件服务要配反向解析（PTR）、SPF/DKIM/DMARC 防被当垃圾邮件；25 端口云商普遍默认封禁

**一句话答：** SMTP 是发信协议，管邮件从客户端到服务器、服务器到服务器的逐跳转发，投递时靠 DNS 的 MX 记录找到对方邮件服务器。运维视角它就是个 TCP 25 端口的文本协议，自建邮箱麻烦在反垃圾体系——PTR、SPF、DKIM 一套都得配。

### 8. SNMP 协议有什么作用？

**要点：**
- **Simple Network Management Protocol**：网络设备（交换机/路由器/防火墙/打印机）的**监控与管理**协议，UDP 161（查询）/162（Trap 告警）
- 结构：管理站（NMS）+ Agent（设备端）+ **MIB**（管理信息库，OID 树状编号）+ Community（v1/v2c 弱认证字符串）
- 两种取数方式：轮询（NMS 周期 get 指标）+ **Trap**（设备主动上报故障）
- 版本：v1/v2c 明文弱安全；**v3 加认证加密**（生产必用）
- 实战：Zabbix/PRRTG 监控交换机端口流量就是走 SNMP 采集 OID（如接口流量 ifHCInOctets）

**一句话答：** SNMP 是网络设备的标准监控协议：管理站按 OID 从设备的 MIB 里取数，端口流量、CPU 温度都是树上的节点；设备出事还能主动发 Trap。轮询走 UDP 161、Trap 走 162。要点是版本安全——v2c 的 community 是明文，生产必须上 v3 的认证加密。

### 9. VPN 工作原理及应用场景？

**要点：**
- 原理：在公网上建立**加密隧道**，把私网流量封装在隧道里传输——认证、加密、封装三层保障
- 常见协议：**IPSec**（网络层，站点到站点标准）、**SSL VPN/OpenVPN**（传输层，远程接入）、**WireGuard**（内核态，现代高性能）、L2TP/PPTP（PPTP 已不安全）
- 场景：**远程办公**（员工接入公司内网）、**站点互联**（分支机构与总部机房打通）、**云上混合云**（VPC 与 IDC 打通）、跨区域服务访问
- 运维要点：路由/防火墙放行隧道端口、子网不能重叠（重叠要 NAT 处理）、密钥定期轮换

**一句话答：** VPN 就是公网上的加密隧道：认证身份、加密数据、封装转发。两大场景——个人远程接入公司内网用 SSL VPN 或 WireGuard，机构对机构打通机房用 IPSec 站点隧道。运维注意隧道两端子网不能重叠，路由和防火墙要放行。

### 10. 简述 ping 命令工作原理？

**要点：**
- 基于 **ICMP**（网络层协议，无端口）：发送 **ICMP Echo Request**（type 8），对方回 **Echo Reply**（type 0）
- 过程：构造 ICMP 包 → 交给 IP 层按目标路由发送 → 对方内核协议栈直接回包（不经过应用）→ 本地统计往返时间（RTT）和丢包
- 输出解读：time=RTT、ttl（每过一个路由器减 1，可推测路径跳数与对方 OS）、丢包率
- 限制：防火墙可禁 ICMP（ping 不通≠服务不可用，telnet 端口才是服务级验证）；ping 走不了代理
- mtr = ping + traceroute 结合（每跳丢包率/延迟持续统计）

**一句话答：** ping 发的是 ICMP Echo 请求，对方协议栈直接回 Echo 应答，全程不涉及端口和应用层，用它测 RTT 和丢包。两个实用认知：ping 不通不代表服务挂了——对方可能禁了 ICMP，验证服务要 telnet 端口；想看沿途哪一跳丢包就用 mtr，它是 ping 和 traceroute 的合体。

### 11. 什么是 MTU？MTU 设置不当会导致哪些问题？

**要点：**
- MTU = 链路层单帧最大载荷（以太网标准 **1500 字节**），IP 包超过就**分片**
- 问题一（分片性能）：频繁分片增加 CPU 和丢包放大（一片丢全包重传）——TCP 会靠 MSS 协商避分片
- 问题二（**VPN/隧道黑洞**）：封装（PPPoE 1492、IPSec、GRE、VXLAN 1450）挤占载荷空间，本端 1500 发出去到隧道口被分片或静默丢 → **现象：ping 小包通、大包不通；SSH 能连上但 ls 一卡就死**
- 排查：`ping -M do -s 1472 target`（1472+28=1500）逐段测出路径最小 MTU（PMTUD）
- 解决：对齐各段 MTU（VXLAN 环境 1450）、或开 TCP MSS clamping

**一句话答：** MTU 是单帧最大字节数，以太网 1500。设大了遇到隧道封装就会分片甚至黑洞——典型症状是小包 ping 得通、SSH 能登录但一传数据就卡死。排查用 ping -M do -s 1472 沿路径找最小 MTU，解决是全网对齐，VXLAN 环境记得设 1450。

### 12. 如何使用 tcpdump 抓取指定主机、端口的数据包？

**要点：**
- 常用组合：
  - 指定主机：`tcpdump -i eth0 host 10.0.0.5`
  - 指定端口：`tcpdump -i eth0 port 8080`
  - 组合条件：`tcpdump -i eth0 src 10.0.0.5 and dst port 443`
  - 排除：`not host 10.0.0.1`
- 常用参数：`-i any`（所有网卡）、`-nn`（不解析域名端口名）、`-c 100`（抓满即停）、`-w file.pcap`（存文件 Wireshark 分析）、`-A`/`-X`（看内容）、`-s 0`（完整包）
- 纪律：生产先加条件再抓，不要裸抓；抓 HTTPS 只能看到加密流量，应用问题优先看日志

**一句话答：** 语法就是 BPF 表达式：host 10.0.0.5 过主机、port 8080 过端口，and/not 组合。参数记五个：-i any 全网卡、-nn 不反解析、-c 限量、-w 存 pcap 给 Wireshark、-A 看明文内容。生产纪律是先想好过滤条件再抓，否则流量大时瞬间写满磁盘。

### 13. tcpdump 抓的数据包怎么分析？

**要点：**
- **本机速览**：`tcpdump -r file.pcap -nn` 重放；`-A` 看应用层明文（HTTP）
- **Wireshark 深度分析**（主力）：
  - 过滤器：`ip.addr==10.0.0.5`、`tcp.port==8080`、`http`、`tcp.flags.syn==1`
  - **Statistics → Conversations**：会话排行（谁流量最大）
  - **Expert Information**：重传/RST/乱序告警汇总
  - **Follow TCP Stream**：还原单连接完整交互
- 常见病象：大量 retransmission（丢包/对端压力大）、RST（端口拒绝/防火墙拦截）、握手无响应（ACL 丢包）、窗口持续为 0（接收方处理不过来）

**一句话答：** 简单的 tcpdump -r 重放加 -A 看明文；深度分析进 Wireshark：Conversations 看流量排行、Expert Information 汇总重传和 RST、Follow Stream 还原单条连接。实战重点认三种病：大量重传是网络丢包，收到 RST 是对端拒绝或防火墙拦，握手发出去没回包是中间设备丢的。

### 14. Linux 网络丢包，如何排查？

**要点：**
- **分层定位**：
  - 应用/内核接收队列：`netstat -s | grep -i -E "drop|overflow|prune"`（socket buffer 溢出统计）
  - 网卡驱动：`ethtool -S eth0 | grep -i -E "drop|miss|discard"`（ring buffer 溢出）
  - 链路：`ip -s link`（RX/TX errors/dropped）
  - conntrack：`dmesg | grep conntrack`（表满丢包）
  - 中断/软中断：`mpstat -P ALL 1` 单核 %si 100%（单队列网卡瓶颈）
- **路径定位**：mtr 双向测（去程/回程丢包可能不对称）
- 解决对应：调大 ring buffer（ethtool -G）、调大 netdev_max_backlog、RPS/RSS 多队列、扩 conntrack、更换带宽

**一句话答：** 由近及远排查：netstat -s 看 socket 层溢出统计，ethtool -S 看网卡 ring buffer 丢包，ip -s link 看接口层，dmesg 查 conntrack 满；路径问题用 mtr 看具体哪跳丢、注意去回程分开测。解决方案跟根因对应：缓冲区满了调大 buffer 或开多队列，表满了扩表，物理链路问题换线。

### 15. 简述 ARP 协议的作用与工作原理？

**要点：**
- 作用：**同一局域网内**把 IP 地址解析成 **MAC 地址**（以太网帧最终靠 MAC 交付）
- 过程：主机 A 要和 B 通信 → 广播 **ARP Request**（谁是 10.0.0.5，请告诉 10.0.0.1）→ B 单播回 **ARP Reply** → 双方写入 **ARP 缓存**（`ip neigh` 查看）
- 免费 ARP（gratuitous ARP）：上线/切换 VIP 时广播宣告"这个 IP 的 MAC 变了"——Keepalived 切换靠它
- 常见问题：ARP 欺骗（伪造应答劫持流量）、代理 ARP、跨网段不能 ARP（走网关）
- 命令：`arp -n`、`ip neigh`、`arping`（检测 IP 冲突）

**一句话答：** ARP 解决"知道 IP 不知道 MAC"的问题：广播问、单播答、结果缓存。跨网段的流量不 ARP 目标而 ARP 网关。运维关联三点：Keepalived 切 VIP 靠免费 ARP 宣告新 MAC，ARP 欺骗是内网劫持的经典手法，排查 IP 冲突用 arping。

### 16. 简述交换机的工作原理？

**要点：**
- 二层设备，基于 **MAC 地址表（CAM 表）**转发：
  1. 收到帧 → 记录**源 MAC + 入端口**进表（学习）
  2. 查目的 MAC → 表中有则**单播转发**到对应端口
  3. 表中无（未知单播）→ **泛洪**到除入口外的所有端口
  4. 广播帧始终泛洪（广播域内）
- MAC 表老化机制（默认约 300s）
- 进阶：VLAN 隔离广播域、STP 防环路、端口安全绑定 MAC
- 与路由器区别：交换机二层按 MAC，路由器三层按 IP，路由器隔离广播域

**一句话答：** 交换机靠 MAC 地址表干活：见一个记一个源 MAC，转发时查表单播发，查不到就泛洪，表项定期老化。它工作在二层不认识 IP。相关的两个重要机制是 VLAN 划广播域、STP 防环——两根线错误互联会产生广播风暴，就是靠 STP 阻塞解决的。

### 17. 简述路由器的工作原理？

**要点：**
- 三层设备，基于**路由表**按目的 IP 转发：
  1. 收包解封装 → 查**目的 IP** 匹配路由表（最长前缀匹配）
  2. 转发到下一跳/出接口，**TTL 减 1**（为 0 丢弃并发 ICMP 超时——traceroute 的原理）
  3. 重写二层帧头（下一跳 MAC）
- 路由来源：直连路由、**静态路由**（手配）、**动态路由**（协议学习：OSPF/BGP）
- 特性：隔离广播域、NAT、ACL、QoS
- Linux 上路由器化：`ip_forward=1` + iptables NAT（软路由/网关原理相同）

**一句话答：** 路由器按目的 IP 查路由表、最长前缀匹配选路，转发时 TTL 减一、重写二层头。路由从三个来源来：直连、静态手配、动态协议学。运维最常打交道的 NAT 也是路由器的活——Linux 开 ip_forward 加 iptables 的 MASQUERADE 就是一台软路由。

### 18. 什么是静态路由与动态路由？

**要点：**
- **静态路由**：手工配置固定路径（`ip route add 10.20.0.0/16 via 10.0.0.1`）；可控、不占资源、**不会自适应拓扑变化**——链路断了要人工切；适合小型网络/明确出口
- **动态路由**：路由器间跑协议自动学习交换路由信息，拓扑变化自动收敛；适合大型/多路径网络
- 默认路由是特殊的静态路由（0.0.0.0/0），出口必配

**一句话答：** 静态路由手工指路，可控但链路挂了不会自己换道；动态路由协议自动学习和收敛，拓扑变化自动切换。实践中混合用：内部骨干跑动态协议，边缘和出口用静态路由加默认路由，简单直接。

### 19. 动态路由协议有哪些及作用？

**要点：**
- **IGP（内部网关协议）**——一个自治系统内：
  - **OSPF**：链路状态，SPF 算法，区域划分，企业内网事实标准，收敛快
  - RIP：距离矢量跳数计费，上限 15 跳，已淘汰（教学保留）
  - IS-IS：链路状态，运营商常用
- **EGP（外部网关协议）**——自治系统之间：**BGP**（BGP-4，路径矢量，互联网骨干唯一的域间协议；云厂商 SDN、K8s Calico BGP 模式也在用）
- 选型直觉：域内 OSPF，域间 BGP；RIP 只在面试题里活着

**一句话答：** 分两类：自治系统内部用 IGP——主流是 OSPF，链路状态算法收敛快；自治系统之间用 BGP，全网互联网骨干就是它撑的。RIP 是老古董只做面试考点。运维能遇到 BGP 的地方除了机房，还有 Calico 的 BGP 组网和云专线。

### 20. VLAN 是什么？解决了什么问题？

**要点：**
- VLAN = 二层**逻辑隔离**：同一台交换机的端口划进不同广播域，互相二层不通
- 解决的问题：广播风暴范围过大、安全隔离（不同部门/业务混在一张大二层）、灵活分组（跨交换机同 VLAN）
- 802.1Q：帧里插 4 字节 **VLAN Tag**（VID 1-4094）；交换机间链路 **Trunk** 带标签，接终端 **Access** 口去标签
- 跨 VLAN 通信需要三层设备（VLANIF/路由器单臂路由）
- Linux：`ip link add link eth0 name eth0.100 type vlan id 100`

**一句话答：** VLAN 在二层把一张物理网切成多个逻辑网，解决三个问题：广播域太大导致风暴、安全上不隔离、组织上不灵活。技术核心是 802.1Q 标签——交换机之间 Trunk 口带标签传多个 VLAN，接电脑的 Access 口剥标签。不同 VLAN 要通信必须过三层。

### 21. Linux 如何添加路由？

**要点：**
- 临时（重启失效）：
  - `ip route add 10.20.0.0/16 via 10.0.0.1 dev eth0`
  - `ip route add default via 10.0.0.1`（默认路由）
  - 主机路由：`ip route add 10.0.0.5/32 via 10.0.0.1`
- 永久：
  - RHEL 系：`/etc/sysconfig/network-scripts/route-eth0`（`10.20.0.0/16 via 10.0.0.1`）
  - Debian 系：/etc/network/interfaces 的 up 行；Netplan/NetworkManager 配置
  - 通用兜底：/etc/rc.local 或 systemd 单元
- 查看：`ip route show`；删除 `ip route del`

**一句话答：** 临时加 ip route add 目标网段 via 下一跳，默认路由就 0.0.0.0/0；要持久就写发行版的网络配置文件，RHEL 是 route-eth0，Debian 系写 interfaces。改完 ip route show 验证，多网卡机器注意 metric 决定优先级。

### 22. Linux 网络慢，如何排查？

**要点：**
- 先定性："慢"是延迟高、带宽小、丢包还是连接建立慢——工具不同
- 基础三测：ping 测 RTT 稳定性、mtr 看路径丢包、iperf3 测实际带宽（排除带宽本身）
- **协议栈层**：`ss -ti` 看连接 RTT/重传、`netstat -s` 重传率和缓冲溢出、`sar -n DEV 1` 看网卡是否跑满
- **主机层**：CPU 软中断单核打满（`mpstat -P ALL`）、conntrack 满、DNS 慢（`time dig 域名`）
- 对比法：同机房两台互测、换已知正常的机器测同目标——隔离是我的问题还是链路问题

**一句话答：** 先把"慢"翻译成指标：ping 看延迟抖动，mtr 看丢包在哪跳，iperf3 测裸带宽——这三个做完通常已经把问题域缩小一半。再上主机层：sar 看网卡是不是跑满、netstat -s 看重传、mpstat 看软中断单核瓶颈、time dig 排 DNS。最后用对比测试隔离是本机、链路还是对端的问题。

### 23. 有哪些常用的网络故障排查工具及作用？

**要点：**
- 连通性：**ping**（ICMP 连通/RTT）、**telnet/nc**（TCP 端口通断，服务级验证）、**mtr**（路径+丢包统计）
- 路径：**traceroute/tracepath**（逐跳路由）
- 配置查看：**ip addr/route/neigh**（地址/路由/ARP）、`ethtool`（网卡状态/协商速率）
- 流量：**ss**（连接状态）、**iftop/nload**（实时带宽）、**tcpdump**（抓包）、**iperf3**（带宽测试）
- DNS：**dig/nslookup/host**
- 组合套路：ping → 端口 telnet → mtr → tcpdump，逐层深入

**一句话答：** 我的工具箱分层记忆：测连通 ping 和 nc，看路径 mtr 和 traceroute，看配置 ip 和 ethtool，看流量 ss 和 iftop，抓包 tcpdump，测带宽 iperf3，查 DNS 用 dig。面试时按"从 ping 到 tcpdump 的排查顺序"讲，比罗列命令名更有说服力。

### 24. Linux 服务器网络不通，如何排查？

**要点：**
- **由近及远**：
  1. 网卡状态：`ip link`（UP？`ethtool eth0` 协商速率？灯亮不亮——物理层）
  2. IP 配置：`ip addr`（地址对不对、掩码错没错）
  3. 网关：`ping 网关IP` → 通则本机二层没问题，继续往上
  4. 公网 IP：`ping 223.5.5.5`（绕开 DNS）
  5. DNS：`ping baidu.com` 失败但 IP 通 → resolv.conf 问题
  6. 端口：`nc -zv 目标 端口`（服务级）
  7. 防火墙：`iptables -L -n` / `firewall-cmd --list-all`、SELinux
  8. 路由：`ip route`（默认路由在不在、策略路由）
- 仍不通：tcpdump 双端抓包——本机发出去了没有、对端收到没有，定位丢在哪
- 云环境补充：安全组、VPC ACL、弹性网卡绑定

**一句话答：** 固定套路从下往上：ip link 看网卡、ip addr 看地址、ping 网关验二层、ping 223.5.5.5 验公网、dig 验 DNS、nc 验端口、查防火墙和路由。还没结果就两端同时 tcpdump，看包死在哪一段。云上多想一层：安全组和 ACL 是最常见的"隐形防火墙"。

### 25. Linux 上 traceroute 路由追踪实现原理是什么？

**要点：**
- 利用 **TTL**：发 TTL=1 的包 → 第一跳路由器丢弃并回 **ICMP Time Exceeded**（暴露自己）→ TTL=2 → 第二跳暴露……逐跳累计绘制路径
- 三种探测包形态：
  - UDP（Linux 默认）：发高位端口 UDP，到终点时目标回 **ICMP Port Unreachable** 判定到达
  - ICMP（Windows tracert 默认）：Echo Request，终点回 Echo Reply
  - **TCP（tcptraceroute/traceroute -T）**：探测指定 TCP 端口——**能穿透禁 ICMP/UDP 的防火墙**，专治"中间全是 *"
- 星号（*）≠ 必丢包：该跳可能只是不回 ICMP 错误消息
- mtr 原理相同但每跳持续发包统计丢包率/延迟分布

**一句话答：** 原理是玩 TTL：发 TTL=1 的包第一跳路由器必回 ICMP 超时暴露自己，TTL 递增逐跳画出路径，到达终点靠端口不可达或应答判定。默认用 UDP，被防火墙挡住满屏星号时换 TCP 模式探测业务端口往往就通了——因为很多设备只放行 TCP。

### 26. 为什么 TIME_WAIT 需要等待 2MSL？

**要点：**
- MSL = 报文最大生存时间（Linux 视角单方向 30s，2MSL=60s 即 TIME_WAIT 时长）
- **原因一：可靠地完成终止**——主动方发的最后一个 ACK 可能丢失；若立即关闭，被动方重发的 FIN 将无人应答（对方只能 RST 或收不到，异常关闭）。等 2MSL 内若收到重传 FIN 还能重发 ACK
- **原因二：让旧报文自然消亡**——2MSL 保证本连接的所有报文都从网络消失，防止"幽灵报文"被同四元组的新连接误收造成数据错乱
- 一句话：一个 MSL 保证"我发出去的都死了"，再一个 MSL 保证"对方发来的也死了"
- 关联：这也是 TIME_WAIT 出现在**主动关闭方**的原因

**一句话答：** 两个目的。一是保最后那个 ACK 万无一失：它丢了对方会重发 FIN，我必须在 TIME_WAIT 里能重答；二是清场：等满 2MSL 让这条连接的旧报文在网络里彻底死光，免得四元组复用后被新连接误收。代价就是主动关闭方要压 60 秒端口——这就是高并发短连接端口耗尽的根源。

## 运维流程规范面试题

### 1. 变更管理流程规范

**要点：**
- 核心环节：**申请 → 评审 → 审批 → 实施 → 验证 → 关闭**；分变更级别（标准变更/常规/重大）
- 重大变更要素：实施方案+**回滚方案**双必备、影响面分析、时间窗口（避开业务高峰）、双人复核、灰度顺序
- 变更窗口管理：默认维护窗口、紧急变更走特批但事后补录
- 度量：变更成功率、变更引发故障占比——反过来优化流程颗粒度
- 一切变更进 CMDB/工单系统留痕，口头变更是事故之源

**一句话答：** 我的变更纪律是"三有一无"：有方案、有回滚、有审批、无口头变更。实施按灰度顺序推进、双人复核关键步骤、完成必须验证业务指标而不仅是服务活着。变更成功率是流程健康度的核心指标。

### 2. 应用发布管理流程

**要点：**
- 标准链路：代码合入（CR）→ CI 构建 → 制品入库（版本化）→ 发布审批 → **灰度发布**（金丝雀/按批次）→ 全量 → 观察确认
- 关键机制：发布可回滚（制品保留、数据库变更向后兼容）、发布检查单（配置、依赖、迁移脚本）、发布窗口
- 回滚演练：回滚脚本要真实演练过，纸面回滚等于没有
- 度量：发布频率、变更失败率、MTTR（DORA 指标）

**一句话答：** 好的发布流程三个关键词：制品化——构建产物版本化入库而不是现场打包；灰度——先金丝雀再全量，出问题炸的只有一小片；可回滚——回滚脚本演练过才算数。配套看 DORA 四指标：发布频率、变更失败率、MTTR、前置时间。

### 3. 故障处理流程

**要点：**
- 优先级铁律：**先止血恢复业务，再定位根因**（分离处理）
- 标准动作：发现（监控/报障）→ 定级定责（P0-P3，指定故障指挥）→ 止血（回滚/切流/扩容/降级）→ 恢复确认 → 根因排查 → **复盘**（时间线、5Why、action list）
- 通报机制：定时同步进展（15-30 分钟一次），不让业务方反复追问
- 复盘文化：无责复盘，改进项有 owner 和 deadline，跟踪闭环

**一句话答：** 核心原则是止血和根因分离：P0 故障第一目标是恢复业务——回滚、切流、降级都是手段，不要在现场恋战找根因。过程中指定故障指挥统一协调、定时对外通报；恢复后 48 小时内无责复盘，改进项落到人和时间。

### 4. 监控告警流程

**要点：**
- 告警分级：P0 电话叫醒 / P1 IM 强提醒 / P2 工作时间处理 / P3 日报——**分级决定响应方式**
- 告警质量治理：每个告警必须有明确 action（收到后做什么），无效告警定期清理；抑制和聚合防风暴
- 监控分层：资源层（CPU/内存/磁盘）→ 中间件层 → 应用层（QPS/错误率/延迟，USE+RED 方法）→ 业务层
- 值班机制：轮换、升级路径（15 分钟无响应升级）、值班交接文档

**一句话答：** 我的告警三原则：分级响应——P0 电话 P1 IM，收到的告警必须有对应 action——响不了的告警就是噪音，定期治理。监控覆盖按 USE 和 RED 方法分层建设，值班有明确升级路径，15 分钟没人认领自动升级到下一级。

### 5. 备份恢复流程

**要点：**
- 策略：全量+增量组合、**3-2-1 原则**（3 份副本、2 种介质、1 份异地）、RPO/RTO 定目标（丢多少数据、多久恢复）
- 备份内容：数据+配置+代码+**证书密钥**（易漏项）
- **恢复演练是灵魂**：季度级恢复演练，没验证过恢复的备份不可信
- 流程：备份任务监控告警（备份失败比备份成功更需要知道）→ 备份介质轮换与保留策略（如 7 天日备+月备）→ 恢复预案文档化

**一句话答：** 备份流程三件事：定 RPO/RTO 目标选备份策略，3-2-1 落地异地多介质，然后——最重要的——定期恢复演练。我判断一个团队备份做没做到位，不问"有没有备份"，问"最近一次恢复演练是什么时候"。

### 6. 权限申请流程

**要点：**
- 原则：**最小权限**、按需申请、限期/定期回收
- 流程：申请人填单（权限范围、用途、期限）→ 资源 owner 审批 → 堡垒机/平台开通 → 定期审计（季度权限 review）
- 落地工具：堡垒机统一入口（不直接给服务器密码）、RBAC 按角色授权、数据库走 Proxy 审计
- 高危权限（root、数据库 DBA）双人授权+操作留痕；离职/转岗即时回收

**一句话答：** 权限管理的核心是最小化加可回收：按需申请写明用途和期限，owner 审批后经堡垒机开通，季度做一次权限 review 清理僵尸权限。技术落地是 RBAC 加堡垒机审计——目标是任何操作都有主、任何人都有边界。

### 7. 事件管理流程

**要点：**
- 事件（Event）：一切影响或可能影响服务的 IT 事件——含未成故障的异常
- 流程：记录（统一入口/工单）→ 分类分级 → 诊断处理 → 恢复 → 关闭 → 记录沉淀
- 与问题管理的关系：事件管"快速恢复"，问题管理管"根因消除"——重复发生的事件要升级为问题单
- 目标指标：事件响应时长、解决时长、重复事件率

**一句话答：** 事件管理的目标是快速恢复而不是追根因——那是问题管理的活。流程上保证两点：所有事件进统一工单留痕可统计，重复出现的事件必须升级成问题单做根因消除，否则永远在救同一个火。

### 8. 应急响应流程

**要点：**
- 应急预案先行：针对已知风险（主库宕机、DDoS、数据泄漏、证书过期）预写** playbook**——职责到人、步骤到命令、联系方式到手机号
- 组织：故障指挥（协调+决策+通报）、处置组、通信组——**单点指挥**避免多头乱战
- 响应节奏：发现即通报 → 定级 → 按预案止血 → 持续通报（15min 节奏）→ 恢复 → 复盘
- 演练：预案不演练就是废纸——季度级故障演练/混沌工程

**一句话答：** 应急的精髓在"事前"：高风险场景提前写好 playbook，谁指挥、谁动手、怎么止血、找谁审批，全都预置好。事中就是执行：单点故障指挥，15 分钟一次进展通报。事后必须问一句：预案为什么没覆盖到？然后把它补上。

### 9. 巡检管理流程

**要点：**
- 理念演进：**人工巡检 → 自动化巡检 → 无人值守**（能自动发现和自愈的就不需要人看）
- 巡检内容清单化：资源水位（磁盘>80%）、错误日志增量、备份任务成功率、证书有效期、慢查询趋势、队列堆积
- 频率分级：每日核心项、每周全量、每月深度（含恢复演练）
- 输出：巡检报告有结论和跟进项——发现问题必须闭环到人

**一句话答：** 巡检的目标是让自己失业：清单化所有要看的点，先做自动化巡检脚本定时跑并推送结果，能自愈的加自动处置。人只处理巡检发现的新问题，并且每一个发现都要闭环到人——巡检报告没人跟进等于白巡。

### 10. 工单管理流程

**要点：**
- 统一入口：所有请求（权限、变更、故障、咨询）走工单，杜绝 IM 直接派活无痕操作
- 生命周期：提交 → 分派（自动路由到负责组）→ 处理 → 验证 → 关闭 → 满意度
- SLA：按优先级定响应/解决时限（P0 15 分钟响应、P2 1 个工作日），超时自动升级
- 价值：工作量可视化、知识沉淀（常见问题转 FAQ/SOP）、责任可追溯

**一句话答：** 工单的核心价值是留痕和度量：所有请求统一入口，按 SLA 定响应时限，超时自动升级。团队收益是两个——工作量有了数据，重复出现的问题转成 SOP。运维最忌"IM 里说一声就干"，出了事连是谁让干的都查不到。

### 11. 日志管理流程

**要点：**
- 全生命周期：产生（格式规范：JSON/统一字段含 traceID）→ 采集（Filebeat/Fluent Bit）→ 传输 → 存储（ES/对象存储，热温冷分层）→ 分析告警 → 归档销毁（保留期合规：等保/行业要求）
- 规范：日志级别合理使用（ERROR 不是垃圾桶）、敏感信息脱敏（密码/身份证禁止落日志）、时间戳统一 UTC 或带时区
- 安全：日志集中后限制删除权限（防攻击者清痕迹）、审计日志独立存储

**一句话答：** 日志管理管全生命周期，我重点抓三条：格式标准化——统一 JSON 带请求 ID 才能全链路追踪；脱敏——密码和敏感数据绝不能进日志；保留与安全——按合规定保留期，审计日志单独存储限制删除权限，防被清痕迹。

### 12. 日常操作规范流程

**要点：**
- 高危操作清单化：rm -rf、drop/truncate、iptables 清空、kill -9 主库、防火墙变更——**双人口令复核**或二次确认
- 操作纪律：生产操作走堡垒机、操作前看当前状态留基线、变更前快照/备份、操作后验证
- 禁止项：生产环境直接测试、明文传输密码、共享账号、无人值守的长时间命令（用 tmux）
- 文档习惯：重要操作写操作记录（时间/人/内容/结果）

**一句话答：** 日常操作规范浓缩成三条铁律：高危命令必须复核——rm -rf、drop 这类不可逆操作双人确认；操作前后留证据——先记录状态、做备份，操作完验证；一切生产操作走堡垒机留审计，共享账号是事故追责的最大障碍。

### 13. 安全运维流程

**要点：**
- 日常安全动作：补丁管理（定期评估+窗口升级）、漏洞扫描与修复闭环（高危限时）、基线加固（CIS Benchmark）
- 账号安全：密钥优先、sudo 最小化、堡垒机审计、离即回收
- 纵深防御：WAF/防火墙最小暴露面、内网分段、敏感数据加密
- 应急：安全事件响应流程（隔离取证优先于直接删除）、日志留存满足合规
- 定期动作：渗透测试、安全巡检、安全意识培训

**一句话答：** 我把安全运维归为四条线：暴露面收敛——端口和权限能少则少；漏洞闭环——扫描到修复到验证有 SLA；账号管控——密钥加堡垒机加审计；应急演练——出了安全事件第一动作是隔离取证不是手忙脚乱删文件。

### 14. 资产管理流程

**要点：**
- **CMDB 是核心**：记录资产（物理机/虚机/云资源/域名/证书/IP 段）及其关系（依赖拓扑）
- 生命周期：采购入库 → 上线登记 → 变更跟随 → 维保到期预警 → 下线销毁
- 数据质量：自动发现+人工维护结合（Agent/云 API 同步），定期盘点核对——**过期的 CMDB 比没有更危险**
- 关联场景：故障时按 CMDB 反查影响面、下线服务器先查依赖、证书到期统一告警

**一句话答：** 资产管理的核心是 CMDB 的数据质量：尽量自动发现和同步——云上走 API、主机走 Agent，人工只维护增量；定期盘点核对。资产数据的最大价值在关系：故障时能反查影响面，下线时能查依赖，证书域名能统一到期告警。

### 15. 配置管理流程（CMDB）

**要点：**
- 配置项（CI）粒度：主机、服务实例、中间件集群、应用、人（owner）及其间关系
- 建设要点：
  - **单一数据源**：所有自动化系统（监控、发布、权限）从 CMDB 取数，不允许各存一份
  - 自动发现优先，人工补充关系与业务属性
  - 变更联动：变更单完成后自动更新 CMDB（否则必腐烂）
  - 模型先行：先定义属性和关系模型再录入数据
- 消费场景：监控自动部署（新机器入 CMDB 自动加监控）、发布定位目标机器、故障影响面分析、成本分摊

**一句话答：** 配置管理的成败在"活不活"：CMDB 必须是唯一数据源，监控、发布、权限都从它取数；新机器入库自动挂监控、变更单完成自动更新——靠人手工维护的 CMDB 三个月就变成谎言。先建模再填数据，关系和 owner 比属性字段更有价值。
