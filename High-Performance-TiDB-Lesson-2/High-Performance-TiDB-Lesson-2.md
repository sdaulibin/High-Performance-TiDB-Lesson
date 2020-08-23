#### 使用 TiUP 部署 TiDB 集群

##### 环境准备

> 由于机器资源限制，在个人笔记本进行测试，笔记本配置为MacBook Pro(2015,4c16g)下

- 使用parallels新建了四个虚拟机，1台中控机器，3台目标机器，安装的操作系统版本都为centos7
- 中控机器分配的资源为1c2g，最大64g的硬盘空间
- 目标机器每台分配的资源为2c4g，最大64g的硬盘空间

![image-20200823213419487](https://i.loli.net/2020/08/23/Pmngr9iDhJeA7Xo.png)

- 每个虚拟机配置了网络，使用centos7的初始化脚本对每个虚拟机进行初始化，包括防火墙、yum源、时区等

  ```
  ## 下载脚本并执行，来自pigx，https://www.bilibili.com/video/av77344954?p=2
  curl -O http://pigx.vip/os7init.sh
  ```

##### 手动配置 SSH 互信及 sudo 免密码

```
## 使用root登录每台机器，创建tidb用户，并设置密码
useradd tidb && passwd tidb
## 执行以下命令，将 tidb ALL=(ALL) NOPASSWD: ALL 添加到文件末尾，即配置好 sudo 免密码。
visudo
tidb ALL=(ALL) NOPASSWD: ALL
## 以 tidb 用户登录到中控机，执行以下命令。按提示输入部署目标机器 tidb 用户密码，执行成功后即创建好 SSH 互信，其他机器同理。
ssh-keygen -t rsa -C "sdaulibin@126.com"
ssh-copy-id -i ~/.ssh/id_rsa.pub 192.168.199.227
ssh-copy-id -i ~/.ssh/id_rsa.pub 192.168.199.228
ssh-copy-id -i ~/.ssh/id_rsa.pub 192.168.199.229
## 以 tidb 用户登录中控机，通过 ssh 的方式登录目标机器 IP。如果不需要输入密码并登录成功，即表示 SSH 互信配置成功。
ssh 192.168.199.227
ssh 192.168.199.228
ssh 192.168.199.229
```

##### 目标主机安装numactl

```shell
sudo yum -y install numactl
```

##### 中控、目标主机安装sshpass

```shell
sudo yum install sshpass
sshpass -V
```

##### 在中控机上安装 TiUP 组件

```
## 使用tidb用户
curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
source .bash_profile
which tiup
## [tidb@binginx-centos ~]$ which tiup
## ~/.tiup/bin/tiup
tiup update --self && tiup update cluster
## download https://tiup-mirrors.pingcap.com/tiup-v1.0.9-linux-amd64.tar.gz 4.32 MiB / 4.32 MiB 100.00% 1.89 MiB p/s
## Updated successfully!
## component cluster version v1.0.9 is already installed
## Updated successfully!
tiup --binary cluster
## /home/tidb/.tiup/components/cluster/v1.0.9/tiup-cluster
```

##### 编辑初始化配置文件

```
git clone https://github.com/pingcap/tiup.git
cd tiup/examples/
cp topology.example.yaml topology.yaml
vim topology.yaml
## 主要修改机器的ip，使用最小拓扑架构，注释tispark、tiflash
## pd_servers:
##   - host: 192.168.199.227
##   - host: 192.168.199.228
##   - host: 192.168.199.229
## tidb_servers:
##   - host: 192.168.199.227
## tikv_servers:
##   - host: 192.168.199.227
##   - host: 192.168.199.228
##   - host: 192.168.199.229
```

##### 执行部署命令

```
# 已设置免密和ssh互相信任，user为tidb
tiup cluster deploy tidb-test nightly ./topology.yaml --user tidb
```

![image-20200823220641516.png](https://i.loli.net/2020/08/23/2DIA6psEFfvXj4H.png)

##### 检查部署的 TiDB 集群情况

```
tiup cluster display tidb-test
## 都为未启动状态
```

![image-20200823220757004.png](https://i.loli.net/2020/08/23/Wj6VRxpwCr2MIce.png)

##### 启动集群，并验证集群状态

```
tiup cluster start tidb-test
tiup cluster display tidb-test
mysql -u root -h 192.168.199.227 -P 4000
```

![image-20200823220911177.png](https://i.loli.net/2020/08/23/rgFJ51kHNBbTMhi.png)

![image-20200823220926884.png](https://i.loli.net/2020/08/23/nTiFYtxBX8IJAoG.png)

![image-20200823221012176.png](https://i.loli.net/2020/08/23/c2ZpdRy4jYNg1GK.png)

##### 检查dashboard

```
## 访问 http://192.168.199.227:2379/dashboard，root用户无需密码，高级测试，查看tidb的性能分析报告
```

![image-20200823221326640.png](https://i.loli.net/2020/08/23/c9OA8r4EzhYPIDF.png)

![image-20200823221357844.png](https://i.loli.net/2020/08/23/dntuRTKjbkOwv2F.png)

#### 使用sysbench进行测试

##### 安装sysbench

```
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.rpm.sh | sudo bash
sudo yum -y install sysbench
```

##### sysbench 配置

```
mkdir sysbench
cd sysbench
vim sysbench.cfg
## mysql-host=192.168.199.227
## mysql-db=tidbtest
## mysql-port=4000
## mysql-user=root
## time=60
## threads=16
## report-interval=10
## db-driver=mysql
```

##### 创建数据库

```
mysql -u root -h 192.168.199.227 -P 4000
create database tidbtest;
use tidbtest;
## 导入数据前设置为乐观事务模式
set global tidb_disable_txn_auto_retry=off;
```

##### 导入数据

```
sysbench oltp_update_non_index --config-file=sysbench.cfg --threads=32 --tables=32 --table-size=10000 prepare
```

##### Point select测试

```
## threads=1
sysbench --config-file=sysbench.cfg oltp_point_select --tables=32 --table-size=10000 run
```

![image-20200823223840562.png](https://i.loli.net/2020/08/23/r8WdUPakEIpmYZe.png)

```
## threads=64
sysbench --config-file=sysbench.cfg oltp_point_select --threads=64 --tables=32 --table-size=10000 run
```

![image-20200823224236524.png](https://i.loli.net/2020/08/23/sgKQcVnR9lumLj4.png)

> 线程数增加到64后，qps有了将近6~8倍的提升

##### Update index 测试

```
## threads=16
sysbench --config-file=sysbench.cfg oltp_update_index --threads=16 --tables=32 --table-size=10000 run
```

![image-20200823224833491.png](https://i.loli.net/2020/08/23/ql7tIG3xWMCVzmH.png)

```
## threads=64
sysbench --config-file=sysbench.cfg oltp_update_index --threads=64 --tables=32 --table-size=10000 run
```

![image-20200823225102864.png](https://i.loli.net/2020/08/23/GYfgrHyRTzPeJvS.png)

> 线程数增加到64后，tps有了将近2倍的提升

#### 使用go-tpc测试

##### 安装go-tpc

```shell
git clone https://github.com/pingcap/go-tpc.git
cd go-tpc
make build
```

##### 准备数据

```
mysql -u root -h 192.168.199.227 -P 4000
create database tpcc
cd go-tpcc/bin
./go-tpc tpcc -H 192.168.199.227 -P 4000 -D tpcc --warehouses 8 prepare -T 8
```

![image-20200823231739420.png](https://i.loli.net/2020/08/23/ArXiP3mISbqTQVG.png)

##### 测试数据

```
## threads=4
./go-tpc tpcc -H 192.168.199.227 -P 4000 -D tpcc --warehouses 8 run --time=1m --threads=4
```

![image-20200823232125086.png](https://i.loli.net/2020/08/23/ohUsfGCtjVYSwEp.png)

```
## threads=16
./go-tpc tpcc -H 192.168.199.227 -P 4000 -D tpcc --warehouses 8 run --time=1m --threads=16
```

![image-20200823232328624.png](https://i.loli.net/2020/08/23/OUnK7EeS3GYPawM.png)

> 线程数增加到16后，tpmc有了将近1.5倍的提升

```
## threads=64
./go-tpc tpcc -H 192.168.199.227 -P 4000 -D tpcc --warehouses 8 run --time=1m --threads=64
```

![image-20200823234753978.png](https://i.loli.net/2020/08/23/7VIi6n3km5QTEG4.png)

> 线程数增加到64后，tpmc的提升不是很大