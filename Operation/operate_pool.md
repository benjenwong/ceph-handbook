# 7. 操作 Pool

----------

如果你开始部署集群时没有创建存储池， Ceph 会用默认存储池 rbd 存放数据。存储池提供的功能：

- **自恢复力：** 你可以设置在不丢数据的前提下允许多少 OSD 失效。对多副本存储池来说，此值是一对象应达到的副本数。典型配置是存储一个对象和它的一个副本（即 size = 2 ）。
- **归置组：** 你可以设置一个存储池的 PG 数量。典型配置给每个 OSD 分配大约 100 个 PG，这样，不用过多计算资源就能得到较优的均衡。配置了多个存储池时，要考虑到这些存储池和整个集群的 PG 数量要合理。
- **CRUSH 规则：** 当你在存储池里存数据的时候，与此存储池相关联的 CRUSH 规则集可控制 CRUSH 算法，并以此操纵集群内对象及其副本的复制。
- **快照：** 用 `ceph osd pool mksnap` 创建快照的时候，实际上创建了某一特定存储池的快照。

要把数据组织到存储池里，你可以列出、创建、删除存储池，也可以查看每个存储池的使用统计数据。

### 7.1 列出存储池
要列出集群的存储池，命令如下：

	ceph osd lspools

在新安装好的集群上，默认只有一个 rbd 存储池。

### 7.2 创建存储池

要创建一个存储池，执行：

	ceph osd pool create {pool-name} {pg-num} [{pgp-num}] 

各参数含义如下：

- `{pool-name}`：存储池名称，必须唯一。
- `{pg_num}`： 存储池的 PG 数目。
- `{pgp_num}`： 存储池的 PGP 数目，此值应该和 PG 数目相等。

关于如何计算合适的 `pg_num` 值，可以使用 Ceph 官方提供的一个计算工具 [**pgcalc**](http://ceph.com/pgcalc/) 。


### 7.3 删除存储池

要删除一存储池，执行：

	ceph osd pool delete {pool-name} [{pool-name} --yes-i-really-really-mean-it]

### 7.4 重命名存储池

要重命名一个存储池，执行：

	ceph osd pool rename {current-pool-name} {new-pool-name}

如果重命名了一个存储池，且认证用户对每个存储池都有访问权限，那你必须用新存储池名字更新用户的能力（即 caps ）。

### 7.5 查看存储池统计信息

要查看某存储池的使用统计信息，执行命令：

	rados df


### 7.6 获取存储池选项值

要获取一个存储池的选项值，执行命令：

	ceph osd pool get {pool-name} {key}

### 7.7 调整存储池选项值

要设置一个存储池的选项值，执行命令：

	ceph osd pool set {pool-name} {key} {value}

常用选项介绍：

- `size`：设置存储池中的对象副本数，详情参见设置对象副本数。仅适用于副本存储池。
- `min_size`：设置 I/O 需要的最小副本数，详情参见设置对象副本数。仅适用于副本存储池。
- `pg_num`：计算数据分布时的有效 PG 数。只能大于当前 PG 数。
- `pgp_num`：计算数据分布时使用的有效 PGP 数量。小于等于存储池的 PG 数。
- `crush_ruleset`：
- `hashpspool`：给指定存储池设置/取消 HASHPSPOOL 标志。
- `target_max_bytes`：达到 `max_bytes` 阀值时会触发 Ceph 冲洗或驱逐对象。
- `target_max_objects`：达到 `max_objects` 阀值时会触发 Ceph 冲洗或驱逐对象。
- `scrub_min_interval`：在负载低时，洗刷存储池的最小间隔秒数。如果是 0 ，就按照配置文件里的 osd_scrub_min_interval 。
- `scrub_max_interval`：不管集群负载如何，都要洗刷存储池的最大间隔秒数。如果是 0 ，就按照配置文件里的 osd_scrub_max_interval 。
- `deep_scrub_interval`：“深度”洗刷存储池的间隔秒数。如果是 0 ，就按照配置文件里的 osd_deep_scrub_interval 。

### 7.8 设置对象副本数

要设置多副本存储池的对象副本数，执行命令：

	ceph osd pool set {poolname} size {num-replicas}

**重要：** `{num-replicas}` 包括对象自身，如果你想要对象自身及其两份拷贝共计三份，指定 size 为 3 。

例如：

	ceph osd pool set data size 3

你可以在每个存储池上执行这个命令。注意，一个处于降级模式的对象，其副本数小于 `pool size` ，但仍可接受 I/O 请求。为保证 I/O 正常，可用 `min_size` 选项为其设置个最低副本数。例如：

	ceph osd pool set data min_size 2

这确保数据存储池里任何副本数小于 `min_size` 的对象都不会收到 I/O 了。

### 7.9 获取对象副本数

要获取对象副本数，执行命令：

	ceph osd dump | grep 'replicated size'

Ceph 会列出存储池，且高亮 `replicated size` 属性。默认情况下， Ceph 会创建一对象的两个副本（一共三个副本，或 size 值为 3 ）。