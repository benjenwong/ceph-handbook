# 5. 增加/删除 OSD

----------

如果您的集群已经在运行，你可以在运行时添加或删除 OSD 。

### 5.1 增加 OSD（ ceph-deploy ）

还可以通过 `ceph-deploy` 工具很方便的增加 OSD。

1、登入 `ceph-deploy` 工具所在的 Ceph admin 节点，进入工作目录。

	ssh {ceph-deploy-node}
	cd /path/ceph-deploy-work-path

2、准备 OSD

    ceph-deploy osd prepare {node-name}:{data-disk}[:{journal-disk}] --filestore
    ceph-deploy osd prepare osdserver1:sdb:/dev/ssd --filestore
	ceph-deploy osd prepare osdserver1:sdc:/dev/ssd --filestore

**重要:** L版ceph-deploy默认后端文件系统类型为blustore，需手动加上 `--filestore` 指定为filestore

`prepare` 命令只准备 OSD 。在大多数操作系统中，硬盘分区创建后，不用 `activate` 命令也会自动执行 `activate` 阶段（通过 Ceph 的 `udev` 规则）。

3、准备好 OSD 后，可以用下列命令激活它。

	ceph-deploy osd activate {node-name}:{data-disk-partition}
	ceph-deploy osd activate osdserver1:/dev/sdb1
	ceph-deploy osd activate osdserver1:/dev/sdc1

`activate` 命令会让 OSD 进入 `up` 且 `in` 状态。该命令使用的分区路径是前面 `prepare` 命令创建的。

### 7.3 删除 OSD（手动）

要想缩减集群尺寸或替换硬件，可在运行时删除 OSD 。在 Ceph 里，一个 OSD 通常是一台主机上的一个 `ceph-osd` 守护进程、它运行在一个硬盘之上。如果一台主机上有多个数据盘，你得逐个删除其对应 `ceph-osd` 。通常，操作前应该检查集群容量，看是否快达到上限了，确保删除 OSD 后不会使集群达到 `near full` 比率。

**警告：** 删除 OSD 时不要让集群达到 `full ratio` 值，删除 OSD 可能导致集群达到或超过 `full ratio` 值。

1、停止需要剔除的 OSD 进程，让其他的 OSD 知道这个 OSD 不提供服务了。停止 OSD 后，状态变为 `down` 。

	ssh {osd-host}
    systemctl stop ceph-osd@{osd-num}

2、将 OSD 标记为 `out` 状态，这个一步是告诉 mon，这个 OSD 已经不能服务了，需要在其他的 OSD 上进行数据的均衡和恢复了。

	ceph osd out {osd-num}

执行完这一步后，会触发数据的恢复过程。此时应该等待数据恢复结束，集群恢复到 `HEALTH_OK` 状态，再进行下一步操作。

3、删除 CRUSH Map 中的对应 OSD 条目，它就不再接收数据了。

	ceph osd crush remove {name}

该步骤会触发数据的重新分布。等待数据重新分布结束，整个集群会恢复到 `HEALTH_OK` 状态。

4、删除 OSD 认证密钥：

	ceph auth del osd.{osd-num}

5、删除 OSD 。

	ceph osd rm {osd-num}
	# for example
	ceph osd rm 1

6、卸载 OSD 的挂载点。
	
	sudo umount /var/lib/ceph/osd/ceph-{osd-num}

如果删除一台主机，操作完以上6步删除完这台服务器所有osd之后，这台服务器的主机名仍然在crush map里面，这时候需要执行第7步

7、从osd crush map删除主机

	ceph osd crush remove ${HOSTNAME}