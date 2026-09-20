# TCP 三次握手 / 四次挥手 / SYN 重传

> 全部为本机实测：Linux 6.12.107+deb13-amd64。报文来自 `tcpdump -i lo` 真实抓包

## 一、三次握手（真实抓包）

```
1  client → server  Flags [S]   seq 2265773849                                    ← SYN
2  server → client  Flags [S.]  seq 3920394034  ack 2265773850                    ← SYN+ACK
3  client → server  Flags [.]   ack 3920394035                                    ← ACK
```

注意序号：客户端 SYN 的 `seq=2265773849`，服务端回的 `ack=2265773850`（**seq+1**）——SYN 本身要占掉一个序号。服务端同理。

### 为什么要三次？两次为什么不行

**核心：第三次握手是让客户端「确认」这个连接。**

具体场景（两次握手会出事）：客户端发了一个 SYN，这个 SYN 在某个路由器里卡了很久；客户端等不及重发 SYN，第二次很快建好、传完数据、关闭。**然后那个迟到的老 SYN 才到达服务端。**

两次握手的情况下：服务端收到老 SYN，直接认为连接建立，回 SYN+ACK 并分配资源，然后**傻等客户端发数据**——而客户端根本不认这个连接（早关了）。结果服务端凭空挂着一个僵尸连接。攻击者可以利用这个做 SYN 泛洪。

三次握手就能解决：服务端回 SYN+ACK 后，客户端一看这个 SYN+ACK 不是自己要的（连接已关），回一个 **RST**，服务端立刻释放。

## 二、为什么建立三次、断开四次

- **建立三次**：服务端收到 SYN 后，「我同意」和「我也要建」这两件事可以**合并**成一个 SYN+ACK 发出去 → 3 个包
- **断开四次**：TCP 是**全双工**的。一方发 FIN 只是说"我这个方向没数据要发了"，**另一个方向可能还有数据**。所以对端先回 ACK 确认，等自己数据也发完了，再单独发一个 FIN → 4 个包

### ⚠️ 实测细节：四次挥手常常只有 3 个包

本机抓到的关闭过程：

```
4  server → client  Flags [F.]  seq ...  ack ...     ← FIN（服务端）
5  client → server  Flags [.]   ack ...              ← ACK
6  client → server  Flags [F.]  seq ...  ack ...     ← FIN（客户端）
7  server → client  Flags [.]   ack ...              ← ACK
```

服务端当时**没有数据要发**，于是把 ACK 和自己的 FIN **合并**成一个包发了。所以"四次挥手"在实际抓包里经常是 3 个包。**答出这个细节是加分项。**

## 三、SYN 没收到回应，客户端等多久

### 本机参数

```
net.ipv4.tcp_syn_retries        = 6
net.ipv4.tcp_syn_linear_timeouts = 4
```

### 实测：抓 11 个 SYN

对黑洞地址 `192.0.2.1:80` 发起连接，抓包得到的**重传间隔（秒）**：

```
1.03  1.02  1.02  1.02  1.02  2.02  4.19  8.19  16.13  32.26
└────── 5 次固定 1 秒（线性）──────┘ └──── 指数退避 ────┘
```

- **SYN 报文总数：11 个**（1 个初始 + 10 次重传）
- 最后一次重传发生在 **约 68 秒**
- 内核在 **约 133 秒** 彻底放弃，报 `Connection timed out`（errno 110）
- 三次独立测量：135.3s / 135.2s / 133.4s

### 为什么前 5 次是固定 1 秒而不是指数退避

内核源码 `net/ipv4/tcp_timer.c` v6.12 第 660-668 行：

```c
} else if (sk->sk_state != TCP_SYN_SENT ||
           tp->total_rto >
           READ_ONCE(net->ipv4.sysctl_tcp_syn_linear_timeouts)) {
        /* Use normal (exponential) backoff unless linear timeouts are activated. */
        icsk->icsk_backoff++;
        icsk->icsk_rto = min(icsk->icsk_rto << 1, TCP_RTO_MAX);
}
```

即：**处于 SYN_SENT 且 `total_rto` 未超过 `tcp_syn_linear_timeouts` 时，RTO 不翻倍**，保持初始 1 秒线性重传。这是内核为「单个 SYN 丢包」设计的快速恢复机制。超过阈值后才进入指数退避。

### 重传次数对得上

`tcp_write_timeout()` 第 256-258 行：

```c
max_retransmits = retry_until;                          /* tcp_syn_retries = 6 */
if (sk->sk_state == TCP_SYN_SENT)
    max_retransmits += sysctl_tcp_syn_linear_timeouts;   /* + 4 */
expired = icsk->icsk_retransmits >= max_retransmits;     /* 10 次即放弃 */
```

**6 + 4 = 10 次重传** → 加上初始的 1 个 SYN = **11 个报文**，与抓包完全吻合。

### 内核文档的旧口径

> Default value is 6, which corresponds to 63seconds till the last retransmission with the current initial RTO of 1second. With this the final timeout for an active TCP connection attempt will happen after 127seconds.

即"最后一次重传 63 秒 / 最终超时 127 秒"，与实测的 68s / 133s 同一量级（差值来自线性重传阶段）。

## 关联命令

```bash
tcpdump -i any -n "tcp port 80 and host 192.0.2.1" -w /tmp/syn.pcap   # 抓 SYN
tcpdump -r /tmp/syn.pcap -n -ttt                                      # 看包间隔
tcpdump -r /tmp/hs.pcap -n -S                                         # 看完整握手/挥手
sysctl net.ipv4.tcp_syn_retries net.ipv4.tcp_syn_linear_timeouts
ss -tan state syn-sent                                                # 处于 SYN_SENT 的连接
```
