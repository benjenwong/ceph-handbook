# 3. 集群服务启停命令
------------

### 3.1 列出节点上所有的 Ceph systemd units

	sudo systemctl status ceph\*.service ceph\*.target

### 3.3 启动所有守护进程

要启动某一 Ceph 节点上的所有守护进程，用下列命令：

	sudo systemctl start ceph.target

### 3.3 停止所有守护进程

要停止某一 Ceph 节点上的所有守护进程，用下列命令：

	sudo systemctl stop ceph\*.service ceph\*.target

### 3.4 按类型启动所有守护进程

要启动某一 Ceph 节点上的某一类守护进程，用下列命令：

	sudo systemctl start ceph-osd.target
	sudo systemctl start ceph-mon.target
	sudo systemctl start ceph-mds.target
	sudo systemctl start ceph-radosgw.target
	sudo systemctl start ceph-mgr.target

### 3.5 按类型停止所有守护进程

要停止某一 Ceph 节点上的某一类守护进程，用下列命令：

	sudo systemctl stop ceph-mon\*.service ceph-mon.target
	sudo systemctl stop ceph-osd\*.service ceph-osd.target
	sudo systemctl stop ceph-mds\*.service ceph-mds.target
	sudo systemctl stop ceph-mgr\*.service ceph-mgr.target
	sudo systemctl stop ceph-radosgw\*.service ceph-radosgw.target

### 3.6 启动单个进程

要启动某节点上一个特定的守护进程例程，用下列命令之一：

	sudo systemctl start ceph-osd@{id}
	sudo systemctl start ceph-mon@{hostname}
	sudo systemctl start ceph-mds@{hostname}
	sudo systemctl start ceph-mgr@{hostname}
	sudo systemctl start ceph-radosgw@rgw.{hostname}

### 3.7 停止单个进程

要停止某节点上一个特定的守护进程例程，用下列命令之一：

	sudo systemctl stop ceph-osd@{id}
	sudo systemctl stop ceph-mon@{hostname}
	sudo systemctl stop ceph-mds@{hostname}
	sudo systemctl stop ceph-mgr@{hostname}
	sudo systemctl stop ceph-radosgw@rgw.{hostname}