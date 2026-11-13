
**报告日期**：2026-10-16  
**分析对象**：localhost（RHEL 7.6，Oracle RAC 节点）  
**数据来源**：sosreport-localhost-2026-10-16-buyxklm  
**故障现象**：整机无响应（Hang），持续约数分钟后自行恢复，未重启


## 1. 故障概述

2026-10-16 上午，主机 `localhost` 出现整机无响应。系统日志显示，在 10:42:30 至 10:44:30 期间，内核连续报告多个 Oracle RAC 相关进程（`orarootagent.bi`、`gipcd.bin`）进入不可中断睡眠状态（D 状态），触发 hung task 检测机制。系统未发生重启，`journalctl --list-boots` 显示仅有一次启动记录（10:39:29 至 16:02:53），说明系统在 hang 后自行恢复。

**核心结论**：本次 hang 并非由存储 I/O、NFS、OOM 或内核崩溃直接引起，而是**进程在内存管理路径（page fault）上等待 `mmap_sem` 读写信号量时被长时间阻塞**。结合内核版本与 Red Hat 官方缺陷记录，高度怀疑是 RHEL 7.6 初始内核 `3.10.0-957.el7.x86_64` 中 `rwsem` 实现存在的已知唤醒丢失缺陷（Red Hat Bugzilla 1690323 / 1547078）所致。该缺陷已在 `kernel-3.10.0-957.12.1.el7` 中修复。


## 2. 系统环境

| 项目   | 内容                                                                                               |
| ---- | ------------------------------------------------------------------------------------------------ |
| 主机名  | localhost                                                                                        |
| 操作系统 | Red Hat Enterprise Linux Server release 7.6 (Maipo)                                              |
| 内核版本 | 3.10.0-957.el7.x86_64（**未包含 rwsem 修复补丁**）                                                        |
| 启动参数 | `crashkernel=auto rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet transparent_hugepage=never` |
| 应用角色 | Oracle RAC 节点（存在 `orarootagent.bi`、`gipcd.bin` 进程）                                               |
| 启动时间 | 2026-10-16 10:39:29 CST                                                                          |
| 是否重启 | 否（仅一个 boot 记录）                                                                                   |


## 3. 故障时间线

| 时间 | 事件 |
|------|------|
| 10:38:59 | 移除 oracle 会话 1006203 |
| 10:39:29 | 内核启动 |
| 10:41:06 | `systemd-logind` 报告：`Failed to start session scope session-1006205.scope: Connection timed out`（hang 前兆） |
| 10:42:30 | 内核开始报告 `orarootagent.bi`（多个 PID）`blocked for more than 120 seconds` |
| 10:42:31 | 继续报告 `orarootagent.bi` 阻塞，其中一个进程在 `rwsem_down_write_failed` 上等待 |
| 10:44:25 | 启动 oracle 会话 1006205 |
| 10:44:30 | 内核报告 `gipcd.bin`（多个 PID）`blocked for more than 120 seconds` |
| 之后 | 系统自行恢复，未重启，日志持续至 16:02:53 |


## 4. 关键日志证据

### 4.1 Hung task 进程列表

被阻塞的进程均为 D 状态（不可中断睡眠），包括：

- `orarootagent.bi`：PID 371345、371383、371400、371401、371597、371833、413139
- `gipcd.bin`：PID 274011、274027、274033

### 4.2 典型 Call Trace（读锁等待）

绝大多数进程的调用栈高度一致：

```
Call Trace:
 [<ffffffff8c5e9134>] ? handle_pte_fault+0x2f4/0xd10
 [<ffffffff8cb67bc9>] schedule+0x29/0x70
 [<ffffffff8cb691fd>] rwsem_down_read_failed+0x10d/0x1a0
 [<ffffffff8c7869a8>] call_rwsem_down_read_failed+0x18/0x30
 [<ffffffff8cb66e80>] down_read+0x20/0x40
 [<ffffffff8cb6f7d8>] __do_page_fault+0x3f8/0x500
 [<ffffffff8cb6f915>] do_page_fault+0x35/0x90
 [<ffffffff8cb6b758>] page_fault+0x28/0x30
```

### 4.3 典型 Call Trace（写锁等待）

其中 PID 371833 的栈为写锁等待，路径为 `mmap`：

