# 2. 监控集群
------------

## 2.1 查看集群状态

ceph -s

输出如下,与输出解释如下

```bash
ceph -s
  cluster:
    id:     99138b56-113d-4347-8857-7bbb92ced5d1
    health: HEALTH_OK
 # 集群信息：集群uuid与状态

  services:
    mon: 1 daemons, quorum s3-9-30-17-169
    # mon: 个数 主机
    mgr: s3-9-30-17-169(active)
    # mgr: 主机
    osd: 6 osds: 6 up, 6 in
    # osd: 节点总数 up数 in数
    rgw: 1 daemon active
    # rgw: 服务个数
 # 已安装的服务信息
 
  data:
    pools:   16 pools, 12032 pgs
    # pool总数 pg总数
    objects: 3.11M objects, 86.4GiB
    # 对象总数 对象占用空间
    usage:   213GiB used, 1.40TiB / 1.61TiB avail
    # 数据使用量 可用量/总量
    pgs:     12032 active+clean
    # pg状态
# 数据信息
```

## 2.2 查看集群使用情况

要检查集群的数据用量及其在存储池内的分布情况，可以用 df 选项，它和 Linux 上的 df 相似。如下：

ceph df

输出如下:

```bash
GLOBAL:
    SIZE        AVAIL       RAW USED     %RAW USED 
    1.61TiB     1.40TiB       213GiB         12.97 
POOLS:
    NAME                         ID     USED        %USED     MAX AVAIL     OBJECTS 
    .rgw.root                    1      3.40KiB         0        673GiB          16 
    china.rgw.buckets.data       2      86.4GiB     11.38        673GiB     3113441 
    china.rgw.buckets.extra      3           0B         0        673GiB           0 
    china.rgw.buckets.index      4           0B         0        673GiB         115 
    china.rgw.buckets.non-ec     5           0B         0        673GiB           0 
    china.rgw.control            6           0B         0        673GiB           8 
    china.rgw.data.root          7           0B         0        673GiB           0 
    china.rgw.gc                 8           0B         0        673GiB           0 
    china.rgw.intent-log         9           0B         0        673GiB           0 
    china.rgw.log                10        149B         0        673GiB        1204 
    china.rgw.meta               11     2.70KiB         0        673GiB          14 
    china.rgw.usage              12          0B         0        673GiB           0 
    china.rgw.users.email        13          0B         0        673GiB           0 
    china.rgw.users.keys         14          0B         0        673GiB           0 
    china.rgw.users.swift        15          0B         0        673GiB           0 
    china.rgw.users.uid          16          0B         0        673GiB           0 
```

输出的 **GLOBAL** 段展示了数据所占用集群存储空间的概要。

- **SIZE：** 集群的总容量。
- **AVAIL：** 集群的可用空间总量。
- **RAW USED：**已用存储空间总量。
- **% RAW USED：**已用存储空间比率。用此值对比 `full ratio` 和 `near full ratio` 来确保不会用尽集群空间。

输出的 **POOLS** 段展示了存储池列表及各存储池的大致使用率。本段没有反映出副本、克隆和快照的占用情况。例如，如果你把 1MB 的数据存储为对象，理论使用率将是 1MB ，但考虑到副本数、克隆数、和快照数，实际使用量可能是 2MB 或更多。

- **NAME：**存储池名字。
- **ID：**存储池唯一标识符。
- **USED：**大概数据量，单位为 KB 、MB 或 GB ；
- **%USED：**各存储池的大概使用率。
- **MAX AVAIL：**最大可用量，一般为可用总量/副本数
- **Objects：**各存储池内的大概对象数。

注意： **POOLS** 段内的数字是估计值，它们不包含副本、快照或克隆。因此，各 Pool 的 **USED** 和 **%USED** 数量之和不会达到 **GLOBAL** 段中的 **RAW USED** 和 **%RAW USED** 数量。


## 2.3 查看osd状态

osd crush map查看

ceph osd tree

Ceph 会打印 CRUSH 树，包括 host 的名称、它上面的 OSD 例程、状态及权重：

```bash
ID CLASS WEIGHT  TYPE NAME               STATUS REWEIGHT PRI-AFF
-1       1.60739 root default
-3       1.60739     host s3-9-30-17-169
 0   ssd 0.26790         osd.0               up  1.00000 1.00000
 1   ssd 0.26790         osd.1               up  1.00000 1.00000
 2   ssd 0.26790         osd.2               up  1.00000 1.00000
 3   ssd 0.26790         osd.3               up  1.00000 1.00000
 4   ssd 0.26790         osd.4               up  1.00000 1.00000
 5   ssd 0.26790         osd.5               up  1.00000 1.00000
 ```

ceph osd dump

Ceph 会打印OSD MAP summary

