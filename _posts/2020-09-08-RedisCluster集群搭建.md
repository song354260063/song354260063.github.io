---
layout:     post                    # 使用的布局（不需要改）
title:      Redis Cluster集群搭建           # 标题
subtitle:   离线搭建 Redis Cluster 集群   #副标题
date:       2020-09-08              # 时间
author:     songjunhao                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Redis
    - Redis Cluster
---

### 引言

本文记录搭建Redis Cluster的流程。

Redis Cluster有两种方式可以搭建，一种是直接用redis原生命令直接配置（较为麻烦），第二种是使用 redis-trib.rb 脚本直接进行搭建，但需要依赖 ruby 。

本文记录第二种。

已验证过 redis 3.2.0 和 4.0.0 版本都没问题。

### 步骤

1. 安装ruby
2. 安装rubygem redis
3. redis实例配置文件编写，启动
4. 执行redis-trib.rb

> 全文均为离线安装。安装包在文末附件部分。

#### 1.安装ruby

上传 all.tar 到服务器中。
执行 tar -xvf all.tar 解压。
进入 ruby 文件夹 执行 rpm -ivh ruby* 。

```
tar -xvf all.tar
cd ruby
rpm -ivh ruby*
```

#### 2.安装rubygem redis

```
gem install –local /redis-3.2.0.gem
```

执行上述命令即可，第二步必须第一步成功，第二步成功后提示 success。安装刚开始可能提示有问题，稍等一会 最后会提示Success，不是很快 有点慢，耐心等待。

#### 3.redis 实例配置启动

ruby 环境准备完成后，那么需要配置集群中 redis 实例了，需要先配置好然后启动实例。

假设设置三主三从，那么就每一个实例单独建一个文件夹，每一个文件夹下存放自己的配置文件，每一个实例端口都设置为不同（当然机器不同端口相同也无所谓）。

配置好之后逐个执行
```
redis-server redis.conf
```

redis 简洁基础配置（将端口 3523 全都改一下成各自实例。）
```
# 端口
port 3523
# 是否后台运行
daemonize yes
dir "/home/song/redis-3523/data"
logfile "3523.log"
dbfilename "dump-3523.db"
cluster-enabled yes
cluster-config-file "nodes-3523.conf"
cluster-require-full-coverage no
requirepass "密码"
# 配置 requirepass 的情况下 masterauth主从都需要配置上，不配置从节点无法同步数据
masterauth "密码"
# 最大内存
maxmemory 5gb
```

**注意**：设置了密码需要特殊处理一下 client.rb 文件

否则执行创建集群 会提示 Sorry, can't connect to node

需要修改

/usr/local/rvm/gems/ruby-2.4.1/gems/redis-3.2.0/lib/redis/client.rb

可能路径不同，find / -name client.rb找一下

```
 DEFAULTS = {
      :url => lambda { ENV["REDIS_URL"] },
      :scheme => "redis",
      :host => "127.0.0.1",
      :port => 6379,
      :path => nil,
      :timeout => 5.0,
      :password => "xxxxxx",
      :db => 0,
      :driver => nil,
      :id => nil,
      :tcp_keepalive => 0,
      :reconnect_attempts => 1,
      :inherit_socket => false
    }
```
添加或修改password就OK了，把 :password => "xxxxxx" 里的 xxxxxx 改为实际密码

#### 4.创建集群

到redis安装目录下执行

--replicas 0 表每个主有几个从 0 表示没有从节点。

```
./redis-trib.rb create --replicas 1 127.0.0.1:3456 127.0.0.1:3457 127.0.0.01:3458 127.0.0.1:3459 127.0.0.1:3460 127.0.0.01:3461
```

关于主从节点的选择及槽的分配，其算法如下：

1> 把节点按照host分类，这样保证master节点能分配到更多的主机中。

2> 遍历host列表，从每个host列表中弹出一个节点，放入interleaved数组。直到所有的节点都弹出为止。

3> 将interleaved数组中前master个数量的节点保存到masters数组中。

4> 计算每个master节点负责的slot数量，16384除以master数量取整，这里记为N。

5> 遍历masters数组，每个master分配N个slot，最后一个master，分配剩下的slot。

6> 接下来为master分配slave，分配算法会尽量保证master和slave节点不在同一台主机上。对于分配完指定slave数量的节点，还有多余的节点，也会为这些节点寻找master。分配算法会遍历两次masters数组。

7> 第一次遍历master数组，在余下的节点列表找到replicas数量个slave。每个slave为第一个和master节点host不一样的节点，如果没有不一样的节点，则直接取出余下列表的第一个节点。

8> 第二次遍历是分配节点数除以replicas不为整数而多出的一部分节点。


执行命令完成后如下图所示，则集群创建成功。

![image.png](https://i.loli.net/2020/09/08/c1hnS9EITJUYdRm.png)

#### 5.附件

1.**ruby 离线安装包**
all.tar 下载地址:https://pan.baidu.com/s/14DleRZT4FOvfBHTUHDq0OA  密码:yevn

2.**redis-3.2.0.gem**

链接:https://pan.baidu.com/s/16sU-Fut5QwuioIzf3skFCQ  密码:0x9d

### 参考致谢

https://blog.csdn.net/xiaobo060/article/details/80616718
https://www.cnblogs.com/ivictor/p/9768010.html

