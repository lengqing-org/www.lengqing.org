---
layout: post
title:  "Ubuntu Server 18.04 安装 OpenStack Stein"
date:   2019-07-18 06:18:00 +0800
categories: Linux
---

**本文参考：[OpenStack安装指南](https://docs.openstack.org/install-guide/index.html)**

**指南中提到指南不适用于生产系统安装，而是为了学习OpenStack而创建最小概念验证。**

**指南中要求的硬件：**

> 1个控制节点，拥有1-2CPU，8G RAM，100GB存储，2NIC
>
> 至少一个计算节点，每个节点拥有2-4+CPU，8+G RAM，100+GB存储，2NIC
>
> 可选1个块存储节点，拥有1-2CPU，4G RAM，100+GB存储，1NIC
>
> 可选多个对象存储节点，每个节点拥有1-2CPU，4+GB RAM，100+GB存储，1NIC
>
> 网络，提供商网络 / 自助服务网络，二选一，自助服务网络基于提供商网络，但更复杂。
>

**指南中提到的最低硬件要求：**

> 控制节点：1个处理器，4 GB内存和5 GB存储
>
> 计算节点：1个处理器，2 GB内存和10 GB存储
>

**指南中提到的密码引用[地址](https://docs.openstack.org/install-guide/environment-security.html)**

---
# 0 本文操作时的环境

**本文所有系统均为VMware Workstation 15 Pro中的Ubuntu Server 18.04.2虚拟机**

**本文仅选择控制节点和计算节点，存储使用本地存储，不使用存储节点**

**控制节点**：2核CPU，4G RAM，100GB存储，2网络接口（ens33：192.168.2.50，ens38：192.168.2.52）

**计算节点**：4核CPU，4G RAM，100GB存储，2网络接口（ens33：192.168.2.51，ens38：192.168.2.53）

**网络**：VMware虚拟网络


# 1 安装前配置

## 1.1 配置网络接口

### 1.1.1 控制节点

配置第一个接口作为管理接口：

IP地址：192.168.2.50

网络掩码：255.255.255.0（或/24）

默认网关：192.168.2.1

Ubuntu静态IP配置就不举例了，我这里直接DHCP服务器绑定了MAC地址和IP。

### 1.1.2 计算节点
差不多与控制节点完全一样

配置第一个接口作为管理接口：

IP地址：192.168.2.51

网络掩码：255.255.255.0（或/ 24）

默认网关：192.168.2.1

## 1.2 配置名称解析

### 1.2.1 控制节点
将节点的主机名设置为controller。

编辑/etc/hosts文件，添加以下内容

>#controller
>
>192.168.2.50      controller
>
>#compute1
>
>192.168.2.51      compute1
>
>#下面是块存储和对象存储的示例，有的话才需要
>#block1
>10.0.0.41       block1
>
>#object1
>10.0.0.51       object1
>
>#object2
>10.0.0.52       object2



### 1.2.2 计算节点

将节点的主机名设置为compute1。

编辑/etc/hosts文件，添加以下内容

>#controller
>
>192.168.2.50      controller
>
>#compute1
>
>192.168.2.51      compute1
>
>#下面是块存储和对象存储的示例，有的话才需要
>#block1
>10.0.0.41       block1
>
>#object1
>10.0.0.51       object1
>
>#object2
>10.0.0.52       object2


## 1.3 验证连接

都ping通了就算成功了

### 1.3.1 控制节点

```shell
ping -c 4 docs.openstack.org
ping -c 4 compute1
```

### 1.3.2 计算节点

```shell
ping -c 4 openstack.org
ping -c 4 controller
```

## 1.4 NTP网络时间协议
控制节点使用网络上的NTP服务器同步时间，其它节点使用控制节点同步时间

### 1.4.1 控制节点
安装chrony

```shell
apt-get install chrony
```

修改/etc/chrony/chrony.conf，
使用cn.ntp.org.cn，添加以下内容
>server cn.ntp.org.cn iburst

允许其它节点连接控制节点，添加以下内容
>allow 192.168.2.0/24

重启NTP服务
```shell
service chrony restart
```

### 1.4.2 计算节点
安装chrony

```shell
apt-get install chrony
```

修改/etc/chrony/chrony.conf，
使用cn.ntp.org.cn，添加以下内容

>server controller iburst

注释掉其它的服务器
>pool ntp.ubuntu.com        iburst maxsources 4
>ool 0.ubuntu.pool.ntp.org iburst maxsources 1
>ool 1.ubuntu.pool.ntp.org iburst maxsources 1
>ool 2.ubuntu.pool.ntp.org iburst maxsources 2

重启NTP服务
```shell
service chrony restart
```

## 1.5 验证NTP设置
[此链接上这个样子就成功了](https://docs.openstack.org/install-guide/environment-ntp-verify.html)

### 1.5.1 控制节点
```shell
chronyc sources
```


### 1.5.2 计算节点
```shell
chronyc sources
```

# 2 安装环境

## 2.1 为Ubuntu Cloud Archive启用存储库

### 2.1.1 这应该是所有节点都要做的
添加存储库
```shell
add-apt-repository cloud-archive:stein
```

## 2.2 安装OpenStack客户端
### 2.2.1 这应该是所有节点都要做的
更新，如果升级中包含新内核，需要重启
```
apt-get update && apt-get dist-upgrade
```

安装
```shell
apt-get install python3-openstackclient
```

## 2.3 数据库的安装与配置
### 2.3.1 数据库仅需要在控制节点安装

安装
```shell
apt-get install mariadb-server python-pymysql
```
创建并编辑/etc/mysql/mariadb.conf.d/99-openstack.cnf文件，并将bind-address 密钥设置为控制器节点的管理IP地址，以允许其他节点通过管理网络进行访问。设置其他键以启用有用选项和UTF-8字符集：

>[mysqld]
>
>bind-address = 192.168.2.50
>
>
>
>default-storage-engine = innodb
>
>innodb_file_per_table = on
>
>max_connections = 4096
>
>collation-server = utf8_general_ci
>
>character-set-server = utf8

重启数据库

```shell
service mysql restart
```

数据库初始化设置

```shell
mysql_secure_installation
```

## 2.4 消息队列

### 2.4.1 消息队列仅在控制节点运行

安装

```shell
apt-get install rabbitmq-server
```

添加openstack用户

```shell
rabbitmqctl add_user openstack 用户密码
```

允许用户进行配置，写入和读取访问 `openstack`

```shell
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

## 2.5 Memcached（身份验证）

### 2.5.1 Memcached仅在控制节点运行

安装

```shell
apt-get install memcached python-memcache
```

编辑`/etc/memcached.conf`文件并配置服务以使用控制器节点的管理IP地址

```shell
vim /etc/memcached.conf
```

改变现有行
> -l 192.168.2.50

重启memcached服务

```shell
service memcached restart
```

## 2.6 Etcd（分布式键值对数据存储系统）

### 2.6.1 Etcd仅在控制节点运行

安装

```shell
apt-get install etcd
```

编辑`/etc/default/etcd`文件并设置`ETCD_INITIAL_CLUSTER`， `ETCD_INITIAL_ADVERTISE_PEER_URLS`，`ETCD_ADVERTISE_CLIENT_URLS`， `ETCD_LISTEN_CLIENT_URLS`控制器节点

>ETCD_NAME="controller"
>
>ETCD_DATA_DIR="/var/lib/etcd"
>
>ETCD_INITIAL_CLUSTER_STATE="new"
>
>ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
>
>ETCD_INITIAL_CLUSTER="controller=http://192.168.2.50:2380"
>
>ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.2.50:2380"
>
>ETCD_ADVERTISE_CLIENT_URLS="http://192.168.2.50:2379"
>
>ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
>
>ETCD_LISTEN_CLIENT_URLS="http://192.168.2.50:2379"

启动启用服务

```shell
systemctl enable etcd
systemctl start etcd
```

# 3 安装OpenStack服务

**Stein的最小部署**

至少需要安装以下服务。按以下指定的顺序安装服务：
身份服务 – [keystone installation for Stein](https://docs.openstack.org/keystone/stein/install/)
映像服务 – [glance installation for Stein](https://docs.openstack.org/glance/stein/install/)
安置服务 – [placement installation for Stein](https://docs.openstack.org/placement/stein/install/)
计算服务 – [nova installation for Stein](https://docs.openstack.org/nova/stein/install/)
网络服务 – [neutron installation for Stein](https://docs.openstack.org/neutron/stein/install/)

指南建议在安装最小部署服务后也安装以下组件：
仪表盘 – [horizon installation for Stein](https://docs.openstack.org/horizon/stein/install/)
块存储服务 – [cinder installation for Stein](https://docs.openstack.org/cinder/stein/install/)

**Stein所有的服务**

<https://docs.openstack.org/stein/install/>

## 3.1 身份服务

### 3.1.1 身份服务仅在控制节点上运行

#### 3.1.1.1 先决条件（创建数据库）

1. 使用数据库访问客户端以`root`用户身份连接到数据库服务器：

   ```shell
   mysql
   ```

2. 创建`keystone`数据库：

   ```shell
   CREATE DATABASE keystone;
   ```

3. 授予对`keystone`数据库的适当访问权限：

   ```shell
   GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
   IDENTIFIED BY 'KEYSTONE_DBPASS';
   GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
   IDENTIFIED BY 'KEYSTONE_DBPASS';
   ```
替换`KEYSTONE_DBPASS`为合适的密码。

4. 退出数据库

   

#### 3.1.1.2 安装和配置组件

1. 安装软件包，如果包管理提示依赖问题，就把`keyston`单独安装
   ```shell
   apt-get install keystone apache2 libapache2-mod-wsgi
   ```

2. 编辑`/etc/keystone/keystone.conf`文件并完成以下操作

   在该[database]`部分中，配置数据库访问：
   >[database]
   >connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
   
   替换`KEYSTONE_DBPASS`为您为数据库选择的密码。

   在该`[token]`部分中，配置Fernet令牌提供程序：

   >[token]
   >provider = fernet

3. 填充Identity服务数据库：

   ```shell
   su -s /bin/sh -c "keystone-manage db_sync" keystone
   ```
4.初始化Fernet密钥存储库：
   
   ```shell
   keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
   keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
   ```

   5.引导身份服务：
   
   ```shell
   keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
   --bootstrap-admin-url http://controller:5000/v3/ \
   --bootstrap-internal-url http://controller:5000/v3/ \
   --bootstrap-public-url http://controller:5000/v3/ \
   --bootstrap-region-id RegionOne
   ```
   替换ADMIN_PASS为管理用户的合适密码。

#### 3.1.1.3 配置Apache HTTP服务器

编辑`/etc/apache2/apache2.conf`文件并配置`ServerName`引用控制器节点的 选项：

> ServerName controller

**SSL**
这里没说

#### 3.1.1.4 完成安装
重启Apache服务
```shell
service apache2 restart
```
配置管理帐户
```shell
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```
替换ADMIN_PASS为3.1.1.2中keystone-manage bootstrap命令中使用的密码。

#### 3.1.1.5 创建域，项目，用户和角色

创建新域

```shell
openstack domain create --description "An Example Domain" example
#输出
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | An Example Domain                |
| enabled     | True                             |
| id          | 54b9ba6ef9814e17bb3de96ace8870bd |
| name        | example                          |
| tags        | []                               |
+-------------+----------------------------------+
```

创建服务项目

```shell
openstack project create --domain default \
  --description "Service Project" service
#输出
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | b0de8cb0e1134f53a8286152f68d992c |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
```

创建`myproject`项目

```shell
openstack project create --domain default \
  --description "Demo Project" myproject
#输出
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 497005cae90144ab83a557650e1cb8b1 |
| is_domain   | False                            |
| name        | myproject                        |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
```

创建`myuser` 用户

```shell
openstack user create --domain default \
  --password-prompt myuser
#输出
User Password:
Repeat User Password:
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 375d48d3283e42d5918a5b51b950cfe1 |
| name                | myuser                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

创建`myrole`角色

 ```shell
  openstack role create myrole
  #输出
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | None                             |
| domain_id   | None                             |
| id          | 9d84709f4538497fa80d2482846a78f5 |
| name        | myrole                           |
+-------------+----------------------------------+
 ```

将`myrole`角色添加到`myproject`项目和`myuser`用户：

```shell
openstack role add --project myproject --user myuser myrole
```

#### 3.1.1.6 验证操作

1. 取消设置临时 变量`OS_AUTH_URL`和`OS_PASSWORD`环境变量：

   ```shell
   unset OS_AUTH_URL OS_PASSWORD
   ```

2. 作为`admin`用户，请求身份验证令牌：

   ```shell
   openstack --os-auth-url http://controller:5000/v3 \
     --os-project-domain-name Default --os-user-domain-name Default \
     --os-project-name admin --os-username admin token issue
   #输出
   Password:
   +------------+-----------------------------------------------------------------+
   | Field      | Value                                                           |
   +------------+-----------------------------------------------------------------+
   | expires    | 2019-07-18T03:19:00+0000                                        |
   | id         | gAAAAABdL9cUXCY-Lk7j9qkxXHOIihLju5ruoWM4vzdcJEPSCBakUB-V9u0z4lU |
   |            | nEZGimv6_wR6zdPndtV7BzyMV52vhEVBkyXLaa7AMuJ545yyME1lDGyHESGcmVi |
   |            |4kNan6QYsNL3B_AidXqxejFohgm81oDFJ8X4vIlucpLnk4ogA-C1amJbU        |
   | project_id |  ddab7b9e10d14c59a64143a8f04ac045                               |
   | user_id    | 835091030b714480b7ec1bbe9c3a9f15                                |
   +------------+-----------------------------------------------------------------+
   ```

   注意，此命令使用`admin`用户的密码。

3. 正如`myuser`用户在上一个中创建的那样，请求一个身份验证令牌：

   ```shell
   openstack --os-auth-url http://controller:5000/v3 \
     --os-project-domain-name Default --os-user-domain-name Default \
     --os-project-name myproject --os-username myuser token issue
   #输出
   Password:
   +------------+-----------------------------------------------------------------+
   | Field      | Value                                                           |
   +------------+-----------------------------------------------------------------+
   | expires    |  2019-07-18T03:22:46+0000                                       |
   | id         | gAAAAABdL9f2pKqTes1RBP8tiqtAWbd2_GHXvFVLCJ4Hc1XSDQRqpTJxt2XorNc |
   |            | 2HeaKCo8ZzlUsA_CFCYwsX02vtlvVFoAFs5Q6xFvI9tageBtJpxewSeL8Qq5kmC |
   |            | kaUIMdGgNlMGNf0GSsq9qOfZuyXuw0NEofb37KV8UlrGtjYGfebjjUORI       |
   | project_id | 497005cae90144ab83a557650e1cb8b1                                |
   | user_id    | 375d48d3283e42d5918a5b51b950cfe1                                |
   +------------+-----------------------------------------------------------------+
   ```

#### 3.1.1.7 创建OpenStack客户端环境脚本

创建客户端环境的脚本`admin`和`demo` 项目和用户。本指南的后续部分引用这些脚本来加载客户端操作的适当凭据。
注意，客户端环境脚本的路径不受限制。为方便起见，您可以将脚本放在任何位置，但请确保它们可以访问并位于适合部署的安全位置，因为它们包含敏感凭据。

1. 创建和编辑`admin-openrc`文件并添加以下内容：

   注意，OpenStack客户端还支持使用`clouds.yaml`文件。有关更多信息，请参阅[os-client-config](http://docs.openstack.org/os-client-config/latest)。

   ```
   export OS_PROJECT_DOMAIN_NAME=Default
   export OS_USER_DOMAIN_NAME=Default
   export OS_PROJECT_NAME=admin
   export OS_USERNAME=admin
   export OS_PASSWORD=ADMIN_PASS
   export OS_AUTH_URL=http://controller:5000/v3
   export OS_IDENTITY_API_VERSION=3
   export OS_IMAGE_API_VERSION=2
   ```

   替换`ADMIN_PASS`为您`admin`在Identity服务中为用户选择的密码。

3. 创建和编辑`demo-openrc`文件并添加以下内容：

   ```
   export OS_PROJECT_DOMAIN_NAME=Default
   export OS_USER_DOMAIN_NAME=Default
   export OS_PROJECT_NAME=myproject
   export OS_USERNAME=myuser
   export OS_PASSWORD=MYUSER_PASS
   export OS_AUTH_URL=http://controller:5000/v3
   export OS_IDENTITY_API_VERSION=3
   export OS_IMAGE_API_VERSION=2
   ```

   替换`DEMO_PASS`为您`demo`在Identity服务中为用户选择的密码。

#### 3.1.1.8 使用脚本

要将客户端作为特定项目和用户运行，只需在运行它们之前加载关联的客户端环境脚本即可。例如：

1. 加载`admin-openrc`文件以使用Identity服务的位置以及`admin`项目和用户凭据填充环境变量：

   ```shell
   source admin-openrc
   ```

2. 请求身份验证令牌：

   ```shell
   openstack token issue
   
   +------------+-----------------------------------------------------------------+
   | Field      | Value                                                           |
   +------------+-----------------------------------------------------------------+
   | expires    | 2019-07-18T04:27:59+0000                                        |
   | id         | gAAAAABdL-c_Q5A7MjnZek0qb9o8jdvSw2ljIKqyZp14dXwjRyNGQpJSiQSAvyZ |
   |            | E-WhjVPc9e1jTtjFKu3UjW_FAhs9ayNRHiNBLRAWdp_RY7Hsb5GHzOF8Q88r8qS |
   |            | R_k4-9wkAKDP7wwNWP5WZXOBNbl-GRfWqkYYdEEhrpbMywjUJbw-17vsk       |
   | project_id | ddab7b9e10d14c59a64143a8f04ac045                                |
   | user_id    |  835091030b714480b7ec1bbe9c3a9f15                               |
   +------------+-----------------------------------------------------------------+
   ```

## 3.2 映像服务

### 3.2.1 映像服务仅在控制节点运行

#### 3.2.1.1 先决条件

1. 要创建数据库，请完成以下步骤：

   使用数据库访问客户端以`root`用户身份连接到数据库服务器：

   ```shell
   mysql
   ```

   创建`glance`数据库：

   ```shell
   CREATE DATABASE glance;
   ```

   授予对`glance`数据库的适当访问权限：

    ```shell
    GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
     IDENTIFIED BY 'GLANCE_DBPASS';
    GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
     IDENTIFIED BY 'GLANCE_DBPASS';
    ```

   替换`GLANCE_DBPASS`为合适的密码。

   退出数据库访问客户端。

2. 来源`admin`凭据来访问仅管理员CLI命令：

   ```
   source admin-openrc
   ```

3. 要创建服务凭据，请完成以下步骤：

   创建`glance`用户：

   ```shell
   openstack user create --domain default --password-prompt glance
   #输出
   User Password:
   Repeat User Password:
   +---------------------+----------------------------------+
   | Field               | Value                            |
   +---------------------+----------------------------------+
   | domain_id           | default                          |
   | enabled             | True                             |
   | id                  | 5f9f2a8cb21f4116882df5d8f6ddc529 |
   | name                | glance                           |
   | options             | {}                               |
   | password_expires_at | None                             |
   +---------------------+----------------------------------+
   ```

   将`admin`角色添加到`glance`用户和 `service`项目：

   ```shell
   openstack role add --project service --user glance admin
   ```

   创建`glance`服务实体：

   ```shell
   openstack service create --name glance \
    --description "OpenStack Image" image
   #输出
   +-------------+----------------------------------+
   | Field       | Value                            |
   +-------------+----------------------------------+
   | description | OpenStack Image                  |
   | enabled     | True                             |
   | id          | a76799e303334fb7baec59a104547e86 |
   | name        | glance                           |
   | type        | image                            |
   +-------------+----------------------------------+
   ```
   
4. 创建Image服务API端点：
   
   ```shell
   openstack endpoint create --region RegionOne \
     image public http://controller:9292
   #输出
   +--------------+----------------------------------+
   | Field        | Value                            |
   +--------------+----------------------------------+
   | enabled      | True                             |
   | id           | b96227302a474289af3d47184e2682c1 |
   | interface    | public                           |
   | region       | RegionOne                        |
   | region_id    | RegionOne                        |
   | service_id   | a76799e303334fb7baec59a104547e86 |
   | service_name | glance                           |
   | service_type | image                            |
   | url          | http://controller:9292           |
   +--------------+----------------------------------+
   
   openstack endpoint create --region RegionOne \
     image internal http://controller:9292
   #输出
   +--------------+----------------------------------+
   | Field        | Value                            |
   +--------------+----------------------------------+
   | enabled      | True                             |
   | id           | 2f5de2bd8fb84143a3370d7c0c6daf4e |
   | interface    | internal                         |
   | region       | RegionOne                        |
   | region_id    | RegionOne                        |
   | service_id   | a76799e303334fb7baec59a104547e86 |
   | service_name | glance                           |
   | service_type | image                            |
   | url          | http://controller:9292           |
   +--------------+----------------------------------+
   
   openstack endpoint create --region RegionOne \
     image admin http://controller:9292
   #输出
   +--------------+----------------------------------+
   | Field        | Value                            |
   +--------------+----------------------------------+
   | enabled      | True                             |
   | id           | 8b88c34bbaf34bda9856c6de2d9b31d6 |
   | interface    | admin                            |
   | region       | RegionOne                        |
   | region_id    | RegionOne                        |
   | service_id   | a76799e303334fb7baec59a104547e86 |
   | service_name | glance                           |
   | service_type | image                            |
   | url          | http://controller:9292           |
   +--------------+----------------------------------+
   ```

#### 3.2.1.2 安装和配置组件

1. 安装包：

   ```shell
   apt-get install glance
   ```

2. 编辑`/etc/glance/glance-api.conf`文件并完成以下操作：

   在该`[database]`部分中，配置数据库访问：

   >[database]
   >
   >connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
   >

   替换`GLANCE_DBPASS`为您为Image服务数据库选择的密码。

   在`[keystone_authtoken]`和`[paste_deploy]`部分中，配置身份服务访问：
   >
   >[keystone_authtoken]
   >
   >www_authenticate_uri = http://controller:5000
   >auth_url = http://controller:5000
   >memcached_servers = controller:11211
   >auth_type = password
   >project_domain_name = Default
   >user_domain_name = Default
   >project_name = service
   >username = glance
   >password = GLANCE_PASS
   >
   >[paste_deploy]
   >
   >flavor = keystone
   >

   替换`GLANCE_PASS`为您`glance`在Identity服务中为用户选择的密码 。
   注意，注释掉或删除该`[keystone_authtoken]`部分中的任何其他选项 。

   在该`[glance_store]`部分中，配置本地文件系统存储和映像文件的位置：

   >[glance_store]
   >
   >stores = file,http
   >default_store = file
   >filesystem_store_datadir = /var/lib/glance/images/
   
3. 编辑`/etc/glance/glance-registry.conf`文件并完成以下操作：
   在该`[database]`部分中，配置数据库访问：

   >[database]
   >
   >connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

   替换`GLANCE_DBPASS`为您为Image服务数据库选择的密码。

   在`[keystone_authtoken]`和`[paste_deploy]`部分中，配置身份服务访问：
   
   >[keystone_authtoken]
   >
   >www_authenticate_uri = http://controller:5000
   >auth_url = http://controller:5000
   >memcached_servers = controller:11211
   >auth_type = password
   >project_domain_name = Default
   >user_domain_name = Default
   >project_name = service
   >username = glance
   >password = GLANCE_PASS
   >
   >[paste_deploy]
   >flavor = keystone

   替换`GLANCE_PASS`为您`glance`在Identity服务中为用户选择的密码 。
   注意,注释掉或删除该`[keystone_authtoken]`部分中的任何其他选项 。

4. 填充Image服务数据库：

   ```shell
   su -s /bin/sh -c "glance-manage db_sync" glance
   ```
   注意,忽略此输出中的任何弃用消息。
   
#### 3.2.1.3 完成安装

重启Image服务

```shell
service glance-registry restart #此行报错没有此服务，下面的验证操作也没问题，怀疑没有需要运行此行
service glance-api restart
```

#### 3.2.1.4 验证操作

1. 来源`admin`凭据来访问仅管理员CLI命令：

   ```shell
   source admin-openrc
   ```

2. 下载源图像：

   ```shell
   wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
   ```

3. 使用[QCOW2](https://docs.openstack.org/glance/stein/glossary.html#term-qemu-copy-on-write-2-qcow2)磁盘格式，[裸](https://docs.openstack.org/glance/stein/glossary.html#term-bare) 容器格式和公共可见性将映像上载到映像服务 ，以便所有项目都可以访问它：

   ```
   openstack image create "cirros" \
     --file cirros-0.4.0-x86_64-disk.img \
     --disk-format qcow2 --container-format bare \
     --public
   #输出
   +------------------+------------------------------------------------------+
   | Field            | Value                                                |
   +------------------+------------------------------------------------------+
   | checksum         | 443b7623e27ecf03dc9e01ee93f67afe                     |
   | container_format | bare                                                 |
   | created_at       | 2019-07-18T03:52:36Z                                 |
   | disk_format      | qcow2                                                |
   | file             | /v2/images/643f8b18-6272-4333-9983-0fb650ebf0c9/file |
   | id               | 643f8b18-6272-4333-9983-0fb650ebf0c9                 |
   | min_disk         | 0                                                    |
   | min_ram          | 0                                                    |
   | name             | cirros                                               |
   | owner            | ddab7b9e10d14c59a64143a8f04ac045                     |
   | protected        | os_hash_algo='sha512', os_hash_value='6513f21e44aa3d |
   |                  | a349f248188a44bc304a3653a04122d8fb4535423c8e1d14cd6a |
   |                  | 153f735bb0982e2161b5b5186106570c17a9e58b64dd39390617 |
   |                  | cd5a350f78', os_hidden='False'                       |
   | schema           | /v2/schemas/image                                    |
   | size             | 12716032                                             |
   | status           | active                                               |
   | tags             |                                                      |
   | updated_at       | 2019-07-18T03:52:36Z                                 |
   | virtual_size     | None                                                 |
   | visibility       | public                                               |
   +------------------+------------------------------------------------------+
   ```

   有关信息**OpenStack的图像创建**参数，请参阅 [Create or update an image (glance)](https://docs.openstack.org/user-guide/common/cli-manage-images.html#create-or-update-an-image-glance)

   有关磁盘和容器的图像格式的信息，请参阅 [Disk and container formats for images](https://docs.openstack.org/image-guide/image-formats.html)

   注意，OpenStack动态生成ID，因此您将在示例命令输出中看到不同的值。

4. 确认上传图像并验证属性：

   ```shell
   openstack image list
   #输出
   +--------------------------------------+--------+--------+
   | ID                                   | Name   | Status |
   +--------------------------------------+--------+--------+
   | 643f8b18-6272-4333-9983-0fb650ebf0c9 | cirros | active |
   +--------------------------------------+--------+--------+
   ```

## 3.3 安置服务

### 3.3.1 安置服务仅运行在控制节点

#### 3.3.1.1 先决条件

在安装和配置放置服务之前，必须创建数据库，服务凭据和API端点。
   ##### 创建数据库
1. 要创建数据库，请完成以下步骤：
   使用数据库访问客户端以`root`用户身份连接到数据库服务器：
   ```shell
   mysql
   ```
   
   创建`placement`数据库：
   ```shell
   CREATE DATABASE placement;
   ```

   授予对数据库的适当访问权限：
   ```shell
   GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' \
     IDENTIFIED BY 'PLACEMENT_DBPASS';
   GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' \
     IDENTIFIED BY 'PLACEMENT_DBPASS';
   ```
   替换`PLACEMENT_DBPASS`为合适的密码。
   退出数据库访问客户端。

  ##### 配置用户和端点

1. 来源`admin`凭据来访问仅管理员CLI命令：
   ```shell
   source admin-openrc
   ```

2. 使用您选择的创建Placement服务用户`PLACEMENT_PASS`：

   ```shell
   openstack user create --domain default --password-prompt placement
   #输出
   User Password:
   Repeat User Password:
   +---------------------+----------------------------------+
   | Field               | Value                            |
   +---------------------+----------------------------------+
   | domain_id           | default                          |
   | enabled             | True                             |
   | id                  | 365658358d934c1e91f204ca14bcaee1 |
   | name                | placement                        |
   | options             | {}                               |
   | password_expires_at | None                             |
   +---------------------+----------------------------------+
   ```

3. 使用admin角色将Placement用户添加到服务项目：

   ```shell
   openstack role add --project service --user placement admin
   ```

4. 在服务目录中创建Placement API条目：

   ```shell
   openstack service create --name placement \
     --description "Placement API" placement
   #输出
   +-------------+----------------------------------+
   | Field       | Value                            |
   +-------------+----------------------------------+
   | description | Placement API                    |
   | enabled     | True                             |
   | id          | d6988c6dc2bd4a03a50970585c2e342f |
   | name        | placement                        |
   | type        | placement                        |
   +-------------+----------------------------------+
   ```

5. 创建Placement API服务端点：
   注意，根据您的环境，端点的URL将根据端口（可能是8780而不是8778，或根本没有端口）和主机名而有所不同。您有责任确定正确的URL。

   ```shell
   openstack endpoint create --region RegionOne \
     placement public http://controller:8778
   #输出
   +--------------+----------------------------------+
   | Field        | Value                            |
   +--------------+----------------------------------+
   | enabled      | True                             |
   | id           | cba1769b76cc4abab3f19b61ad3c7513 |
   | interface    | public                           |
   | region       | RegionOne                        |
   | region_id    | RegionOne                        |
   | service_id   | d6988c6dc2bd4a03a50970585c2e342f |
   | service_name | placement                        |
   | service_type | placement                        |
   | url          | http://controller:8778           |
   +--------------+----------------------------------+
     
   openstack endpoint create --region RegionOne \
     placement internal http://controller:8778
   #输出
   +--------------+----------------------------------+
   | Field        | Value                            |
   +--------------+----------------------------------+
   | enabled      | True                             |
   | id           | bd6c0ea2b16343c7a595dcd42a3b6153 |
   | interface    | internal                         |
   | region       | RegionOne                        |
   | region_id    | RegionOne                        |
   | service_id   | d6988c6dc2bd4a03a50970585c2e342f |
   | service_name | placement                        |
   | service_type | placement                        |
   | url          | http://controller:8778           |
   +--------------+----------------------------------+
     
   openstack endpoint create --region RegionOne \
     placement admin http://controller:8778
   #输出
   +--------------+----------------------------------+
   | Field        | Value                            |
   +--------------+----------------------------------+
   | enabled      | True                             |
   | id           | f37fe946cc0b45d280893d4a47e1eacc |
   | interface    | admin                            |
   | region       | RegionOne                        |
   | region_id    | RegionOne                        |
   | service_id   | d6988c6dc2bd4a03a50970585c2e342f |
   | service_name | placement                        |
   | service_type | placement                        |
   | url          | http://controller:8778           |
   +--------------+----------------------------------+
   ```

#### 3.3.1.2 安装和配置组件
1. 安装包：
   ```shell
   apt-get install placement-api
   ```

2. 编辑`/etc/placement/placement.conf`文件并完成以下操作：

   在该`[placement_database]`部分中，配置数据库访问：

   >[placement_database]
   >
   >connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement
   >

   替换`PLACEMENT_DBPASS`为您为放置数据库选择的密码。

   在`[api]`和`[keystone_authtoken]`部分中，配置身份服务访问：
   >[api]
   >
   >auth_strategy = keystone
   >
   >[keystone_authtoken]
   >
   >auth_url = http://controller:5000/v3
   >memcached_servers = controller:11211
   >auth_type = password
   >project_domain_name = default
   >user_domain_name = default
   >project_name = service
   >username = placement
   >password = PLACEMENT_PASS
   

替换`PLACEMENT_PASS`为您`placement`在Identity服务中为用户选择的密码 。
   注意，注释掉或删除该`[keystone_authtoken]` 部分中的任何其他选项。

3. 填充`placement`数据库：
   ```
   su -s /bin/sh -c "placement-manage db sync" placement
   ```
   注意，忽略此输出中的任何弃用消息。

#### 3.3.1.3 完成安装
重新加载Web服务器
```shell
service apache2 restart
```

#### 3.3.1.4 验证安装

1. 来源`admin`凭据来访问仅管理员CLI命令：

   ```shell
   source admin-openrc
   ```

2. 执行状态检查以确保一切正常：

   ```shell
   placement-status upgrade check #这里需要sudo，不然可能出不来正确结果
   #输出
   +----------------------------------+
   | Upgrade Check Results            |
   +----------------------------------+
   | Check: Missing Root Provider IDs |
   | Result: Success                  |
   | Details: None                    |
   +----------------------------------+
   | Check: Incomplete Consumers      |
   | Result: Success                  |
   | Details: None                    |
   +----------------------------------+
   ```

   该命令的输出因发布而异。有关详细信息，请参阅[placement-status upgrade check](https://docs.openstack.org/placement/stein/cli/placement-status.html#placement-status-checks) 

3. 针对展示位置API运行一些命令：
   安装[osc-placement](https://docs.openstack.org/osc-placement/latest/)插件：
   注意，此示例使用[PyPI](https://pypi.org/)和[pip](https://pypi.org/project/pip/)，但如果您使用的是分发包，则可以从其存储库中安装该包。

   ```shell
   apt-get install python-pip #ubuntu没这个插件也就算了，pip还得自己装
   pip install osc-placement
   ```
   

   列出可用的资源类和特征：

   ```shell
   openstack --os-placement-api-version 1.2 resource class list --sort-column name
   #输出
   openstack: '--os-placement-api-version 1.2 resource class list --sort-column name' is not an openstack command. See 'openstack --help'.
Did you mean one of these?
    orchestration build info
    orchestration resource type list
    orchestration resource type show
    orchestration service list
    orchestration template function list
    orchestration template validate
    orchestration template version list
   #而不是下面的输出
     +----------------------------+
     | name                       |
     +----------------------------+
     | DISK_GB                    |
     | IPV4_ADDRESS               |
     | ...                        |
     
    openstack --os-placement-api-version 1.6 trait list --sort-column name
   #输出
   openstack: '--os-placement-api-version 1.6 trait list --sort-column name' is not an openstack command. See 'openstack --help'.
   Did you mean one of these?
     application credential create
     application credential delete
     application credential list
     application credential show
     configuration show
   #而不是下面的输出
     +---------------------------------------+
     | name                                  |
     +---------------------------------------+
     | COMPUTE_DEVICE_TAGGING                |
     | COMPUTE_NET_ATTACH_INTERFACE          |
     | ...                                   |
   ```

## 3.4 计算服务

### 3.4.1 安装和配置控制节点

#### 3.4.1.1 先决条件

1. 要创建数据库，请完成以下步骤：

   使用数据库访问客户端以`root`用户身份连接到数据库服务器：
   ```shell
   mysql
   ```

   创建`nova_api`，`nova`和`nova_cell0`数据库：
   ```shell
   CREATE DATABASE nova_api;
   CREATE DATABASE nova;
   CREATE DATABASE nova_cell0;
   ```

   授予对数据库的适当访问权限：

   ```shell
   GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
       IDENTIFIED BY 'NOVA_DBPASS';
   GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
       IDENTIFIED BY 'NOVA_DBPASS';
     
   GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
       IDENTIFIED BY 'NOVA_DBPASS';
   GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
       IDENTIFIED BY 'NOVA_DBPASS';
     
   GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' \
       IDENTIFIED BY 'NOVA_DBPASS';
   GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \
       IDENTIFIED BY 'NOVA_DBPASS';
   ```

     替换`NOVA_DBPASS`为合适的密码。

   退出数据库访问客户端。

2. 来源`admin`凭据来访问仅管理员CLI命令：

   ```shell
   source admin-openrc
   ```

3. 创建Compute服务凭据：

   创建`nova`用户：

   ```shell
   openstack user create --domain default --password-prompt nova
   #输出
   User Password:
   Repeat User Password:
   +---------------------+----------------------------------+
   | Field               | Value                            |
   +---------------------+----------------------------------+
   | domain_id           | default                          |
   | enabled             | True                             |
   | id                  | 83842db153d34caebf79ac9e6a38196e |
   | name                | nova                             |
   | options             | {}                               |
   | password_expires_at | None                             |
   +---------------------+----------------------------------+
   ```

   将`admin`角色添加到`nova`用户：

   ```shell
   openstack role add --project service --user nova admin
   ```

   创建`nova`服务实体：
   ```shell
   openstack service create --name nova \
     --description "OpenStack Compute" compute
   #输出
   +-------------+----------------------------------+
   | Field       | Value                            |
   +-------------+----------------------------------+
   | description | OpenStack Compute                |
   | enabled     | True                             |
   | id          | 6a65433a36d54660a41a39d1aa37fca9 |
   | name        | nova                             |
   | type        | compute                          |
   +-------------+----------------------------------+
   ```

4. 创建Compute API服务端点：

   ```shell
   openstack endpoint create --region RegionOne \
     compute public http://controller:8774/v2.1
   #输出
   +--------------+----------------------------------+
   | Field        | Value                            |
   +--------------+----------------------------------+
   | enabled      | True                             |
   | id           | fcdb5eae3e144778b72e4810372a94f0 |
   | interface    | public                           |
   | region       | RegionOne                        |
   | region_id    | RegionOne                        |
   | service_id   | 6a65433a36d54660a41a39d1aa37fca9 |
   | service_name | nova                             |
   | service_type | compute                          |
   | url          | http://controller:8774/v2.1      |
   +--------------+----------------------------------+
   
   openstack endpoint create --region RegionOne \
     compute internal http://controller:8774/v2.1
   #输出
   +--------------+----------------------------------+
   | Field        | Value                            |
   +--------------+----------------------------------+
   | enabled      | True                             |
   | id           | a5721cd78b2f4c7a8b8f52f50d5213f3 |
   | interface    | internal                         |
   | region       | RegionOne                        |
   | region_id    | RegionOne                        |
   | service_id   | 6a65433a36d54660a41a39d1aa37fca9 |
   | service_name | nova                             |
   | service_type | compute                          |
   | url          | http://controller:8774/v2.1      |
   +--------------+----------------------------------+
   
   openstack endpoint create --region RegionOne \
     compute admin http://controller:8774/v2.1
   #输出
   +--------------+----------------------------------+
   | Field        | Value                            |
   +--------------+----------------------------------+
   | enabled      | True                             |
   | id           | f08c35a57bb7492f875eed0be5759bbf |
   | interface    | admin                            |
   | region       | RegionOne                        |
   | region_id    | RegionOne                        |
   | service_id   | 6a65433a36d54660a41a39d1aa37fca9 |
   | service_name | nova                             |
   | service_type | compute                          |
   | url          | http://controller:8774/v2.1      |
   +--------------+----------------------------------+
   ```
   

#### 3.4.1.2 安装和配置组件

1. 安装包：
   
   ```shell
   apt-get install nova-api nova-conductor \
     nova-novncproxy nova-scheduler
   ```

2. 编辑`/etc/nova/nova.conf`文件并完成以下操作：
   在`[api_database]`和`[database]`部分中，配置数据库访问：
   
   >[api_database]
   >
   >connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api
   >
   >[database]
   >
   >connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova
   >
   
   替换`NOVA_DBPASS`为您为Compute数据库选择的密码。
   
   在该`[DEFAULT]`部分中，配置`RabbitMQ`消息队列访问：
   >[DEFAULT]
   >
   >transport_url = rabbit://openstack:RABBIT_PASS@controller
   >

   替换`RABBIT_PASS`为您为`openstack` 帐户选择的密码`RabbitMQ`。
   
   在`[api]`和`[keystone_authtoken]`部分中，配置身份服务访问：
   >[api]
   >
   >auth_strategy = keystone
   >
   >[keystone_authtoken]
   >
   >auth_url = http://controller:5000/v3
   >memcached_servers = controller:11211
   >auth_type = password
   >project_domain_name = Default
   >user_domain_name = Default
   >project_name = service
   >username = nova
   >password = NOVA_PASS

   替换`NOVA_PASS`为您`nova`在Identity服务中为用户选择的密码。
   注意，注释掉或删除该`[keystone_authtoken]` 部分中的任何其他选项。

   在该`[DEFAULT]`部分中，配置`my_ip`选项以使用控制器节点的管理接口IP地址：
   >[DEFAULT]
   >
   >my_ip = 192.168.2.50
   >

   在该`[DEFAULT]`部分中，启用对网络服务的支持：
   >[DEFAULT]
   >
   >use_neutron = true
   >firewall_driver = nova.virt.firewall.NoopFirewallDriver

   注意，默认情况下，Compute使用内部防火墙驱动程序。由于Networking服务包含防火墙驱动程序，因此必须使用`nova.virt.firewall.NoopFirewallDriver`防火墙驱动程序禁用Compute防火墙驱动 程序。

   配置**/etc/nova/nova.conf**的`[neutron]`部分。 有关更多信息，请参阅[网络服务安装指南](https://docs.openstack.org/neutron/stein/install/controller-install-ubuntu.html#configure-the-compute-service-to-use-the-networking-service)。

   在该`[vnc]`部分中，配置VNC代理以使用控制器节点的管理接口IP地址：
   >[vnc]
   >enabled = true
   >
   >server_listen = $my_ip
   >server_proxyclient_address = $my_ip

   在该`[glance]`部分中，配置Image服务API的位置：
   >[glance]
   >
   >api_servers = http://controller:9292
   >

   在该`[oslo_concurrency]`部分中，配置锁定路径：
   >[oslo_concurrency]
   >
   >lock_path = /var/lib/nova/tmp
   >

   由于包装错误，请从`[DEFAULT]`部分中删除`log_dir`选项 。
   在该`[placement]`部分中，配置对Placement服务的访问权限：
   
   >[placement]
   >
   >region_name = RegionOne
   >project_domain_name = Default
   >project_name = service
   >auth_type = password
   >user_domain_name = Default
   >auth_url = http://controller:5000/v3
   >username = placement
>password = PLACEMENT_PASS

替换`PLACEMENT_PASS`为您`placement`在安装[Placement](https://docs.openstack.org/placement/stein/install/)时为服务用户 选择的密码 。注释掉或删除该`[placement]`部分中的任何其他选项。

3. 填充`nova-api`数据库：

   ```shell
   su -s /bin/sh -c "nova-manage api_db sync" nova
   ```

   注意，忽略此输出中的任何弃用消息。

4. 注册`cell0`数据库：

   ```shell
   su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
   ```

5. 创建`cell1`单元格：

   ```shell
   su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
   #输出
   245a3d9b-1074-487a-bbe9-8b1baa634a9e
   ```

6. 填充新星数据库：

   ```shell
   su -s /bin/sh -c "nova-manage db sync" nova
   ```

7. 验证nova cell0和cell1是否正确注册：

   ```shell
   su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
   #输出
   +-------+--------------------------------------+右边还有3栏，放不下
   | Name  | UUID                                 |
   +-------+--------------------------------------+
   | cell0 | 00000000-0000-0000-0000-000000000000 |
   | cell1 | 245a3d9b-1074-487a-bbe9-8b1baa634a9e |
   +-------+--------------------------------------+
   ```

#### 3.4.1.3 完成安装
重启Compute服务：

  ```shell
service nova-api restart
service nova-scheduler restart
service nova-conductor restart
service nova-novncproxy restart
  ```

### 3.4.2 安装并配置计算节点

#### 3.4.2.1 安装和配置组件

1. 安装包：

   ```
   apt-get install nova-compute
   ```

2. 编辑`/etc/nova/nova.conf`文件并完成以下操作：
   在该`[DEFAULT]`部分中，配置`RabbitMQ`消息队列访问：
   
   >[DEFAULT]
   >
   >transport_url = rabbit://openstack:RABBIT_PASS@controller
   >

   替换`RABBIT_PASS`为您为`openstack` 帐户选择的密码`RabbitMQ`。

   在`[api]`和`[keystone_authtoken]`部分中，配置身份服务访问：
   >[api]
   >
   >auth_strategy = keystone
   >
   >[keystone_authtoken]
   >
   >auth_url = http://controller:5000/v3
   >memcached_servers = controller:11211
   >auth_type = password
   >project_domain_name = Default
   >user_domain_name = Default
   >project_name = service
   >username = nova
   >password = NOVA_PASS

   替换`NOVA_PASS`为您`nova`在Identity服务中为用户选择的密码。
   注意,注释掉或删除该`[keystone_authtoken]`部分中的任何其他选项 。

   在该`[DEFAULT]`部分中，配置`my_ip`选项：
   >[DEFAULT]
   >
   >my_ip = 192.168.2.51
   >

   在该`[DEFAULT]`部分中，启用对网络服务的支持：
   >[DEFAULT]
   >
   >use_neutron = true
   >firewall_driver = nova.virt.firewall.NoopFirewallDriver

注意,默认情况下，Compute使用内部防火墙服务。由于Networking包含防火墙服务，因此必须使用`nova.virt.firewall.NoopFirewallDriver`防火墙驱动程序禁用Compute防火墙服务。

配置**/etc/nova/nova.conf**的`[neutron]`部分。 有关更多详细信息，请参阅[网络服务安装指南](https://docs.openstack.org/neutron/stein/install/compute-install-ubuntu.html#configure-the-compute-service-to-use-the-networking-service)。

在该`[vnc]`部分中，启用并配置远程控制台访问：

>[vnc]
   >
   >enabled = true
   >server_listen = 0.0.0.0
   >server_proxyclient_address = $my_ip
   >novncproxy_base_url = http://controller:6080/vnc_auto.html

服务器组件侦听所有IP地址，并且代理组件仅侦听计算节点的管理接口IP地址。基本URL指示您可以使用Web浏览器访问此计算节点上的实例的远程控制台的位置。

注意,如果要访问远程控制台的Web浏览器驻留在无法解析`controller`主机名的主机上，则必须`controller`使用控制器节点的管理接口IP地址替换 。

在该`[glance]`部分中，配置Image服务API的位置：
>[glance]
   >
   >api_servers = http://controller:9292
   >

在该`[oslo_concurrency]`部分中，配置锁定路径：
>[oslo_concurrency]
   >
   >lock_path = /var/lib/nova/tmp
   >

在该`[placement]`部分中，配置Placement API：
>[placement]
   >
   >region_name = RegionOne
   >project_domain_name = Default
   >project_name = service
   >auth_type = password
   >user_domain_name = Default
   >auth_url = http://controller:5000/v3
   >username = placement
   >password = PLACEMENT_PASS

替换`PLACEMENT_PASS`为您`placement`在Identity服务中为用户选择的密码 。注释掉该`[placement]`部分中的任何其他选项。

#### 3.4.2.2 完成安装

1. 确定您的计算节点是否支持虚拟机的硬件加速：

   ```shell
   egrep -c '(vmx|svm)' /proc/cpuinfo
   ```

   如果此命令返回值是`1`或更高，则计算节点支持硬件加速，通常不需要其他配置。

   如果此命令返回值`0`，则您的计算节点不支持硬件加速，您必须配置`libvirt`为使用QEMU而不是KVM。

   编辑文件中的`[libvirt]`部分，`/etc/nova/nova-compute.conf`如下所示：
   >[libvirt]
   >
   >virt_type = qemu
   >

2. 重新启动Compute服务：

   ```shell
   service nova-compute restart
   ```

### 3.4.3 到控制节点将计算节点添加到单元数据库

1. 获取管理员凭据以启用仅管理员CLI命令，然后确认数据库中是否存在计算主机：

   ```shell
   source admin-openrc
   openstack compute service list --service nova-compute
   #输出 
   +----+--------------+----------+------+---------+-------+----------------------------+
   | ID | Binary       | Host     | Zone | Status  | State | Updated At                 |
   +----+--------------+----------+------+---------+-------+----------------------------+
   |  8 | nova-compute | compute1 | nova | enabled | up    | 2019-07-19T08:22:57.000000 |
   +----+--------------+----------+------+---------+-------+----------------------------+
   ```

2. 发现计算主机：

   ```shell
   su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
   #输出 #也有可能不一样，我后来复查了这里，所以我这里可能不是新装的样子
   Found 2 cell mappings.
   Skipping cell0 since it does not contain hosts.
   Getting computes from cell 'cell1': 245a3d9b-1074-487a-bbe9-8b1baa634a9e
   Found 0 unmapped computes in cell: 245a3d9b-1074-487a-bbe9-8b1baa634a9e
   ```
   
   注意,添加新计算节点时，必须在控制器节点上运行`nova-manage cell_v2 discover_hosts` 以注册这些新计算节点。或者，您可以在以下位置设置适当的间隔 ：`/etc/nova/nova.conf`

   >[scheduler]
   >discover_hosts_in_cells_interval = 300

### 3.4.4 验证操作

#### 3.4.4.1 验证操作仅需在控制节点运行

1. 来源`admin`凭据来访问仅管理员CLI命令：

   ```shell
   source admin-openrc
   ```

2. 列出服务组件以验证每个进程的成功启动和注册：

   ```shell
   openstack compute service list
   #输出   #最后一栏受限于文章宽度，省略了时间
   +----+----------------+------------+----------+---------+-------+--------------+
   | ID | Binary         | Host       | Zone     | Status  | State | Updated At   |
   +----+----------------+------------+----------+---------+-------+--------------+
   |  1 | nova-scheduler | controller | internal | enabled | up    | 2019-07-19 --|
   |  7 | nova-conductor | controller | internal | enabled | up    | 2019-07-19 --|
   |  8 | nova-compute   | compute1   | nova     | enabled | up    | 2019-07-19 --|
   +----+----------------+------------+----------+---------+-------+--------------+
   ```

   注意，此输出应指示在控制器节点上启用的两个服务组件以及在计算节点上启用的一个服务组件。

3. 列出Identity服务中的API端点以验证与Identity服务的连接：

   注意，端点列表可能会有所不同，具体取决于OpenStack组件的安装。

   ```shell
   openstack catalog list
   #输出
   +-----------+-----------+-----------------------------------------+
   | Name      | Type      | Endpoints                               |
   +-----------+-----------+-----------------------------------------+
   | nova      | compute   | RegionOne                               |
   |           |           |   internal: http://controller:8774/v2.1 |
   |           |           | RegionOne                               |
   |           |           |   admin: http://controller:8774/v2.1    |
   |           |           | RegionOne                               |
   |           |           |   public: http://controller:8774/v2.1   |
   |           |           |                                         |
   | glance    | image     | RegionOne                               |
   |           |           |   internal: http://controller:9292      |
   |           |           | RegionOne                               |
   |           |           |   admin: http://controller:9292         |
   |           |           | RegionOne                               |
   |           |           |   public: http://controller:9292        |
   |           |           |                                         |
   | placement | placement | RegionOne                               |
   |           |           |   internal: http://controller:8778      |
   |           |           | RegionOne                               |
   |           |           |   public: http://controller:8778        |
   |           |           | RegionOne                               |
   |           |           |   admin: http://controller:8778         |
   |           |           |                                         |
   | keystone  | identity  | RegionOne                               |
   |           |           |   public: http://controller:5000/v3/    |
   |           |           | RegionOne                               |
   |           |           |   admin: http://controller:5000/v3/     |
   |           |           | RegionOne                               |
   |           |           |   internal: http://controller:5000/v3/  |
   |           |           |                                         |
   +-----------+-----------+-----------------------------------------+
   ```

   注意，忽略此输出中的任何警告。

4. 列出Image服务中的图像以验证与Image服务的连接：

   ```shell
   openstack image list
   #输出
   +--------------------------------------+--------+--------+
   | ID                                   | Name   | Status |
   +--------------------------------------+--------+--------+
   | 643f8b18-6272-4333-9983-0fb650ebf0c9 | cirros | active |
   +--------------------------------------+--------+--------+
   ```

5. 检查单元格和放置API是否正常运行以及其他必要的先决条件是否到位：

   ```shell
   nova-status upgrade check
   #输出
   +--------------------------------------------------------------------+
   | Upgrade Check Results                                              |
   +--------------------------------------------------------------------+
   | Check: Cells v2                                                    |
   | Result: Success                                                    |
   | Details: No host mappings or compute nodes were found. Remember to |
   |   run command 'nova-manage cell_v2 discover_hosts' when new        |
   |   compute hosts are deployed.                                      |
   +--------------------------------------------------------------------+
   | Check: Placement API                                               |
   | Result: Success                                                    |
   | Details: None                                                      |
   +--------------------------------------------------------------------+
   | Check: Ironic Flavor Migration                                     |
   | Result: Success                                                    |
   | Details: None                                                      |
   +--------------------------------------------------------------------+
   | Check: Request Spec Migration                                      |
   | Result: Success                                                    |
   | Details: None                                                      |
   +--------------------------------------------------------------------+
   | Check: Console Auths                                               |
   | Result: Success                                                    |
   | Details: None                                                      |
   +--------------------------------------------------------------------+
   ```



## 3.5 网络服务



**重新配置网络接口将中断网络连接，因此建议使用本地终端会话来执行这些过程。**

### 3.5.1 主机网络

emmm。。。这部分重复了一下标题1.1-1.3的内容，就不说了



### 3.5.2 安装和配置控制节点

#### 3.5.2.1 先决条件

在配置OpenStack Networking（neutron）服务之前，必须创建数据库，服务凭据和API端点。

1. 要创建数据库，请完成以下步骤：

   使用数据库访问客户端以`root`用户身份连接到数据库服务器：

   ```shell
   mysql -u root -p
   ```

   创建`neutron`数据库：

   ```shell
   CREATE DATABASE neutron;
   ```

   授予对`neutron`数据库的适当访问权限，替换 `NEUTRON_DBPASS`为合适的密码：

   ```shell
   GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
     IDENTIFIED BY 'NEUTRON_DBPASS';
   GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
     IDENTIFIED BY 'NEUTRON_DBPASS';
   ```

   退出数据库访问客户端。

2. 来源`admin`凭据来访问仅管理员CLI命令：

   ```shell
   source admin-openrc
   ```

3. 要创建服务凭据，请完成以下步骤：

   创建`neutron`用户：

   ```shell
   openstack user create --domain default --password-prompt neutron
   #输出
   User Password:
   Repeat User Password:
   +---------------------+----------------------------------+
   | Field               | Value                            |
   +---------------------+----------------------------------+
   | domain_id           | default                          |
   | enabled             | True                             |
   | id                  | e195545b47324d1fb56c81cd80561d11 |
   | name                | neutron                          |
   | options             | {}                               |
   | password_expires_at | None                             |
   +---------------------+----------------------------------+
   ```

   将`admin`角色添加到`neutron`用户：

   ```shell
   openstack role add --project service --user neutron admin
   ```

   注意，此命令不提供输出。

   创建`neutron`服务实体：

   ```shell
openstack service create --name neutron \
     --description "OpenStack Networking" network
#输出
   +-------------+----------------------------------+
   | Field       | Value                            |
   +-------------+----------------------------------+
   | description | OpenStack Networking             |
   | enabled     | True                             |
   | id          | d819e88f42e642c8867c8db49e32c6ef |
   | name        | neutron                          |
   | type        | network                          |
   +-------------+----------------------------------+
   ```
   
4. 创建网络服务API端点：

   ```shell
   openstack endpoint create --region RegionOne \
     network public http://controller:9696
   #输出
   +--------------+----------------------------------+
   | Field        | Value                            |
   +--------------+----------------------------------+
   | enabled      | True                             |
   | id           | a925a892d81345689578481717a54d7e |
   | interface    | public                           |
   | region       | RegionOne                        |
   | region_id    | RegionOne                        |
   | service_id   | d819e88f42e642c8867c8db49e32c6ef |
   | service_name | neutron                          |
   | service_type | network                          |
   | url          | http://controller:9696           |
   +--------------+----------------------------------+
   
   openstack endpoint create --region RegionOne \
     network internal http://controller:9696
   #输出
   +--------------+----------------------------------+
   | Field        | Value                            |
   +--------------+----------------------------------+
   | enabled      | True                             |
   | id           | 4d32d47c2fee4f2fa9b81f0a906a5d3c |
   | interface    | internal                         |
   | region       | RegionOne                        |
   | region_id    | RegionOne                        |
   | service_id   | d819e88f42e642c8867c8db49e32c6ef |
   | service_name | neutron                          |
   | service_type | network                          |
   | url          | http://controller:9696           |
   +--------------+----------------------------------+
   
   openstack endpoint create --region RegionOne \
     network admin http://controller:9696
   #输出
   +--------------+----------------------------------+
   | Field        | Value                            |
   +--------------+----------------------------------+
   | enabled      | True                             |
   | id           | fcb3a04b0915449691dcff6eb2f3cf86 |
   | interface    | admin                            |
   | region       | RegionOne                        |
   | region_id    | RegionOne                        |
   | service_id   | d819e88f42e642c8867c8db49e32c6ef |
   | service_name | neutron                          |
   | service_type | network                          |
   | url          | http://controller:9696           |
   +--------------+----------------------------------+
   ```

##### 配置网络选项

您可以使用选项1和2表示的两种体系结构之一来部署网络服务。

选项1部署了最简单的架构，该架构仅支持将实例附加到提供商（外部）网络。没有自助（私有）网络，路由器或浮动IP地址。只有该`admin`特权用户或其他特权用户才能管理提供商网络。

选项2使用支持将实例附加到自助服务网络的第3层服务来增强选项1。该`demo`非特权用户或其他非特权用户可以管理自助服务网络，包括提供自助服务和提供商网络之间连接的路由器。此外，浮动IP地址提供与使用来自外部网络（如Internet）的自助服务网络的实例的连接。

自助服务网络通常使用覆盖网络。诸如VXLAN之类的覆盖网络协议包括额外的报头，这些报头增加了开销并减少了有效载荷或用户数据的可用空间。在不了解虚拟网络基础结构的情况下，实例尝试使用1500字节的默认以太网最大传输单元（MTU）来发送分组。Networking服务自动通过DHCP为实例提供正确的MTU值。但是，某些云映像不使用DHCP或忽略DHCP MTU选项，并且需要使用元数据或脚本进行配置。

注意，选项2还支持将实例附加到提供商网络。

选择以下网络选项之一以配置特定于其的服务。

[网络选项1：提供商网络](https://docs.openstack.org/neutron/stein/install/controller-install-option1-ubuntu.html)

[网络选项2：自助服务网络](https://docs.openstack.org/neutron/stein/install/controller-install-option2-ubuntu.html)

**我这里选择自助服务网络，功能更强大了，看起来也不是很复杂**

##### 安装组件

```shell
apt-get install neutron-server neutron-plugin-ml2 \
  neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent \
  neutron-metadata-agent
```

##### 配置服务器组件

编辑`/etc/neutron/neutron.conf`文件并完成以下操作：

在该`[database]`部分中，配置数据库访问：

>[database]
>
>connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron
>

替换`NEUTRON_DBPASS`为您为数据库选择的密码。

注意,注释掉或删除`[database]`部分中`connection`的任何其他选项 。

在该`[DEFAULT]`部分中，启用模块化第2层（ML2）插件，路由器服务和重叠的IP地址：

>[DEFAULT]
>
>core_plugin = ml2
>service_plugins = router
>allow_overlapping_ips = true

在该`[DEFAULT]`部分中，配置`RabbitMQ` 消息队列访问：

>[DEFAULT]
>
>transport_url = rabbit://openstack:RABBIT_PASS@controller
>

  替换`RABBIT_PASS`为您`openstack`在RabbitMQ中为帐户选择的密码 。

在`[DEFAULT]`和`[keystone_authtoken]`部分中，配置身份服务访问：

>[DEFAULT]
>
>auth_strategy = keystone
> 
>[keystone_authtoken]
>
>www_authenticate_uri = http://controller:5000
>auth_url = http://controller:5000
>memcached_servers = controller:11211
>auth_type = password
>project_domain_name = default
>user_domain_name = default
>project_name = service
>username = neutron
>password = NEUTRON_PASS

替换`NEUTRON_PASS`为您`neutron` 在Identity服务中为用户选择的密码。
注意,注释掉或删除该`[keystone_authtoken]`部分中的任何其他选项 。

在`[DEFAULT]`和`[nova]`部分中，配置网络以通知Compute网络拓扑更改：

>[DEFAULT]
>
>notify_nova_on_port_status_changes = true
>notify_nova_on_port_data_changes = true
>
>[nova]
>
>auth_url = http://controller:5000
>auth_type = password
>project_domain_name = default
>user_domain_name = default
>region_name = RegionOne
>project_name = service
>username = nova
>password = NOVA_PASS

替换`NOVA_PASS`为您`nova` 在Identity服务中为用户选择的密码。

在该`[oslo_concurrency]`部分中，配置锁定路径：

>[oslo_concurrency]
>
>lock_path = /var/lib/neutron/tmp
>

##### 配置模块化第2层（ML2）插件[¶](https://docs.openstack.org/neutron/stein/install/controller-install-option2-ubuntu.html#configure-the-modular-layer-2-ml2-plug-in)

ML2插件使用Linux桥接机制为实例构建第2层（桥接和交换）虚拟网络基础架构。

编辑`/etc/neutron/plugins/ml2/ml2_conf.ini`文件并完成以下操作：
在该`[ml2]`部分中，启用flat，VLAN和VXLAN网络：

>[ml2]
>
>type_drivers = flat,vlan,vxlan
>

在该`[ml2]`部分中，启用VXLAN自助服务网络：

>[ml2]
>
>tenant_network_types = vxlan
>

在该`[ml2]`部分中，启用Linux桥和第2层填充机制：

>[ml2]
>
>mechanism_drivers = linuxbridge,l2population
>

警告,配置ML2插件后，删除`type_drivers`选项中的值 可能会导致数据库不一致。
注意,Linux网桥代理仅支持VXLAN重叠网络。

在该`[ml2]`部分中，启用端口安全性扩展驱动程序：

>[ml2]
>
>extension_drivers = port_security
>

在该`[ml2_type_flat]`部分中，将提供商虚拟网络配置为扁平网络：

>[ml2_type_flat]
>
>flat_networks = provider
>

在该`[ml2_type_vxlan]`部分中，为自助服务网络配置VXLAN网络标识符范围：

>[ml2_type_vxlan]
>
>vni_ranges = 1:1000
>

在该`[securitygroup]`部分中，启用ipset以提高安全组规则的效率：

>[securitygroup]
>
>enable_ipset = true
>

##### 配置Linux桥代理

Linux网桥代理为实例构建第2层（桥接和交换）虚拟网络基础架构并处理安全组。

编辑`/etc/neutron/plugins/ml2/linuxbridge_agent.ini`文件并完成以下操作：
在该`[linux_bridge]`部分中，将提供者虚拟网络映射到提供者物理网络接口：

>[linux_bridge]
>physical_interface_mappings = provider:ens38

替换`ens38`为基础提供程序物理网络接口的名称。有关 更多信息，请参阅[主机网络](https://docs.openstack.org/neutron/stein/install/environment-networking-ubuntu.html)

在该`[vxlan]`部分中，启用VXLAN重叠网络，配置处理覆盖网络的物理网络接口的IP地址，并启用第2层填充：

>[vxlan]
>enable_vxlan = true
>local_ip = 192.168.2.52
>l2_population = true

替换`192.168.2.52`为处理覆盖网络的基础物理网络接口的IP地址。示例体系结构使用管理接口将流量隧道传输到其他节点。因此，请替换`OVERLAY_INTERFACE_IP_ADDRESS`为控制器节点的管理IP地址。有关更多信息，请参阅 [主机网络](https://docs.openstack.org/neutron/stein/install/environment-networking-ubuntu.html)

在该`[securitygroup]`部分中，启用安全组并配置Linux网桥iptables防火墙驱动程序：

>[securitygroup]
>
>enable_security_group = true
>firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

通过验证以下所有`sysctl`值都设置为`1`：确保您的Linux操作系统内核支持网桥过滤器：

```shell
sysctl net.bridge.bridge-nf-call-iptables
#输出
net.bridge.bridge-nf-call-iptables = 1

sysctl net.bridge.bridge-nf-call-ip6tables
#输出
net.bridge.bridge-nf-call-ip6tables = 1
```

要启用网络桥接支持，通常`br_netfilter`需要加载内核模块，Ubuntu Server 18.04应该是默认加载的。

```shell
modprobe br_netfilter
```



##### 配置第3层代理

第3层（L3）代理为自助服务虚拟网络提供路由和NAT服务。

编辑`/etc/neutron/l3_agent.ini`文件并完成以下操作：

在本`[DEFAULT]`节中，配置Linux桥接接口驱动程序和外部网桥：

>[DEFAULT]
>
>interface_driver = linuxbridge
>

##### 配置DHCP代理

DHCP代理为虚拟网络提供DHCP服务。

编辑`/etc/neutron/dhcp_agent.ini`文件并完成以下操作：
在本`[DEFAULT]`节中，配置Linux桥接接口驱动程序，Dnsmasq DHCP驱动程序，并启用隔离的元数据，以便提供商网络上的实例可以通过网络访问元数据：

>[DEFAULT]
>
>interface_driver = linuxbridge
>dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
>enable_isolated_metadata = true


##### 配置元数据代理

元数据代理提供配置信息，例如实例的凭证。

编辑`/etc/neutron/metadata_agent.ini`文件并完成以下操作：

在该`[DEFAULT]`部分中，配置元数据主机和共享密钥：

>[DEFAULT]
>
>nova_metadata_host = controller
>metadata_proxy_shared_secret = METADATA_SECRET

替换`METADATA_SECRET`为元数据代理的适当秘密。

##### 配置Compute服务以使用Networking服务

注意，必须安装Nova计算服务才能完成此步骤。有关更多详细信息，请参阅[docs网站](https://docs.openstack.org/)的“ 安装指南”部分 下的计算安装指南 。

编辑`/etc/nova/nova.conf`文件并执行以下操作：

在该`[neutron]`部分中，配置访问参数，启用元数据代理并配置密码：

>[neutron]
>
>url = http://controller:9696
>auth_url = http://controller:5000
>auth_type = password
>project_domain_name = default
>user_domain_name = default
>region_name = RegionOne
>project_name = service
>username = neutron
>password = NEUTRON_PASS
>service_metadata_proxy = true
>metadata_proxy_shared_secret = METADATA_SECRET

替换`NEUTRON_PASS`为您`neutron` 在Identity服务中为用户选择的密码。
替换`METADATA_SECRET`为您为元数据代理选择的秘密。

##### 完成安装

填充数据库：

```shell
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

注意，数据库填充稍后会出现在网络中，因为该脚本需要完整的服务器和插件配置文件。

1. 重新启动Compute API服务：

   ```shell
   service nova-api restart
   ```

2. 重新启动网络服务。

   ```shell
service neutron-server restart
   service neutron-linuxbridge-agent restart
   service neutron-dhcp-agent restart
   service neutron-metadata-agent restart
   service neutron-l3-agent restart
   ```

### 3.5.3 安装和配置计算节点

##### 安装组件

```shell
apt-get install neutron-linuxbridge-agent
```

##### 配置公共组件

Networking公共组件配置包括身份验证机制，消息队列和插件。

注意,默认配置文件因分发而异。您可能需要添加这些部分和选项，而不是修改现有的部分和选项。此外，`...`配置片段中的省略号（）表示您应保留的潜在默认配置选项。

编辑`/etc/neutron/neutron.conf`文件并完成以下操作：

在该`[database]`部分中，注释掉任何`connection`选项，因为计算节点不直接访问数据库。

在该`[DEFAULT]`部分中，配置`RabbitMQ` 消息队列访问：

>[DEFAULT]
>
>transport_url = rabbit://openstack:RABBIT_PASS@controller
>

替换`RABBIT_PASS`为您`openstack` 在RabbitMQ中为帐户选择的密码。

在`[DEFAULT]`和`[keystone_authtoken]`部分中，配置身份服务访问：

>[DEFAULT]
>
>auth_strategy = keystone
>
>[keystone_authtoken]
>
>www_authenticate_uri = http://controller:5000
>auth_url = http://controller:5000
>memcached_servers = controller:11211
>auth_type = password
>project_domain_name = default
>user_domain_name = default
>project_name = service
>username = neutron
>password = NEUTRON_PASS

替换`NEUTRON_PASS`为您`neutron` 在Identity服务中为用户选择的密码。

注意，注释掉或删除该`[keystone_authtoken]`部分中的任何其他选项 。

在该`[oslo_concurrency]`部分中，配置锁定路径：

>[oslo_concurrency]
>
>lock_path = /var/lib/neutron/tmp
>

##### 配置网络选项

选择为控制器节点选择的相同网络选项，以配置特定于其的服务。然后，返回此处并继续 [配置计算服务以使用网络服务](https://docs.openstack.org/neutron/stein/install/compute-install-ubuntu.html#neutron-compute-compute-ubuntu)。

[网络选项1：提供商网络](https://docs.openstack.org/neutron/stein/install/compute-install-option1-ubuntu.html)

[网络选项2：自助服务网络](https://docs.openstack.org/neutron/stein/install/compute-install-option2-ubuntu.html)

**这里依旧是选择第二个选项**

##### 配置Linux桥代理

Linux网桥代理为实例构建第2层（桥接和交换）虚拟网络基础架构并处理安全组。

编辑`/etc/neutron/plugins/ml2/linuxbridge_agent.ini`文件并完成以下操作：
在该`[linux_bridge]`部分中，将提供者虚拟网络映射到提供者物理网络接口：

>[linux_bridge]
>physical_interface_mappings = provider:ens38

替换`ens38`为基础提供程序物理网络接口的名称。有关 更多信息，请参阅[主机网络](https://docs.openstack.org/neutron/stein/install/environment-networking-ubuntu.html)

在该`[vxlan]`部分中，启用VXLAN重叠网络，配置处理覆盖网络的物理网络接口的IP地址，并启用第2层填充：

>[vxlan]
>enable_vxlan = true
>local_ip = 192.168.2.53
>l2_population = true

替换`192.168.2.53`为处理覆盖网络的基础物理网络接口的IP地址。

在该`[securitygroup]`部分中，启用安全组并配置Linux网桥iptables防火墙驱动程序：

>[securitygroup]
>
>enable_security_group = true
>firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

通过验证以下所有`sysctl`值都设置为`1`：确保您的Linux操作系统内核支持网桥过滤器：

```shell
sysctl net.bridge.bridge-nf-call-iptables
#输出
net.bridge.bridge-nf-call-iptables = 1

sysctl net.bridge.bridge-nf-call-ip6tables
#输出
net.bridge.bridge-nf-call-ip6tables = 1
```

要启用网络桥接支持，通常`br_netfilter`需要加载内核模块，Ubuntu Server 18.04应该是默认加载的。

```shell
modprobe br_netfilter
```

##### 配置Compute服务以使用Networking服务

编辑`/etc/nova/nova.conf`文件并完成以下操作：

在该`[neutron]`部分中，配置访问参数：

>[neutron]
>
>url = http://controller:9696
>auth_url = http://controller:5000
>auth_type = password
>project_domain_name = default
>user_domain_name = default
>region_name = RegionOne
>project_name = service
>username = neutron
>password = NEUTRON_PASS

替换`NEUTRON_PASS`为您`neutron` 在Identity服务中为用户选择的密码。

##### 完成安装

1. 重新启动Compute服务：

   ```shell
   service nova-compute restart
   ```

2. 重新启动Linux桥代理：

   ```shell
   service neutron-linuxbridge-agent restart
   ```



## 3.6 仪表盘

### 3.6.1 仪表盘仅在控制节点安装运行

#### 安装和配置组件

1. 安装包：

   ```shell
   apt-get install openstack-dashboard
   ```

2. 编辑 `/etc/openstack-dashboard/local_settings.py` 文件并完成以下操作：

   配置仪表板以在`controller`节点上使用OpenStack服务 ：

   >OPENSTACK_HOST = "controller"
   >

   在仪表板配置部分中，允许主机访问仪表板：

   >ALLOWED_HOSTS = ['one.example.com', 'two.example.com']
   >

   注意，不要`ALLOWED_HOSTS`在Ubuntu配置部分下编辑参数。`ALLOWED_HOSTS`也可以`['*']`接受所有主机。这可能对开发工作有用，但可能不安全，不应在生产中使用。有关详细信息，请参阅 [Django文档](https://docs.djangoproject.com/en/dev/ref/settings/#allowed-hosts)。

   配置`memcached`会话存储服务：

   >SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
   >   
   >   CACHES = {
   >       'default': {
   >            'BACKEND':    'django.core.cache.backends.memcached.MemcachedCache',
   >            'LOCATION': 'controller:11211',
   >       }
   >   }
   

   

注意，注释掉任何其他会话存储配置。

启用Identity API版本3：

>OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
   >

启用对域的支持：

>OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
   >

配置API版本：

>OPENSTACK_API_VERSIONS = {
   >     "identity": 3,
   >     "image": 2,
   >     "volume": 3,
   >     }
   >

配置`Default`为通过仪表板创建的用户的默认域：

>OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
   >

配置`user`为通过仪表板创建的用户的默认角色：

>OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
   >

如果选择网络选项1，请禁用对第3层网络服务的支持：

>OPENSTACK_NEUTRON_NETWORK = {
   >
   >     'enable_router': False,
   >     'enable_quotas': False,
   >     'enable_ipv6': False,
   >     'enable_distributed_router': False,
   >     'enable_ha_router': False,
   >     'enable_lb': False,
   >     'enable_firewall': False,
   >     'enable_vpn': False,
   >     'enable_fip_topology_check': False,
   >      }
   >

（可选）配置时区：

>TIME_ZONE = "Asia/Shanghai"
   >

替换`TIME_ZONE`为适当的时区标识符。有关更多信息，请参阅[时区列表](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)。

3. `/etc/apache2/conf-available/openstack-dashboard.conf`如果未包含，请添加以下行 。

   >WSGIApplicationGroup %{GLOBAL}
   >

#### 完成安装

重新加载Web服务器配置：

```shell
service apache2 reload
```

#### 验证操作

使用Web浏览器访问仪表板 `http://controller/horizon`。

使用`admin`或`demo`用户和`default`域凭据进行身份验证。

