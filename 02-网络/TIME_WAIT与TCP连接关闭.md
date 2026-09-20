# TIME_WAIT 与 TCP 连接关闭

> 本文数值均为本机实测：Linux 6.12.107+deb13-amd64（Debian 13）

## 一句定义

TIME_WAIT 是**主动关闭连接的一方**（先发 FIN 的那端）在四次挥手结束后进入的状态，Linux 上持续 **60 秒**，用于保证最后一个 ACK 能重发、并让旧连接的迷途报文自然消亡。

## 机制展开

### 谁产生 TIME_WAIT

主动关闭方。本机实测：

```
本端 connect 127.0.0.1:22 → 本端 close()
本端端口 47046 → TIME-WAIT
对端 sshd       → 不进 TIME_WAIT
```

**为什么必须是主动关闭方**，两个原因：

1. **怕最后一个 ACK 丢失**。主动关闭方发完 FIN、收到对端 FIN、回了 ACK 就想释放连接。若该 ACK 丢失，对端会重传 FIN；此时若本端已彻底删除连接，只能用 RST 回应，对端会认为是"异常断开"而非"正常关闭"。留在 TIME_WAIT 期间才能重发 ACK。
2. **怕旧连接的迷途报文串到新连接**。同一四元组（源IP:源端口 + 目的IP:目的端口）若立刻建立新连接，网络中延迟的上个连接报文会被新连接接收导致数据错乱。等 2MSL 后老报文必然消亡。

### 为什么是 2MSL

MSL = 报文在网络中的最大存活时间。等 2MSL 覆盖两段：

- 本端发出的最后一个 ACK 最多存活 MSL
- 对端重传的 FIN 最多存活 MSL

**本机实测：TIME_WAIT 存活 60~65 秒**（每 5s 采样一次，60s 时仍在，65s 时已消失）。即 Linux 的 TIME_WAIT 固定为 60 秒。

### ⚠️ 高频陷阱：与 tcp_fin_timeout 无关

`net.ipv4.tcp_fin_timeout`（本机 = 60）管的是 **FIN_WAIT_2**，即孤儿连接在 FIN_WAIT_2 状态等对端 FIN 的超时时间。内核文档原文：

> The length of time an orphaned (no longer referenced by any application) connection will remain in the FIN_WAIT_2 state before it is aborted at the local end.

两个值碰巧都是 60，极易答错。且 **TIME_WAIT 时长内核未提供 sysctl 直接调整**，只能通过 `tcp_tw_reuse` 变相绕过。

### 关闭时带未读数据 → 发 RST，不进 TIME_WAIT

实测中第一次未出现 TIME_WAIT：连接后不读数据直接 `close()`，接收缓冲区尚有对端数据 → 内核发 **RST** 而非 FIN，跳过 TIME_WAIT。

用内核计数器验证（`nstat -az TcpOutRsts`）：

```
不读数据 close → TcpOutRsts 1013571 → 1013572   (+1，发 RST)
读走数据 close → TcpOutRsts 1013572 → 1013572   (不变，走正常 FIN)
```

**结论：连接上有未读数据时关闭会发 RST，不走四次挥手。**

## 面试常考 / 注意点

### 几万个 TIME_WAIT 正不正常

**稳态 TIME_WAIT 数 ≈ QPS × 60 秒。**

- QPS 500 的短连接服务 → 稳态约 3 万个，**正常**
- 判断标准不是数量，而是看两条硬约束：

| 约束 | 本机实测值 | 打满后的现象 |
|---|---|---|
| 本地端口数 | `ip_local_port_range = 32768-60999` → 28232 个 | 新连接报 `cannot assign requested address` |
| `tcp_max_tw_buckets` | 8192 | 内核**直接销毁** TIME_WAIT 并打警告（数字卡在此值不动即是它） |

- **反向信号**：服务端（被动关闭方）本不该有大量 TIME_WAIT。若服务端一堆 TIME_WAIT，说明服务端在主动关连接 —— 通常是 keepalive 未配好，或服务端超时比客户端短。

### 处理手段

- **改长连接**（连接池、keep-alive）—— 根治，其余都是缓解
- **`net.ipv4.tcp_tw_reuse`** —— 本机 = 2。内核文档语义：
  ```
  0 - disable
  1 - global enable
  2 - enable for loopback traffic only   ← 默认值
  ```
  只对**主动发起连接的一方**生效，前提是 `tcp_timestamps` 开启（本机 = 1）。对"服务端被动关闭"类问题无效。
- **扩大 `ip_local_port_range`**
- **调大 `tcp_max_tw_buckets`** —— 是保护阈值不是优化项，不可当常规手段
- ❌ **不要提 `tcp_tw_recycle`** —— 本机 kernel 6.12 的 `/proc/sys/net/ipv4/` 中已无此文件，内核 ip-sysctl 文档中亦无此条目，早已移除。历史上它在 NAT 环境下按时间戳丢包，会打死客户端。

## 关联命令

```bash
ss -s                                  # 总览，含 timewait 计数
ss -tan state time-wait                # 列出 TIME_WAIT 连接
ss -tan state time-wait | wc -l        # 计数
sysctl net.ipv4.tcp_tw_reuse net.ipv4.tcp_fin_timeout \
       net.ipv4.tcp_max_tw_buckets net.ipv4.ip_local_port_range
nstat -az TcpOutRsts                   # 发出去的 RST 数（验证 RST vs FIN）
cat /proc/sys/net/ipv4/ip_local_port_range
```
