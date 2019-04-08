# 8. 对象存储常用操作
---


Tstack对象存储对外部署，有两种部署模型：

- 单独的OSS集群
    - 部署了三个 rgw 使用 haporxy+keepalived 实现高可用
    - 只用做对象存储使用
    - radosgw-admin提供用户管理功能

- Tstack自动化部署时部署的对象存储
    - 第一个mon节点作为对象存储网关
    - 该集群默认用作基础云平台的块设备后端
    - 对接上层之后上层RBAC提供认证，亦支持radosgw-admin操作

以下操作皆使用命令行工具

### 8.1 用户管理

#### 用户创建

    radosgw-admin user create --uid=test --display-name="test admin user" --access-key=test --secret=test --system

- `uid` 用户唯一标识符
- `display-name` 显示名称
- `access-key` S3 access key
- `secret` secret key

#### 获取用户信息

要获取有关用户的信息，您必须指定 uid

    radosgw-admin user info --uid=test

#### 用户启用与停用

创建用户时，默认情况下会启用该用户。但是，您可以暂停用户权限并在以后重新启用它们。要暂停用户，请指定 用户ID

    radosgw-admin user suspend --uid=test

要重新启用已暂停的用户，请指定用户ID

    radosgw-admin user enable --uid=test

#### 删除用户

删除用户时，将从系统中删除用户和子用户，需要指定 uid

    radosgw-admin user rm --uid=test

可选项包括：

- 清除数据：--purge-data清除与UID关联的所有数据。
- 清除keys：--purge-keys清除与UID关联的所有key。

#### 增加删除key

用户必须有key才能访问S3或者Swift接口，要使用S3，用户需要一个由 `access key` 和 `secret key` 组成的密钥对，要使用Swift，用户通常需要 `secret key` （密码），并将其与关联的用户ID一起使用。您可以创建密钥，并指定或生成`access key` 和/或 `secret key`。您也可以删除密钥。有以下选项可用:

- --key-type=<type> 指定密钥类型。选项有：s3，swift
- --access-key=<key> 手动指定 S3 access key.
- --secret-key=<key> 手动指定 S3 secret key 或者 Swift secret key.
- --gen-access-key 自动生成一个随机的 S3 access key.
- --gen-secret自动生成一个随机的 S3 secret key or a random Swift secret key.

1. 例如，为用户增加一个S3 密钥对:

    radosgw-admin key create --uid=test --access-key testkey  --secret-key testkey

输出如下：

```bash
{"user_id": "test",
"display_name": "test admin user ",
"email": "",
"suspended": 0,
"max_buckets": 1000,
"auid": 0,
"subusers": [],
"keys": [
    {
        "user": "test",
        "access_key": "XHOTAIS2EKC1JISECK3P",
        "secret_key": "wfaLNNMdsZuVZp4moM0cjwJlmrtG9BnE2lCh9It9"
    },
    # 创建用户时未指定自动生成的
    {
        "user": "test",
        "access_key": "testkey",
        "secret_key": "testkey"
    }
    # 新加的
...
```

2. 删除key

删除一个S3密钥对，需要指定用户id和 `access key`

    radosgw-admin key rm --uid=test --key-type=s3 --access-key=testkey

#### 添加/删除管理员能力(权限)

Ceph 用 “能力”（ capabilities, caps ）这个术语来描述给认证用户的授权，这样才能使用 Mon、 OSD 和 MDS 的功能。能力也用于限制对某一存储池内的数据或某个命名空间的访问。 Ceph 管理员用户可在创建或更新普通用户时赋予他相应的能力。

Ceph存储集群提供了一个管理API，使用户能够通过REST API执行管理功能。默认情况下，用户无权访问此API。要使用户能够执行管理功能，请为用户提供管理能力。

要向用户添加管理能力，请执行以下操作：

    radosgw-admin caps add --uid={uid} --caps={caps}

您可以向用户，存储桶，元数据和使用（利用率）添加读取，写入或所有功能。例如：

    --caps="[users|buckets|metadata|usage|zone]=[*|read|write|read, write]"

例如：

    radosgw-admin caps add --uid=test --caps="users=\*;buckets=\*"

要从用户删除管理功能，请执行以下操作：

    radosgw-admin caps rm --uid=test --caps={caps}

例如：

    radosgw-admin caps rm --uid=test --caps="users=\*;buckets=\*"

### 8.2 桶操作

#### 8.2.1 Windows操作系统

windows操作系统可使用S3Browser工具来操作，该工具与使用手册在部署包 `ansible-deploy\roles\ansible-ceph\ceph-rgw\files` 目录下

#### 8.2.2 类Uninx操作系统

我们想要在类 Unix 系统下适用 S3 服务，可以使用工具 s3cmd。安装方法：

##### 安装S3cmd

跑完脚本默认是装好的，如果手动装的则需要手动安装s3cmd，版本最低为 1.6.1

##### 配置s3cfg

```bash
[default]
access_key = tstack
secret_key = tstack
host_base = tstack-s3.oa.com
host_bucket = tstack-s3.oa.com

bucket_location = US
check_ssl_certificate = False
check_ssl_hostname = False
signature_v2 = True
use_https = False
```

##### s3cmd常用命令


- 列出使用这对key的所有桶

    s3cmd ls

- 列出test桶内对象

    s3cmd ls s3://BUCKET_NAME

- 创建桶

    s3cmd mb s3://BUCKET_NAME

- 上传一个对象

    s3cmd put FILE s3://BUCKET_NAME

- 下载一个对象

    s3cmd get s3://BUCKET_NAME/OBJECT

- 下载一个目录

    s3cmd sync 3://BUCKET_NAME/DIR

- 删除桶

    s3cmd del s3://BUCKET_NAME

- 删除对象

    s3cmd rm s3://BUCKET_NAME/OBJECT

更多详细信息查看 `s3cmd help`