```
 [<ffffffff8cb67bc9>] schedule+0x29/0x70
 [<ffffffff8cb694b5>] rwsem_down_write_failed+0x225/0x3a0
 [<ffffffff8c65e75b>] ? destroy_inode+0x3b/0x60
 [<ffffffff8c7869d7>] call_rwsem_down_write_failed+0x17/0x30
 [<ffffffff8cb66ecd>] down_write+0x2d/0x3d
 [<ffffffff8c5d60e0>] vm_mmap_pgoff+0xa0/0x120
 [<ffffffff8c5f07c6>] SyS_mmap_pgoff+0x116/0x270
 [<ffffffff8c430c22>] SyS_mmap+0x22/0x30
 [<ffffffff8cb74ddb>] system_call_fastpath+0x22/0x27
```

### 4.4 排除性证据

- **无 OOM**：未出现 `Out of memory` 或 `oom-killer`。
- **无 soft/hard lockup**：未出现 `soft lockup`、`hard lockup`。
- **无内核 panic/Oops**：未出现 `BUG:`、`Oops`、`panic`。
- **无存储 I/O 错误**：未出现 `I/O error`、`blk_update_request`、`SCSI error`、`nvme timeout`。
- **无 NFS 问题**：未出现 `NFS: server not responding`、`RPC timeout`。
- **SATA link down**：大量 `ata*: SATA link down`，均为启动阶段或空闲端口，未伴随 I/O 错误，判断为干扰项。


## 5. 根因分析

### 5.1 直接原因：`mmap_sem` 锁竞争

所有被阻塞的进程都卡在**缺页异常（page fault）路径**上，需要获取进程地址空间的 `mmap_sem` 读锁（`down_read`）。其中一个进程在 `mmap` 系统调用中需要获取写锁（`down_write`）。当某个进程长时间持有 `mmap_sem` 写锁或读锁未释放时，后续所有需要访问该地址空间的进程都会在 `rwsem_down_read_failed` 或 `rwsem_down_write_failed` 中等待，最终被 hung task 检测机制捕获。

### 5.2 根本原因：内核 `rwsem` 已知缺陷

系统运行的内核版本为 `3.10.0-957.el7.x86_64`，这是 RHEL 7.6 的初始内核版本。Red Hat 官方缺陷记录如下：

| 项目 | 内容 |
|------|------|
| Bugzilla 编号 | **1690323**、**1547078** |
| 修复补丁标题 | `locking/rwsem: Fix (possible) missed wakeup` |
| 提交者 | Waiman Long（Red Hat） |
| 修复版本 | **`kernel-3.10.0-957.12.1.el7`** 及更高版本 |
| 当前版本 | `3.10.0-957.el7`，**不包含此修复** |

该缺陷会导致 `rwsem` 等待者无法被正确唤醒，从而无限期阻塞。本次故障的调用栈特征（`rwsem_down_read_failed` / `rwsem_down_write_failed` + `__do_page_fault` / `vm_mmap_pgoff`）与已知缺陷高度吻合。

### 5.3 相关 Red Hat 官方文档参考

以下文档均可在 Red Hat Customer Portal 直接访问，搜索时**直接使用 Solution ID**（而非 Bugzilla 编号）：

| Solution ID | 标题 | 与本次故障的关联 |
|-------------|------|------------------|
| **5782891** | RHEL7.9 standard (non-RT) kernel and RHEL7.9 Realtime (kernel-rt) crash due to blocked task detection | 文档描述 `unregister_shrinker()` 在尝试获取 `shrinker_rwsem` 时阻塞，与 Oracle 进程因信号量问题挂起的模式相似 |
| **2292461** | Why do the server got stuck for 30 minutes and then recovered back alone? | 文档调用栈包含 `rwsem_down_write_failed` 和 `vm_mmap_pgoff`，与 PID 371833 的 `mmap` 路径写锁等待高度一致 |

### 5.4 触发条件

Oracle RAC 节点上运行大量内存密集型进程，频繁进行 `mmap`、缺页异常和内存回收操作。当多个进程同时竞争 `mmap_sem` 时，若内核恰好触发 `rwsem` 唤醒丢失缺陷，就会导致锁无法释放，进而引发连锁阻塞，表现为整机无响应。


## 6. 影响范围

