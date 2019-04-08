# 1. 常见 MON 故障处理

----------

Monitor 维护着 Ceph 集群的信息，如果 Monitor 无法正常提供服务，那整个 Ceph 集群就不可访问。一般来说，在实际运行中，Ceph Monitor的个数是 2n + 1 ( n >= 0) 个，在线上至少3个，只要正常的节点数 >= n+1，Ceph 的 Paxos 算法就能保证系统的正常运行。所以，当 Monitor 出现故障的时候，不要惊慌，冷静下来，一步一步地处理。

### 1.1 开始排障

在遭遇 Monitor 故障时，首先回答下列几个问题：

**Mon 进程在运行吗？**

我们首先要确保 Mon 进程是在正常运行的。很多人往往忽略了这一点。

**是否可以连接 Mon Server？**

有时候我们开启了防火墙，导致无法与 Monitor 的 IP 或端口进行通信。尝试使用 `ssh` 连接服务器，如果成功，再尝试用其他工具（如 `telnet` ， `nc` 等）连接 monitor 的端口。

**ceph -s 命令是否能运行并收到集群回复？**

如果答案是肯定的，那么你的集群已启动并运行着。你可以认为如果已经形成法定人数，monitors 就只会响应 `status` 请求。
  
如果 `ceph -s` 阻塞了，并没有收到集群的响应且输出了很多 `fault` 信息，很可能此时你的 monitors 全部都 down 掉了或只有部分在运行（但数量不足以形成法定人数）。

**ceph -s 没完成是什么情况？**

如果你到目前为止没有完成前述的几步，请返回去完成。然后你需要 `ssh` 登录到服务器上并使用 monitor 的管理套接字。

### 1.2 使用 Mon 的管理套接字

通过管理套接字，你可以用 Unix 套接字文件直接与指定守护进程交互。这个文件位于你 Mon 节点的 `run` 目录下，默认配置它位于 `/var/run/ceph/ceph-mon.ID.asok `，但要是改过配置就不一定在那里了。如果你在那里没找到它，请检查 `ceph.conf` 里是否配置了其它路径，或者用下面的命令获取：

    ceph-conf --name mon.ID --show-config-value admin_socket

请牢记，只有在 Mon 运行时管理套接字才可用。Mon 正常关闭时，管理套接字会被删除；如果 Mon 不运行了、但管理套接字还存在，就说明 Mon 不是正常关闭的。不管怎样，Mon 没在运行，你就不能使用管理套接字， `ceph` 命令会返回类似 `Error 111: Connection Refused` 的错误消息。

访问管理套接字很简单，就是让 `ceph` 工具使用 `asok` 文件。对于 Dumpling 之前的版本，命令是这样的：

	ceph --admin-daemon /var/run/ceph/ceph-mon.<id>.asok <command>

你也可以用另一个（推荐的）命令：

	ceph daemon mon.<id> <command>

`ceph` 工具的 `help` 命令会显示管理套接字支持的其它命令。请仔细了解一下 `config get` 、 `config show` 、 `mon_status` 和 `quorum_status` 命令，在排除 Mon 故障时它们会很有用。

### 1.3 理解 MON_STATUS

当集群形成法定人数后，或在没有形成法定人数时通过管理套接字， 用 `ceph` 工具可以获得 `mon_status` 信息。命令会输出关于 monitor 的大多数信息，包括部分 `quorum_status` 命令的输出内容。

	ceph mon_status -f json-pretty

下面是 `mon_status` 的输出样例：

	{
		"name": "c",
  		"rank": 2,
  		"state": "peon",
  		"election_epoch": 38,
  		"quorum": [
        	1,
        	2
		],
  		"outside_quorum": [],
  		"extra_probe_peers": [],
  		"sync_provider": [],
  		"monmap": { 
			"epoch": 3,
      		"fsid": "5c4e9d53-e2e1-478a-8061-f543f8be4cf8",
      		"modified": "2013-10-30 04:12:01.945629",
      		"created": "2013-10-29 14:14:41.914786",
      		"mons": [
            	{ 	
					"rank": 0,
              		"name": "a",
              		"addr": "127.0.0.1:6789\/0"
				},
            	{ 
					"rank": 1,
              		"name": "b",
              		"addr": "127.0.0.1:6790\/0"
				},
            	{ 
					"rank": 2,
              		"name": "c",
              		"addr": "127.0.0.1:6795\/0"
				}
			]
		}
	}

从上面的信息可以看出， monmap 中包含 3 个monitor （ *a*，*b* 和 *c*），只有 2 个 monitor 形成了法定人数， *c* 是法定人数中的 *peon* 角色（非 *leader* 角色）。

还可以看出， **a** 并不在法定人数之中。请看 `quorum` 集合。在集合中只有 *1* 和 *2* 。这不是 monitor 的名字，而是它们加入当前 monmap 后确定的等级。丢失了等级 0 的 monitor，根据 monmap ，这个 monitor 就是 `mon.a` 。

那么，monitor 的等级是如何确定的？

当加入或删除 monitor 时，会（重新）计算等级。计算时遵循一个简单的规则： `IP:PORT` 的组合值越**大**， 等级越**低**（等级数字越大，级别越低）。因此在上例中， `127.0.0.1:6789` 比其他 `IP:PORT` 的组合值都小，所以 `mon.a` 的等级是 0 。

### 1.4 最常见的 Mon 问题

#### 达到了法定人数但是有至少一个 Monitor 处于 Down 状态

当此种情况发生时，根据你运行的 Ceph 版本，可能看到类似下面的输出：

	root@OPS-ceph1:~# ceph health detail
	HEALTH_WARN 1 mons down, quorum 1,2 b,c
	mon.a (rank 0) addr 127.0.0.1:6789/0 is down (out of quorum)

