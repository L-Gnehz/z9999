
## 1. 场景描述

一台 RedHat 服务器存在两张网卡：

| 网卡     | IP 地址         | 说明     |
| ------ | ------------- | ------ |
| ens192 | 10.10.10.1/24 | 默认路由出口 |
| ens224 | 10.20.20.1/24 | 第二网卡   |

默认路由指向 `ens192`。

需求：

- 从任意网段都能 `ping` 通 `10.10.10.1`
- 从任意网段都能 `ping` 通 `10.20.20.1`

> 前提：客户端到服务器两个 IP 的路由可达，且中间防火墙允许 ICMP。

---

## 2. 核心结论

RHEL/CentOS 默认启用严格反向路径过滤 `rp_filter=1`，在多网卡、非对称路由场景下，可能导致从非默认路由网卡进入的 ICMP 请求被丢弃。

解决方式：

```ini
net.ipv4.conf.all.rp_filter = 2
```

即把反向路径过滤从“严格模式”调整为“宽松模式”。

---

## 3. 原理说明

### 3.1 rp_filter 只负责入站验证

当 ping 请求到达某张网卡时，内核会做反向路径检查：

- 取数据包的源 IP
- 查路由表，判断回包给这个源 IP 会从哪个网卡出去
- 比较“入站网卡”和“回程出口网卡”是否一致

严格模式 `rp_filter=1`：

- 要求入站网卡必须是最优回程出口
- 如果不一致，直接丢弃数据包

宽松模式 `rp_filter=2`：

- 只要源 IP 在路由表中有任意可达路径即可
- 不再要求入站网卡必须等于最优回程出口
- 允许非对称路由

### 3.2 出站路由独立决策

`rp_filter` 不影响回包从哪张网卡发出。

内核回复 ICMP 时，会重新查路由表，只根据目的 IP 选择出口。

因此会出现：

- 请求从 `ens224` 进入
- 回包从 `ens192` 默认路由出去

这就是典型的非对称路由。

### 3.3 为什么默认路由是 ens192，配置后 ens224 也能通？

以跨网段访问为例：从 `10.10.10.0/24` 网段 `ping 10.20.20.1`。

请求包到达 `ens224`：

- 严格模式下：内核发现回包给 `10.10.10.x` 的最优出口是 `ens192`，与入站网卡 `ens224` 不一致，丢弃。
- 宽松模式下：只要 `10.10.10.x` 在路由表中可达，就放行。

回包阶段：

- 内核查路由表，发现 `10.10.10.0/24` 直连 `ens192`
- 回包从 `ens192` 发出

所以：

> `rp_filter=2` 解决的是“入站包不被丢弃”，  
> 默认路由解决的是“回包从 ens192 出去”。  
> 两者职责不同，因此配置后 ens224 也能通。

---

## 4. 配置步骤

### 4.1 临时生效（测试用）

```bash
sysctl -w net.ipv4.conf.all.rp_filter=2
```

可选同时设置具体网卡：

```bash
sysctl -w net.ipv4.conf.ens192.rp_filter=2
sysctl -w net.ipv4.conf.ens224.rp_filter=2
```

### 4.2 永久生效（推荐）

创建或编辑配置文件：

```bash
vi /etc/sysctl.d/99-multi-nic.conf
```

写入：

```ini
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.ens192.rp_filter = 2
net.ipv4.conf.ens224.rp_filter = 2
```

加载配置：

```bash
sysctl -p /etc/sysctl.d/99-multi-nic.conf
```

---

## 5. 可选优化：ARP 参数

多网卡环境中，建议优化 ARP 行为，避免 ARP Flux 问题。

```ini
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
```

说明：

| 参数 | 建议值 | 作用 |
|---|---|---|
| `arp_ignore` | 1 | 仅当 ARP 请求的目标 IP 配置在入站网卡上时才应答 |
| `arp_announce` | 2 | 发送 ARP 请求时优先使用与目标同子网的接口地址 |

完整配置示例：

```ini
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.ens192.rp_filter = 2
net.ipv4.conf.ens224.rp_filter = 2

net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2
```

加载：

```bash
sysctl -p /etc/sysctl.d/99-multi-nic.conf
```

---

## 6. 关于 IP 转发

如果服务器只需要自身被 ping 通，不需要作为路由器转发其他设备流量，则不需要开启：

```ini
net.ipv4.ip_forward = 1
```

上述 `rp_filter` 和 ARP 参数调整已足够解决本场景连接性问题。

---

## 7. 验证方法

### 7.1 查看内核参数

```bash
sysctl net.ipv4.conf.all.rp_filter
sysctl net.ipv4.conf.ens192.rp_filter
sysctl net.ipv4.conf.ens224.rp_filter
```

### 7.2 查看路由

```bash
ip route show
ip route show default
```

### 7.3 跨网段 Ping 测试

从 `10.10.10.0/24` 网段客户端：

```bash
ping 10.20.20.1
```

从 `10.20.20.0/24` 网段客户端：

```bash
ping 10.10.10.1
```

两个方向均应成功。

### 7.4 抓包排查（可选）

```bash
tcpdump -i ens224 icmp
tcpdump -i ens192 icmp
```

可观察请求从哪张网卡进入、回包从哪张网卡发出。

---

## 8. 配置总结

| 参数 | 建议值 | 作用 |
|---|---|---|
| `net.ipv4.conf.all.rp_filter` | 2 | 关闭严格反向路径过滤，允许非对称路由 |
| `net.ipv4.conf.ens192.rp_filter` | 2 | 对 ens192 使用宽松反向路径过滤 |
| `net.ipv4.conf.ens224.rp_filter` | 2 | 对 ens224 使用宽松反向路径过滤 |
| `net.ipv4.conf.all.arp_ignore` | 1 | 避免错误网卡应答 ARP |
| `net.ipv4.conf.all.arp_announce` | 2 | 优化 ARP 请求源地址选择 |
| `net.ipv4.ip_forward` | 按需 | 仅当服务器需要转发流量时开启 |

---

## 9. 一句话记忆

> `rp_filter=2` 让从 `ens224` 进来的包不被内核因“回程出口不是 ens224”而丢弃；  
> 回包从哪出去仍由路由表决定，默认路由是 `ens192`，所以回包从 `ens192` 出去。  
> 这就是配置后 `ens224` 也能 ping 通的原因。

---

## 10. 注意事项

1. `rp_filter=2` 会放宽反向路径校验，生产环境需评估安全影响。
2. “任意网段”要求上游网络路由可达，且防火墙允许 ICMP。
3. 如果客户端或中间防火墙有严格状态检测，可能对非对称路由敏感，需要额外确认。
4. 若需要强制回程也走同一网卡，可进一步考虑策略路由，但本场景通常不需要。