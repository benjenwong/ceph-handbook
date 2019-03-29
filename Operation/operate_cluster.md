# 1. 操作集群 
------------

## 1.1 ceph组件说明

|服务名|用途|
|:-:|:-|
|OSDs|对象存储守护进程，负责数据存储、复制、恢复、平衡数据，并通过检查其他osd进程获取心跳，为ceph-mon提供信息|
|Monitors|维护集群状态的映射，包括mon映射，manager映射，OSD映射和CRUSH映射。负责管理osd和客户端之间的身份验证|
|MDSs|Ceph文件系统存储元数据，Ceph的元数据服务器允许POSIX文件系统的用户来执行基本的命令(例如ls，find)占用资源比较少|
|MGRS|负责跟踪运行时指标和Ceph集群的当前状态，包括存储利用率，当前性能指标和系统负载。Ceph Manager还依靠python的模块来管理和发布Ceph集群信息，例如通过Ceph Dashboard和 REST API|
|RGWS|Ceph对象存储网关，它是一个用于与Ceph存储集群交互的HTTP服务器。由于它提供了与OpenStack Swift和Amazon S3兼容的接口，因此Ceph对象网关具有自己的用户管理。Ceph对象网关可以将数据存储在用于存储来自Ceph文件系统客户端或Ceph块设备客户端的数据的相同Ceph存储集群中。|


## 1.2 用 SYSTEMD 控制 CEPH


### 1.2.1 列出节点上所有的 Ceph systemd units

	sudo systemctl status ceph\*.service ceph\*.target

### 1.2.2 启动所有守护进程

要启动某一 Ceph 节点上的所有守护进程，用下列命令：

	sudo systemctl start ceph.target

### 1.2.3 停止所有守护进程

要停止某一 Ceph 节点上的所有守护进程，用下列命令：

	sudo systemctl stop ceph\*.service ceph\*.target

### 1.2.4 按类型启动所有守护进程

要启动某一 Ceph 节点上的某一类守护进程，用下列命令：

	sudo systemctl start ceph-osd.target
	sudo systemctl start ceph-mon.target
	sudo systemctl start ceph-mds.target
	sudo systemctl start ceph-radosgw.target
	sudo systemctl start ceph-mgr.target

### 1.2.5 按类型停止所有守护进程

要停止某一 Ceph 节点上的某一类守护进程，用下列命令：

	sudo systemctl stop ceph-mon\*.service ceph-mon.target
	sudo systemctl stop ceph-osd\*.service ceph-osd.target
	sudo systemctl stop ceph-mds\*.service ceph-mds.target
	sudo systemctl stop ceph-mgr\*.service ceph-mgr.target
	sudo systemctl stop ceph-radosgw\*.service ceph-radosgw.target

### 1.2.6 启动单个进程

要启动某节点上一个特定的守护进程例程，用下列命令之一：

	sudo systemctl start ceph-osd@{id}
	sudo systemctl start ceph-mon@{hostname}
	sudo systemctl start ceph-mds@{hostname}
	sudo systemctl start ceph-mgr@{hostname}
	sudo systemctl start ceph-radosgw@rgw.{hostname}

### 1.2.7 停止单个进程

要停止某节点上一个特定的守护进程例程，用下列命令之一：

	sudo systemctl stop ceph-osd@{id}
	sudo systemctl stop ceph-mon@{hostname}
	sudo systemctl stop ceph-mds@{hostname}
	sudo systemctl stop ceph-mgr@{hostname}
	sudo systemctl stop ceph-radosgw@rgw.{hostname}