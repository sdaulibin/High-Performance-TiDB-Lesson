#### 环境准备

> 由于机器资源限制，在个人笔记本进行测试，笔记本配置为MacBook Pro(2015,4c16g)下，cpu：2.2 GHz 四核Intel Core i7，内存：16 GB 1600 MHz DDR3

- 使用parallels新建了四个虚拟机，1台中控机器，3台目标机器，安装的操作系统版本都为centos7

- 中控机器分配的资源为1c2g，最大64g的硬盘空间，ip为192.168.199.226

- 目标机器每台分配的资源为2c4g，最大64g的硬盘空间，ip地址分别为192.168.199.227、192.168.199.228、192.168.199.229

- 使用haproxy对三台tidb做负载均衡，端口3306

  | 192.168.199.227 | pd，tikv，tidb |
  | :-------------- | :------------- |
  | 192.168.199.228 | pd，tikv，tidb |
  | 192.168.199.229 | pd，tikv，tidb |
  | 192.168.199.226 | haproxy        |

#### 使用go-tpc测试

##### 安装go-tpc

```shell
git clone https://github.com/pingcap/go-tpc.git
cd go-tpc
make build
```

##### 准备数据

```
mysql -u root -h 192.168.199.226 -P 3306 -u root
create database tpcc
cd go-tpcc/bin
./go-tpc tpcc -H 192.168.199.226 -P 3306 -D tpcc --warehouses 8 prepare -T 8
```

![image-20200831230137988.png](https://i.loli.net/2020/09/01/jpMhg3KTfyksBcd.png)

#### Profile TiDB

##### 测试数据

```
## threads=16
./go-tpc tpcc -H 192.168.199.226 -P 3306 -D tpcc --warehouses 8 run --time=1m --threads=16
```

##### 生成Profile文件

```
mkdir profiles
curl http://192.168.199.227:10080/debug/zip?seconds=80 --output tidb-227-debug.zip
/go-tpc/bin/go-tpc tpcc -H 192.168.199.226 -P 3306 -D tpcc --warehouses 8 run --time=1m --threads=16
curl http://192.168.199.228:10080/debug/zip?seconds=80 --output tidb-228-debug.zip
/go-tpc/bin/go-tpc tpcc -H 192.168.199.226 -P 3306 -D tpcc --warehouses 8 run --time=1m --threads=16
curl http://192.168.199.229:10080/debug/zip?seconds=80 --output tidb-229-debug.zip
/go-tpc/bin/go-tpc tpcc -H 192.168.199.226 -P 3306 -D tpcc --warehouses 8 run --time=1m --threads=16
```

- 192.168.199.227 tpmc 3298

![image-20200831231720908.png](https://i.loli.net/2020/09/01/baRGgxwhemyNT6p.png)

- 192.168.199.228 tpmc 2964

![image-20200831231758463.png](https://i.loli.net/2020/09/01/JX9W1wFKNzxPuVl.png)

- 192.168.199.229 tpmc 3379

![image-20200831232117594.png](https://i.loli.net/2020/09/01/w19qfaVnIsJSlNm.png)

- profile文件

![image-20200831232246294.png](https://i.loli.net/2020/09/01/vgh25PwkqXl7NRt.png)

```
## 下载zip文件到主机，安装graphviz，解压zip，打开profile文件
scp tidb@192.168.199.226:/home/tidb/profile/tidb-227-debug.zip .
scp tidb@192.168.199.226:/home/tidb/profile/tidb-228-debug.zip .
scp tidb@192.168.199.226:/home/tidb/profile/tidb-229-debug.zip .
brew install graphviz
go tool pprof -http=:8080 profile
go tool pprof -http=:8080 heap
go tool pprof -http=:8080 mutex
```

- 192.168.199.227 Profile Graph

![image-20200901112929026.png](https://i.loli.net/2020/09/01/JU6MNekaQFn9Hsv.png)

192.168.199.227 Profile Flame Graph

![image-20200901143624421.png](https://i.loli.net/2020/09/01/en814udypxw5Xsg.png)

- 192.168.199.227 heap

![image-20200901113102712.png](https://i.loli.net/2020/09/01/P1rhDoBV9QOm47i.png)

- 192.168.199.228 Profile Graph 

![image-20200901113247030](https://i.loli.net/2020/09/01/uYMnscNQ6eT9bHi.png)

- 192.168.199.228 Profile Flame Graph

![image-20200901143900081.png](https://i.loli.net/2020/09/01/YCtoKW4IPQrG2Ui.png)

- 192.168.199.228 heap

  ![image-20200901113356567](https://i.loli.net/2020/09/01/ukPAbcgmFnZV1YS.png)

- 192.168.199.229 Profile Graph

  ![image-20200901113558781](https://i.loli.net/2020/09/01/eIrd43MZ8BED9qc.png)

- 192.168.199.229 Profile Flame Graph

  ![image-20200901144027313.png](https://i.loli.net/2020/09/01/o2jEKftOD5AyTRw.png)
  
- 192.168.199.229 heap

  ![image-20200901113625080](https://i.loli.net/2020/09/01/yUXk5PlJ8stvEY6.png)

##### 问题分析

- 通过分析三台TiDB cpu的火焰图，发现server.(*Server).xxx的一系列方法耗时较长，这部分主要功能是解析执行sql
- 通过分析三台TiDB heap的火焰图，发现tikv.(*copIteratorWorker).run.xxx的一系列方法占用heap较大

#### Profile TiKV

##### 测试数据

```
## threads=16
./go-tpc tpcc -H 192.168.199.226 -P 3306 -D tpcc --warehouses 8 run --time=1m --threads=16
```

##### 使用dashboard查看TiKV火焰图

- 选中三个TiKV实例

![image-20200831233712358.png](https://i.loli.net/2020/09/01/6vsR4qmLSMxgAYr.png)

- 分析完成

![image-20200831233945410.png](https://i.loli.net/2020/09/01/EVyjb2S9CcQLD7K.png)

- TiKV 192.168.199.227 Profile Flame Graph

![image-20200901221848014.png](https://i.loli.net/2020/09/01/fpKuq5B9FgAvYda.png)

- TiKV 192.168.199.228 Profile Flame Graph

  ![image-20200901222058713.png](https://i.loli.net/2020/09/01/MiOY8gByALWZKnU.png)

- TiKV 192.168.199.229 Profile Flame Graph

  ![image-20200901222209645.png](https://i.loli.net/2020/09/01/EdB5hv6IZCFWaxs.png)

##### 问题分析

- 通过分析三台TiKV cpu的火焰图，发现grpc-server、raftstore耗时较长