- **业务影响**：Oracle RAC 节点短时不可用，`orarootagent` 和 `gipcd` 等集群进程阻塞，可能影响数据库实例、集群心跳和节点间通信。
- **系统影响**：整机无响应，用户无法登录或执行操作，但系统未崩溃、未重启，数据未丢失。
- **持续时间**：从 10:41 左右出现前兆，至 10:44 后逐渐恢复，核心阻塞约 3 分钟。


## 7. 解决方案与建议

### 7.1 首选方案：升级内核

将内核升级至 **`3.10.0-957.12.1.el7`** 或更高版本（建议升级至 RHEL 7.9 最新稳定内核），以包含 `rwsem` 修复补丁（Bugzilla 1690323 / 1547078）。

```bash
yum update kernel
```

**注意事项**：
- Oracle RAC 环境需遵循 Oracle 认证的内核版本，升级前请与 Oracle 支持确认兼容性。
- 按滚动升级流程逐节点操作，避免同时重启所有节点。
- 升级后需重启系统生效。

### 7.2 临时缓解措施（无法立即升级时）

- 延长 hung task 超时时间，减少误报和恐慌：
  ```bash
  sysctl -w kernel.hung_task_timeout_secs=300
  echo "kernel.hung_task_timeout_secs = 300" >> /etc/sysctl.conf
  ```
- 减少警告次数：
  ```bash
  sysctl -w kernel.hung_task_warnings=5
  ```
- 调整内存回收倾向，降低 `mmap_sem` 竞争压力：
  ```bash
  sysctl -w vm.swappiness=10
  echo "vm.swappiness = 10" >> /etc/sysctl.conf
  ```
- 确认透明大页已禁用（当前启动参数已包含 `transparent_hugepage=never`，无需额外操作）。

### 7.3 监控与排查

- 监控 D 状态进程：
  ```bash
  ps -eo state,pid,ppid,wchan:32,cmd | awk '$1 ~ /D/'
  ```
- 定期检查内核日志中是否再次出现 `blocked for more than`：
  ```bash
  grep -i "blocked for more than" /var/log/messages
  ```
- 检查 Oracle 集群日志，确认 hang 期间是否触发集群重组或资源切换：
  ```bash
  tail -200 $ORACLE_HOME/log/*/alert*.log
  ```

### 7.4 长期建议

- 保持内核版本与 Red Hat 和 Oracle 的认证组合一致，及时应用安全与缺陷修复补丁。
- 对 Oracle RAC 节点进行内存压力监控，确保 `MemAvailable` 充足，避免频繁直接回收。
- 若问题复现，收集 `vmcore` 并提交 Red Hat 支持进行深度分析，同时引用 Bugzilla 1690323 / 1547078 和 Solution 5782891 / 2292461。


## 8. 结论

本次 `localhost` 整机 Hang 的直接原因是多个 Oracle RAC 进程在缺页异常路径上等待 `mmap_sem`，被 hung task 机制捕获。根本原因是 RHEL 7.6 初始内核 `3.10.0-957.el7.x86_64` 中 `rwsem` 实现存在已知唤醒丢失缺陷（Red Hat Bugzilla 1690323 / 1547078），该缺陷已在 `kernel-3.10.0-957.12.1.el7` 中修复。

**建议尽快升级内核至 `3.10.0-957.12.1.el7` 或更高版本**，并在升级前采取临时缓解措施，同时加强 Oracle 集群日志和内存压力的监控。


## 附录 A：关键日志片段

### A.1 Hung task（读锁等待）

```
Oct 16 10:42:30 localhost kernel: INFO: task orarootagent.bi:371345 blocked for more than 120 seconds.
Oct 16 10:42:30 localhost kernel: "echo 0 > /proc/sys/kernel/hung_task_timeout_secs" disables this message.
Oct 16 10:42:30 localhost kernel: orarootagent.bi D ffff948d1f39d140     0 371345      1 0x00000080
Oct 16 10:42:30 localhost kernel: Call Trace:
Oct 16 10:42:30 localhost kernel: [<ffffffff8c5e9134>] ? handle_pte_fault+0x2f4/0xd10
Oct 16 10:42:30 localhost kernel: [<ffffffff8cb67bc9>] schedule+0x29/0x70
Oct 16 10:42:30 localhost kernel: [<ffffffff8cb691fd>] rwsem_down_read_failed+0x10d/0x1a0
Oct 16 10:42:30 localhost kernel: [<ffffffff8c7869a8>] call_rwsem_down_read_failed+0x18/0x30
Oct 16 10:42:30 localhost kernel: [<ffffffff8cb66e80>] down_read+0x20/0x40
Oct 16 10:42:30 localhost kernel: [<ffffffff8cb6f7d8>] __do_page_fault+0x3f8/0x500
Oct 16 10:42:30 localhost kernel: [<ffffffff8cb6f915>] do_page_fault+0x35/0x90
Oct 16 10:42:30 localhost kernel: [<ffffffff8cb6b758>] page_fault+0x28/0x30
```

