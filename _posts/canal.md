---
layout:     post
title:      "Canal"
subtitle:   "Canal简明教程"
date:       2024-08-24 12:00:00
author:     "Yangzx"
header-img: "img/post-bg-2015.jpg"
tags:
    - Canal
---

# Canal
网上也有很多canal的教程，但是我个人感觉很多地方说得过于繁琐了，弄得太负责反而不好理解。所以我自己总结归纳了：
## 简介
Canal是阿里云开源的用于数据库自动化同步的工具，它可以监听mysql的binlog日志，根据binlog日志的执行内容，来将数据同步到其他数据库。**缺点是它仅仅是针对mysql数据库**，其他数据库可能不支持。
## 部署

它需要部署2个容器：
1. deployer（server）：它的作用是**从mysql读取binlog，然后存起来**
2. adapter：它的作用是**从deployer里读消息，写入目标数据库**。

可以认为一个是读，一个是写。想要使用Canal，只需要改好这两个程序的配置，再启动起来就可以了。

除了这两个外，其实还有一个admin，就是一个用来管理的ui界面，可以不用下载。

建议直接部署在Linux下或使用Docker容器，因为它提供的启动程序是.sh文件，但是在windows中是一般没有.sh的启动环境的。如果windows本地安装了Git，可以使用Git Bash的bash命令来运行。但是我认为最好还是用linux或直接用Docker。我是使用Docker来部署的。

## 配置
### deployer
deployer包下的conf自然就是配置文件了，映入眼帘的有两个properties，我们可以打开看看，看里面的参数命名就可以知道，他们都是配置admin这个ui界面的，如果我们不使用的话其实完全可以不用理。处理数据连接的properties是在example下的。

看上去有挺多参数的，但是根据我们连接数据库的常识，我们一般要使得deployer能够连接到我们的mysql，一般要改的参数无非就3个：**IP地址和端口、数据库用户名、密码**。

它们对应的就是下面三个参数：
1. canal.instance.master.address
2. canal.instance.dbUsername
3. canal.instance.dbPassword

改了这三个参数就OK了，如果想深入了解其他参数是什么意思，可以再看官方文档。改好配置就可以直接启动了。

### adapter
打开adapter包下的application.yml文件进行配置，里面有些参数我们是不需要配的，这要根据你究竟要同步到哪里来定。核心的一些配置说明如下：
```yaml
canal.conf:
  canalServerHost: 127.0.0.1:11111          # 对应单机模式下的canal server的ip:port
  zookeeperHosts: slave1:2181               # 对应集群模式下的zk地址, 如果配置了canalServerHost, 则以canalServerHost为准
  mqServers: slave1:6667 #or rocketmq       # kafka或rocketMQ地址, 与canalServerHost不能并存
  flatMessage: true                         # 扁平message开关, 是否以json字符串形式投递数据, 仅在kafka/rocketMQ模式下有效
  batchSize: 50                             # 每次获取数据的批大小, 单位为K
  syncBatchSize: 1000                       # 每次同步的批数量
  retries: 0                                # 重试次数, -1为无限重试
  timeout:                                  # 同步超时时间, 单位毫秒
  mode: tcp # kafka rocketMQ                # canal client的模式: tcp kafka rocketMQ
  srcDataSources:                           # 源数据库
    defaultDS:                              # 自定义名称
      url: jdbc:mysql://127.0.0.1:3306/mytest?useUnicode=true   # jdbc url 
      username: root                                            # jdbc 账号
      password: 121212                                          # jdbc 密码
  canalAdapters:                            # 适配器列表
  - instance: example                       # canal 实例名或者 MQ topic 名
    groups:                                 # 分组列表
    - groupId: g1                           # 分组id, 如果是MQ模式将用到该值
      outerAdapters:                        # 分组内适配器列表
      - name: logger                        # 日志打印适配器
```
目前adapter数据订阅的方式支持两种，一是通过canal server，二是订阅kafka/RocketMQ的消息。所以上面需要二选一。