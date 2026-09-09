# /proc 目录 与 drop_caches

> 2026-09-08 在服务器(Debian, 内核 6.12.107)实测验证。

## /proc 是什么

**procfs:内核暴露运行时状态的伪文件系统**。不占磁盘(`df -h /proc` 显示 size 0),文件内容是**读取时由内核实时生成**的,所以 `cat /proc/meminfo` 每次结果都可能不同。挂载点 `/proc`,类型 proc。

三大块内容:

### 1. 数字目录 = 每个 PID 一个进程

```
/proc/1/comm    → systemd(PID 1)
/proc/1/cmdline → 启动命令
/proc/PID/status、maps、fd/、environ ...
```

`/proc/self` 是魔法软链:谁访问它,它就指向谁的 PID。

### 2. 系统信息文件(只读)

- `/proc/meminfo` — `free` 命令的数据源
- `/proc/cpuinfo` — `lscpu` 的数据源
- `/proc/loadavg` — 1/5/15 分钟负载、运行/总进程数、最近 PID
- `/proc/uptime` — 开机秒数
- `/proc/net/`、`/proc/mounts` 等

### 3. /proc/sys/ = sysctl 可调参数(可写)

```
/proc/sys/vm/     → 内存管理(swap、脏页、缓存)
/proc/sys/net/    → 网络(tcp 参数、路由)
/proc/sys/kernel/ → 内核通用
```

改文件 = 改内核参数,等价于 `sysctl -w vm.drop_caches=1`;重启失效,持久化写 `/etc/sysctl.conf`。

## drop_caches 三种值

只写触发的动作开关(权限 `--w-------`,root 也 cat 不了,写入即触发回收):

- **echo 1** — 释放 pagecache(文件读写缓存)
- **echo 2** — 释放 dentry/inode(目录项和索引节点缓存,slab 的一部分)
- **echo 3** — 1+2 同时释放

前面的 `sync` 必要:缓存里有**脏页**(已改未落盘数据),直接 drop 只丢干净页,sync 强制先写回磁盘再清。

## 注意点(面试常考)

1. **不释放应用内存**。只回收 cache 这类"空闲时可挪用"的部分。`buff/cache` 内存吃紧时本来就会自动让位——手动 drop 通常没必要,反而让之后磁盘 IO 变慢(缓存要重建)。排查内存压力看 `MemAvailable`。
2. 真正的内存泄漏在 anonymous 内存,drop_caches 对它无效;内存真满时内核自己会回收缓存,回收不动就 OOM kill。

## 关联命令

```bash
free -h              # 总览(buff/cache、available)
cat /proc/meminfo    # 底层数据源
sysctl vm.swappiness # 查/改单个参数
cat /proc/mounts     # 确认 procfs 挂载
```