```bash
epoch 58
# OSD CRUSH MAP版本
fsid 99138b56-113d-4347-8857-7bbb92ced5d1
created 2019-03-07 19:08:38.063429
modified 2019-04-03 10:36:23.140853
flags sortbitwise,recovery_deletes,purged_snapdirs
# 集群FLAGS
crush_version 13
full_ratio 0.95
# 当OSD数据使用率达到95%，ceph会将OSD标记为full
backfillfull_ratio 0.9
# 当OSD数据使用率达到90%，集群会将该OSD的数据backfill到其他OSD
nearfull_ratio 0.85
# 当OSD数据使用率达到85%，ceph会将OSD标记为nearfull
require_min_compat_client jewel
min_compat_client jewel
# 客户端兼容最低版本
require_osd_release luminous
# osd版本
pool 1 '.rgw.root' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 53 flags hashpspool stripe_width 0 application rgw
pool 2 'china.rgw.buckets.data' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 8192 pgp_num 8192 last_change 54 flags hashpspool stripe_width 0 application rgw
pool 3 'china.rgw.buckets.extra' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 28 flags hashpspool stripe_width 0
pool 4 'china.rgw.buckets.index' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 55 flags hashpspool stripe_width 0 application rgw
pool 5 'china.rgw.buckets.non-ec' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 30 flags hashpspool stripe_width 0
pool 6 'china.rgw.control' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 56 flags hashpspool stripe_width 0 application rgw
pool 7 'china.rgw.data.root' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 32 flags hashpspool stripe_width 0
pool 8 'china.rgw.gc' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 33 flags hashpspool stripe_width 0
pool 9 'china.rgw.intent-log' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 34 flags hashpspool stripe_width 0
pool 10 'china.rgw.log' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 57 flags hashpspool stripe_width 0 application rgw
pool 11 'china.rgw.meta' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 58 flags hashpspool stripe_width 0 application rgw
pool 12 'china.rgw.usage' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 37 flags hashpspool stripe_width 0
pool 13 'china.rgw.users.email' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 38 flags hashpspool stripe_width 0
pool 14 'china.rgw.users.keys' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 39 flags hashpspool stripe_width 0
pool 15 'china.rgw.users.swift' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 40 flags hashpspool stripe_width 0
pool 16 'china.rgw.users.uid' replicated size 2 min_size 1 crush_rule 0 object_hash rjenkins pg_num 256 pgp_num 256 last_change 41 flags hashpspool stripe_width 0
# 分别为 poolid ,poolname ,副本数 ,最小可读写副本数 ,crush_rule id……
max_osd 6
# osd总数
osd.0 up   in  weight 1 up_from 5 up_thru 49 down_at 0 last_clean_interval [0,0) 9.30.17.169:6801/2944694 9.30.17.169:6802/2944694 9.30.17.169:6803/2944694 9.30.17.169:6804/2944694 exists,up ed866aac-11ed-43ef-96bb-25017db61aa8
osd.1 up   in  weight 1 up_from 9 up_thru 49 down_at 0 last_clean_interval [0,0) 9.30.17.169:6805/2946023 9.30.17.169:6806/2946023 9.30.17.169:6807/2946023 9.30.17.169:6808/2946023 exists,up 2e350fba-25f6-4d37-9cbe-514258d72a2c
osd.2 up   in  weight 1 up_from 13 up_thru 49 down_at 0 last_clean_interval [0,0) 9.30.17.169:6809/2947243 9.30.17.169:6810/2947243 9.30.17.169:6811/2947243 9.30.17.169:6812/2947243 exists,up 3e028840-87b2-4122-a8a8-4b4f71a206ad
osd.3 up   in  weight 1 up_from 17 up_thru 49 down_at 0 last_clean_interval [0,0) 9.30.17.169:6813/2948340 9.30.17.169:6814/2948340 9.30.17.169:6815/2948340 9.30.17.169:6816/2948340 exists,up 33cb661f-e833-43b2-8702-e70cd89bcf8a
osd.4 up   in  weight 1 up_from 21 up_thru 49 down_at 0 last_clean_interval [0,0) 9.30.17.169:6817/2949495 9.30.17.169:6818/2949495 9.30.17.169:6819/2949495 9.30.17.169:6820/2949495 exists,up 6ea0232c-4665-4b7f-9cff-b6414257272e
osd.5 up   in  weight 1 up_from 25 up_thru 49 down_at 0 last_clean_interval [0,0) 9.30.17.169:6821/2950556 9.30.17.169:6822/2950556 9.30.17.169:6823/2950556 9.30.17.169:6824/2950556 exists,up d368256e-be35-44a5-aa32-6c431a2df983
# 后面四个端口分别是： public(用来监听来自Monitor和Client的连接)  ,cluster(用来监听来自OSD Peer的连接) ,back(OSD之间心跳) ,front(mon与osd心跳检测)
```