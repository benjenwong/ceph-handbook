# 2. 常见 OSD 故障处理

----------

进行 OSD 排障前，先检查一下 monitors 和网络。如果 `ceph health` 或 `ceph -s` 返回的是健康状态，这意味着 monitors 形成了法定人数。如果 monitor 还没达到法定人数、或者 monitor 状态错误，要先解决 monitor 的问题。核实下你的网络，确保它在正常运行，因为网络对 OSD 的运行和性能有显著影响。

### 2.1 收集 OSD 数据

开始 OSD 排障的第一步最好先收集信息，另外还有监控 OSD 时收集的，如 `ceph osd tree` 。

#### Ceph 日志

如果你没改默认路径，可以在 `/var/log/ceph` 下找到 Ceph 的日志：

	ls /var/log/ceph

如果看到的日志还不够详细，可以增大日志级别。请参考[1.12 日志和调试](../Operation/log_debug.md)，查阅如何保证看到大量日志又不影响集群运行。

#### 显示可用空间

可能会引起文件系统问题。用 `df` 命令显示文件系统的可用空间。

	df -h

其它用法见 `df --help` 。

#### I/O 统计信息

用 `iostat` 工具定位 I/O 相关问题。

	iostat -x

#### 诊断信息

要查看诊断信息，配合 `less` 、 `more` 、 `grep` 或 `tail` 使用 `dmesg` ，例如：

	dmesg | grep scsi

### 2.2 停止数据向外重平衡

你得周期性地对集群的子集进行维护，或解决某个故障域的问题（如某个机架）。如果你不想在停机维护 OSD 时让 CRUSH 自动重均衡，首先设置集群的 `noout` 标志：

	ceph osd set noout

设置了 noout 后，你就可以停机维护失败域内的 OSD 了。

	systemctl stop ceph-osd@{osd-num}

**注意：**在定位某故障域内的问题时，停机的 OSD 内的 PG 状态会变为 `degraded` 。

维护结束后，重启 OSD 。

	systemctl start ceph-osd@{osd-num}

最后，解除 `noout` 标志。

	ceph osd unset noout

### 2.3 OSD 没运行

通常情况下，简单地重启 `ceph-osd` 进程就可以让它重回集群并恢复。

#### OSD 起不来

如果你重启了集群，但其中一个 OSD 起不来，依次检查：

- **配置文件**： 如果你新装的 OSD 不能启动，检查下配置文件，确保它符合规定（比如 `host` 而非 `hostname` ，等等）。

- **检查路径**： 检查配置文件的路径，以及 OSD 的数据和日志分区路径。如果你分离了 OSD 的数据和日志分区、而配置文件和实际挂载点存在出入，启动 OSD 时就可能失败。如果你想把日志存储于一个块设备，应该为日志硬盘分区并为各 OSD 分别指定一个分区。

- **检查最大线程数**： 如果你的节点有很多 OSD ，也许就会触碰到默认的最大线程数限制（如通常是 32k 个），尤其是在恢复期间。你可以用 `sysctl` 增大线程数，把最大线程数更改为支持的最大值（即 4194303 ），看看是否有用。例如：

	sysctl -w kernel.pid_max=4194303

如果增大最大线程数解决了这个问题，你可以把此配置 `kernel.pid_max` 写入配置文件 `/etc/sysctl.conf`，使之永久生效，例如：

	kernel.pid_max = 4194303

#### OSD 失败

当 `ceph-osd` 挂掉时，monitor 可通过活着的 `ceph-osd` 了解到此情况，并通过 `ceph health` 命令报告：

	ceph health
	HEALTH_WARN 1/3 in osds are down

特别地，有 `ceph-osd` 进程标记为 `in` 且 `down` 的时候，你也会得到警告。你可以用下面的命令得知哪个 `ceph-osd` 进程挂了：

    ceph health detail
    HEALTH_WARN 1/3 in osds are down
    osd.0 is down since epoch 23, last address 192.168.106.220:6800/11080

如果有硬盘失败或其它错误使 `ceph-osd` 不能正常运行或重启，将会在日志文件 `/var/log/ceph/` 里输出一条错误信息。

如果守护进程因心跳失败、或者底层核心文件系统无响应而停止，查看 `dmesg` 获取硬盘或者内核错误。


### 2.4 OSD 龟速或无响应