### A.2 Hung task（写锁等待）

```
Oct 16 10:42:31 localhost kernel: INFO: task orarootagent.bi:371833 blocked for more than 120 seconds.
Oct 16 10:42:31 localhost kernel: orarootagent.bi D ffff94b01e3430c0     0 371833      1 0x00000080
Oct 16 10:42:31 localhost kernel: Call Trace:
Oct 16 10:42:31 localhost kernel: [<ffffffff8cb67bc9>] schedule+0x29/0x70
Oct 16 10:42:31 localhost kernel: [<ffffffff8cb694b5>] rwsem_down_write_failed+0x225/0x3a0
Oct 16 10:42:31 localhost kernel: [<ffffffff8c65e75b>] ? destroy_inode+0x3b/0x60
Oct 16 10:42:31 localhost kernel: [<ffffffff8c7869d7>] call_rwsem_down_write_failed+0x17/0x30
Oct 16 10:42:31 localhost kernel: [<ffffffff8cb66ecd>] down_write+0x2d/0x3d
Oct 16 10:42:31 localhost kernel: [<ffffffff8c5d60e0>] vm_mmap_pgoff+0xa0/0x120
Oct 16 10:42:31 localhost kernel: [<ffffffff8c5f07c6>] SyS_mmap_pgoff+0x116/0x270
Oct 16 10:42:31 localhost kernel: [<ffffffff8c430c22>] SyS_mmap+0x22/0x30
Oct 16 10:42:31 localhost kernel: [<ffffffff8cb74ddb>] system_call_fastpath+0x22/0x27
```


## 附录 B：参考信息（含直接链接）

### B.1 Red Hat Bugzilla（缺陷跟踪，部分需登录）

| 编号 | 标题 | 链接 |
|------|------|------|
| **1690323** | `locking/rwsem: Fix (possible) missed wakeup`（主跟踪 Bug） | https://bugzilla.redhat.com/show_bug.cgi?id=1690323 |
| **1547078** | 同一修复的相关/重复 Bug | https://bugzilla.redhat.com/show_bug.cgi?id=1547078 |

### B.2 Red Hat Customer Portal Solution（知识库）

| Solution ID | 标题 | 链接 |
|-------------|------|------|
| **5782891** | RHEL7.9 standard (non-RT) kernel and RHEL7.9 Realtime (kernel-rt) crash due to blocked task detection | https://access.redhat.com/solutions/5782891 |
| **2292461** | Why do the server got stuck for 30 minutes and then recovered back alone? | https://access.redhat.com/solutions/2292461 |

### B.3 内核修复版本验证

在 Oracle Linux 的官方 changelog 中，可以明确看到 `kernel-3.10.0-957.12.1.el7` 包含了此修复：

- **链接**：https://yum.oracle.com/repowatch/oraclelinux7.x86_64/ol7_latest_archive/617a5ce5dacda1bee5fe9792ee5592f36a593561?display=changelog
- **Changelog 中对应条目**：
  ```
  [kernel] locking/rwsem: Fix (possible) missed wakeup (Waiman Long) [1690323 1547078]
  ```

### B.4 检索建议

- **Red Hat Customer Portal**：直接在搜索框输入 Solution ID（如 `5782891`、`2292461`），比输入 Bugzilla 编号更容易定位。
- **Bugzilla**：部分 Bug 可能因权限限制无法公开查看，但标题和编号可作为提交 SR 时引用。
- **Oracle Linux changelog**：无需登录，可直接查看，适合作为"补丁确实存在"的第三方佐证。


**报告结束**