##### 如何解决？

首先，确认 mon.a 进程是否运行。

其次，确定可以从其他 monitor 节点连到 `mon.a` 所在节点。同时检查下端口。如果开了防火墙，还需要检查下所有 monitor 节点的 `iptables` ，以确定没有丢弃/拒绝连接。

如果前两步没有解决问题，请继续往下走。

首先，通过管理套接字检查问题 monitor 的 `mon_status` 。考虑到该 monitor 并不在法定人数中，它的状态应该是  `probing` ， `electing` 或 `synchronizing` 中的一种。如果它恰巧是 `leader` 或 `peon` 角色，它会认为自己在法定人数中，但集群中其他 monitor 并不这样认为。或者在我们处理故障的过程中它加入了法定人数，所以再次使用 `ceph -s` 确认下集群状态。如果该 monitor 还没加入法定人数，继续。

**`probing` 状态是什么情况？**

这意味着该 monitor 还在搜寻其他 monitors 。每次你启动一个 monitor，它会去搜寻 `monmap` 中的其他 monitors ，所以会有一段时间处于该状态。此段时间的长短不一。例如，单节点 monitor 环境， monitor 几乎会立即通过该阶段。在多 monitor 环境中，monitors 在找到足够的节点形成法定人数之前，都会处于该状态，这意味着如果 3 个 monitor 中的 2 个 down 了，剩下的 1 个会一直处于该状态，直到你再启动一个 monitor 。

如果你的环境已经形成法定人数，只要 monitor 之间可以互通，新 monitor 应该可以很快搜寻到其他 monitors 。如果卡在 probing 状态，并且排除了连接的问题，那很有可能是该 monitor 在尝试连接一个错误的 monitor 地址。可以根据 `mon_status` 命令输出中的 `monmap` 内容，检查其他 monitor 的地址是否和实际相符。如果不相符，请跳至**恢复 Monitor 损坏的 monmap**。如果相符，这些 monitor 节点间可能存在严重的时钟偏移问题，请首先参考**时钟偏移**，如果没有解决问题，可以搜集相关的日志并向社区求助。

**`electing` 状态是什么情况？**

这意味着该 monitor 处于选举过程中。选举应该很快就可以完成，但偶尔也会卡住，这往往是 monitors 节点时钟偏移的一个标志，跳转至**时钟偏移**获取更多信息。如果时钟是正确同步的，可以搜集相关日志并向社区求助。此种情况除非是一些（*非常*）古老的 bug ，往往都是由时钟不同步引起的。


#### 恢复 Monitor 损坏的 monmap

monmap 通常看起来是下面的样子，这取决于 monitor 的个数：

    epoch 3
    fsid 5c4e9d53-e2e1-478a-8061-f543f8be4cf8
    last_changed 2013-10-30 04:12:01.945629
    created 2013-10-29 14:14:41.914786
    0: 127.0.0.1:6789/0 mon.a
    1: 127.0.0.1:6790/0 mon.b
    2: 127.0.0.1:6795/0 mon.c

不过也不一定就是这样的内容。比如，在早期版本的 Ceph 中，有一个严重 bug 会导致 `monmap` 的内容全为 0 。这意味着即使用 `monmaptool` 也不能读取它，因为全 0 的内容没有任何意义。另外一些情况，某个 monitor 所持有的 monmap 已严重过期，以至于无法搜寻到集群中的其他 monitors 。在这些状况下，你有两种可行的解决方法：

#### 时钟偏移

Monitor 节点间明显的时钟偏移会对 monitor 造成严重的影响。这通常会导致一些奇怪的问题。为了避免这些问题，在 monitor 节点上应该运行时间同步工具。

**允许的最大时钟偏移量是多少？**

默认最大允许的时钟偏移量是 `0.05 秒`。

**如何增加最大时钟偏移量？**

通过 mon-clock-drift-allowed 选项来配置。尽管你 ***可以*** 修改但不代表你 ***应该*** 修改。时钟偏移机制之所以是合理的，是因为有时钟偏移的 monitor 可能会表现不正常。未经测试而修改该值，尽管没有丢失数据的风险，但仍可能会对 monitors 的稳定性和集群的健康造成不可预知的影响。

**如何知道是否存在时钟偏移？**

Monitor 会用 `HEALTH_WARN` 的方式警告你。 `ceph health detail` 应该输出如下格式的信息：

	mon.c addr 10.10.0.1:6789/0 clock skew 0.08235s > max 0.05s (latency 0.0045s)

这表示 `mon.c` 已被标记出正在遭受时钟偏移。

**如果存在时钟偏移该怎么处理？**

同步各 monitor 节点的时钟。运行 NTP 客户端会有帮助。如果你已经启动了 NTP 服务，但仍遭遇此问题，检查一下使用的 NTP 服务器是否离你的网络太过遥远，然后可以考虑在你的网络环境中运行自己的 NTP 服务器。最后这种选择可趋于减少 monitor 时钟偏移带来的问题。

#### 磁盘空间不足导致 MON DOWN

当 monitor 进程检测到本地可用磁盘空间不足时，会停止 monitor 服务。Monitor 的日志中应该会有类似如下信息的输出：

	2016-09-01 16:45:54.994488 7fb1cac09700  0 mon.jyceph01@0(leader).data_health(62) update_stats avail 5% total 297 GB, used 264 GB, avail 18107 MB
	2016-09-01 16:45:54.994747 7fb1cac09700 -1 mon.jyceph01@0(leader).data_health(62) reached critical levels of available space on local monitor storage -- shutdown!

清理本地磁盘，增大可用空间，重启 monitor 进程，即可恢复正常。