一个反复出现的问题是 OSD 龟速或无响应。在深入性能问题前，你应该先确保不是其他故障。例如，确保你的网络运行正常、且 OSD 在运行，还要检查 OSD 是否被恢复流量拖住了。

**Tip：** 较新版本的 Ceph 能更好地处理恢复，可防止恢复进程耗尽系统资源而导致 `up` 且 `in` 的 OSD 不可用或响应慢。

#### 网络问题

Ceph 是一个分布式存储系统，所以它依赖于网络来互联 OSD 们、复制对象、从错误中恢复和检查心跳。网络问题会导致 OSD 延时和震荡（反复经历 up and down，详情可参考下文中的相关小节） 。

确保 Ceph 进程和 Ceph 依赖的进程已建立连接和/或在监听。

	netstat -a | grep ceph
	netstat -l | grep ceph
	sudo netstat -p | grep ceph

检查网络统计信息。

	netstat -s

#### 驱动器配置

一个存储驱动器应该只用于一个 OSD 。如果有其它进程共享驱动器，顺序读写吞吐量会成为瓶颈，包括日志、操作系统、monitor 、其它 OSD 和非 Ceph 进程。

Ceph 在日志记录完成*之后*才会确认写操作，所以使用 `ext4` 或 `XFS` 文件系统时高速的 SSD 对降低响应延时很有吸引力。与之相比， `btrfs` 文件系统可以同时读写日志和数据分区。

**注意：** 给驱动器分区并不能改变总吞吐量或顺序读写限制。把日志分离到单独的分区可能有帮助，但最好是另外一块硬盘的分区。


#### 日志记录级别

如果你为追踪某问题提高过日志级别，结束后又忘了调回去，这个 OSD 将向硬盘写入大量日志。如果你想始终保持高日志级别，可以考虑给默认日志路径（即 `/var/log/ceph/$cluster-$name.log` ）挂载一个单独的硬盘。

### OLD REQUESTS 或 SLOW REQUESTS

如果某 `ceph-osd` 守护进程对一请求响应很慢，它会生成日志消息来抱怨请求耗费的时间过长。

较新版本的 Ceph 抱怨 “slow requests” ：

	{date} {osd.num} [WRN] 1 slow requests, 1 included below; oldest blocked for > 30.005692 secs
	{date} {osd.num}  [WRN] slow request 30.005692 seconds old, received at {date-time}: osd_op(client.4240.0:8 benchmark_data_ceph-1_39426_object7 [write 0~4194304] 0.69848840) v4 currently waiting for subops from [610]

可能的原因有：

- 坏驱动器（查看 `dmesg` 输出）
- 内核文件系统缺陷（查看 `dmesg` 输出）
- 集群过载（检查系统负载、 iostat 等等）

可能的解决方法：

- 重启 OSD

### 2.5 震荡的 OSD

我们建议同时部署 public（前端）网络和 cluster（后端）网络，这样能更好地满足对象复制的网络性能需求。另一个优点是你可以运营一个不连接互联网的集群，以此避免某些拒绝服务攻击。 OSD 们互联和检查心跳时会优选 cluster（后端）网络。

然而，如果 cluster（后端）网络失败、或出现了明显的延时，同时 public（前端）网络却运行良好， OSD 目前不能很好地处理这种情况。这时 OSD 们会向 monitor 报告邻居 `down` 了、同时报告自己是 `up` 的，我们把这种情形称为震荡（ flapping ）。

如果有原因导致 OSD 震荡（反复地被标记为 `down` ，然后又 `up` ），你可以强制 monitor 停止这种震荡状态：

	ceph osd set noup      # prevent OSDs from getting marked up
	ceph osd set nodown    # prevent OSDs from getting marked down

这些标记记录在 osdmap 数据结构里：

	ceph osd dump | grep flags
	flags no-up,no-down

可用下列命令清除标记：

	ceph osd unset noup
	ceph osd unset nodown

Ceph 还支持另外两个标记 `noin` 和 `noout` ，它们可防止正在启动的 OSD 被标记为 `in` （可以分配数据），或被误标记为 `out` （不管 `mon osd down out interval` 的值是多少）。

**注意：** `noup` 、 `noout` 和 `nodown` 从某种意义上说是临时的，一旦标记被清除了，被它们阻塞的动作短时间内就会发生。另一方面， `noin` 标记阻止 OSD 启动后加入集群，但其它守护进程都维持原样。