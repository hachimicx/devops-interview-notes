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

## 二、四次挥手详解

### 完整报文序列（本机实测，`tcpdump -i lo` 抓包）

```
 1.  00:00:00.000000  client.49500 > server.34567  Flags [S]   seq 2652636233
 2.  00:00:00.000014  server.34567 > client.49500  Flags [S.]  seq 3214520920  ack 2652636234
 3.  00:00:00.000012  client.49500 > server.34567  Flags [.]   ack 3214520921
 4.  00:00:00.000161  server.34567 > client.49500  Flags [P.]  seq 3214520921:3214520926  ← 服务端发数据
 5.  00:00:00.000006  client.49500 > server.34567  Flags [.]   ack 3214520926
 ---------- 以上为建连与数据传输，以下为四次挥手 ----------
 6.  00:00:00.304184  client.49500 > server.34567  Flags [F.]  seq 2652636234  ack 3214520926  ← ① 客户端 FIN
 7.  00:00:00.000298  server.34567 > client.49500  Flags [.]   ack 2652636235                 ← ② 服务端 ACK
 8.  00:00:06.695665  server.34567 > client.49500  Flags [F.]  seq 3214520926  ack 2652636235  ← ③ 服务端 FIN
 9.  00:00:00.000021  client.49500 > server.34567  Flags [.]   ack 3214520927                 ← ④ 客户端 ACK
```

注意 ② 与 ③ 之间相隔 **6.7 秒**——这 6.7 秒就是服务端停留在 `CLOSE_WAIT` 的时长（服务端应用此时尚未调用 `close()`）。

### 四个报文对应的状态迁移

| 步骤 | 报文 | 主动关闭方状态 | 被动关闭方状态 |
|---|---|---|---|
| ① | 主动方发 `FIN` | `FIN_WAIT_1` | `ESTABLISHED` |
| ② | 被动方回 `ACK` | `FIN_WAIT_2` | `CLOSE_WAIT` |
| ③ | 被动方发 `FIN` | `FIN_WAIT_2` | `LAST_ACK` |
| ④ | 主动方回 `ACK` | `TIME_WAIT`（持续 2MSL） | `CLOSED` |

**本机 `ss -tan` 实测状态**（同一连接的两次采样）：

```
客户端发 FIN 后：
   FIN-WAIT-2   local=127.0.0.1.49500  peer=127.0.0.1.34567   ← 主动关闭方
   CLOSE-WAIT   local=127.0.0.1.34567  peer=127.0.0.1.49500   ← 被动关闭方

服务端 close 之后：
   TIME-WAIT    local=127.0.0.1.49500  peer=127.0.0.1.34567   ← 主动关闭方
   （服务端连接已 CLOSED，不再出现）
```

### 为什么是四个而不是三个

TCP 是全双工的，`FIN` 只表示**本方向**不再发送数据，另一方向仍可继续传输。因此被动关闭方收到 `FIN` 后：

1. 先回 `ACK` 确认（此时本端可能仍有数据要发，不能立即关闭）
2. 待自身数据发送完毕，再单独发一个 `FIN`

两个动作无法合并，故为四个报文。

### 合并为三个的情况

若被动关闭方**无待发数据**，其 `ACK` 可与自身 `FIN` 在同一报文中发出，表现为三个报文。前文 sshd 那次抓包即为此情形（服务端发完 banner 后无剩余数据）。

### 半关闭（half-close）

主动关闭方处于 `FIN_WAIT_2` 时，**仍可接收数据**，仅不再发送——该状态即半关闭。

- 应用层由 `shutdown(SHUT_WR)` 触发，与 `close()` 不同：`close()` 同时关闭读写两个方向
- 本机实测：客户端 `shutdown(SHUT_WR)` 后，服务端仍持有连接处于 `CLOSE_WAIT` 达 8 秒，期间客户端一直停留在 `FIN_WAIT_2`，连接未断开

### 各状态的排障意义

| 状态 | 含义 | 堆积时说明什么 |
|---|---|---|
| `FIN_WAIT_1` | 已发 FIN，等对端 ACK | 通常瞬时，堆积说明对端无响应 |
| `FIN_WAIT_2` | 已收 ACK，等对端 FIN | 由 `tcp_fin_timeout` 控制（本机 60s），超时内核强制关闭 |
| `CLOSE_WAIT` | **对端已关闭，本端应用尚未 `close()`** | **应用代码缺陷**（未正确释放连接），不是内核参数问题 |
| `LAST_ACK` | 被动方已发 FIN，等最后 ACK | 通常瞬时 |
| `TIME_WAIT` | 主动方等待 2MSL | 主动关闭方过多，考虑连接池 / keepalive |

**排障要点：`CLOSE_WAIT` 堆积查应用代码，`TIME_WAIT` 堆积查连接模式（短连接 vs 长连接）。二者成因完全不同，不能混为一谈。**

### 相关内核参数

```bash
net.ipv4.tcp_fin_timeout     # FIN_WAIT_2 超时，本机 60
net.ipv4.tcp_max_orphans     # 孤儿连接上限
net.ipv4.tcp_max_tw_buckets  # TIME_WAIT 上限，本机 8192
```


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
