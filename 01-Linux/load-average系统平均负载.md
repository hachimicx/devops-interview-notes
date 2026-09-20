# load average（系统平均负载）

> 本机实测：Linux 6.12.107+deb13-amd64，2 核（nproc=2）

## 一句定义

load average 是 **运行队列长度的指数衰减平均值**，统计的是 **R（运行/可运行）+ D（不可中断睡眠）** 两种状态的进程数，每 5 秒采样一次，输出 1/5/15 分钟三个值。**没有绝对高低，只有相对核数。**

## 机制展开

### 到底数什么 —— 内核源码原文

`kernel/sched/loadavg.c`（v6.12）第 16-25 行：

```c
 * The global load average is an exponentially decaying average of nr_running +
 * nr_uninterruptible.
 *
 * Once every LOAD_FREQ:
 *   nr_active = 0;
 *   for_each_possible_cpu(cpu)
 *	nr_active += cpu_of(cpu)->nr_running + cpu_of(cpu)->nr_uninterruptible;
 *   avenrun[n] = avenrun[0] * exp_n + nr_active * (1 - exp_n)
```

第 82-83 行：

```c
nr_active = this_rq->nr_running - adjust;
nr_active += (int)this_rq->nr_uninterruptible;
```

即：**load = R + D**。

### Linux 比传统 Unix 多算的那一种是 D

- 传统 Unix（含 BSD 早期）只统计 **R**（可运行 + 正在运行）
- **Linux 额外把 D（TASK_UNINTERRUPTIBLE，不可中断睡眠）算进去**
- **为什么**：D 状态进程虽然不占 CPU，但它被卡住了（绝大多数是等磁盘/网络 I/O）。若只算 R，一台 I/O 打满的机器 load 会很低、看起来"很闲"，实际已在拖。把 D 计入，load 才能反映 I/O 压力
- **副作用**：D 状态**不可中断**，`kill -9` 也杀不掉，只能等 I/O 返回或重启

### 指数衰减怎么算

`include/linux/sched/loadavg.h`：

```c
#define LOAD_FREQ  (5*HZ+1)   /* 每 5 秒采样一次 */
#define EXP_1       1884      /* 1/exp(5sec/1min)  as fixed-point */
#define EXP_5       2014      /* 1/exp(5sec/5min) */
#define EXP_15      2037      /* 1/exp(5sec/15min) */
```

公式：`newload = load * exp + active * (FIXED_1 - exp)`

每 5 秒算一次。EXP 值越小衰减越快，所以 1 分钟那个反应最灵敏，15 分钟那个最迟钝。

### 实测：2 核跑满负荷，load 怎么爬

本机 2 核，起 2 个满负荷进程，每 15 秒采样：

```
t=  0s  loadavg(1/5/15) = 0.09 0.05 0.01
t= 15s  loadavg(1/5/15) = 0.43 0.13 0.04
t= 30s  loadavg(1/5/15) = 0.78 0.22 0.07
t= 45s  loadavg(1/5/15) = 0.97 0.30 0.10
t= 60s  loadavg(1/5/15) = 1.20 0.38 0.13
t= 75s  loadavg(1/5/15) = 1.30 0.44 0.15
t= 90s  loadavg(1/5/15) = 1.38 0.50 0.18
```

1 分钟值向 **2.0（= 核数）** 爬升；5/15 分钟值因衰减慢还很低 —— 这就是"指数衰减平均"的直观体现。

停止负载后回落：`0.73 0.45 0.17` → 30 秒后 `0.44 0.40 0.17`。

## 面试常考 / 注意点

### "load 8.5 是不是快扛不住了" —— 必须先问核数

**满负载 = 核数，没有固定数值。**

- 单核：load 1.0 = 满
- 本机 2 核：load 2.0 = 满
- 16 核：load 8.5 只用了约一半，很健康

判断经验（业界常用口径，非内核定义）：

| load / 核数 | 含义 |
|---|---|
| < 0.7 | 健康 |
| ≈ 1.0 | 满载，开始排队 |
| > 1.0 | 有进程在排队等 CPU |

### 三步判断法（面试按这个答）

1. **除以核数** —— `nproc` 拿到核数再比
2. **区分 R 还是 D** —— 看 `/proc/stat` 的 `procs_running` / `procs_blocked`。
   **D 高 = I/O 瓶颈，加 CPU 没用**；R 高才是 CPU 不够
3. **看趋势** —— 1/5/15 三个值的关系：
   - `1 > 5 > 15`：负载正在上升
   - `1 < 5 < 15`：正在回落
   - 三个都高：持续压力

### /proc/loadavg 的第 4、5 字段（多数人不知道）

本机实测：

```
0.12 0.06 0.01 1/151 251366
└──1min──┘└5min┘└15min┘ └┬┘  └──┬──┘
              可运行/总进程数   最近创建的 PID
```

第 4 字段 `1/151` = 当前可运行进程数 / 系统总进程数，可直接用来判断 R 队列长度。

## 关联命令

```bash
cat /proc/loadavg                        # 原始三值 + 可运行/总数 + 最近 PID
uptime                                   # 人读版
nproc                                    # 核数（判断满负载的基准）
grep -E "procs_running|procs_blocked" /proc/stat   # R 与 D 队列长度
ps -eo pid,stat,comm | awk '$2 ~ /^D/'   # 列出 D 状态进程
vmstat 1                                 # r 列=运行队列，b 列=阻塞(D)进程
```
