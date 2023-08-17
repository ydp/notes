---
title: nsq-quick-start
date: 2016-09-09 13:58:09
categories: nsq
tags: [nsq, 入门]
---

#### 安装

在mac上
``` bash
brew install nsq
```

* 启动管理拓扑信息并提供最终一致性的发现服务的守护进程

``` bash
nsqlookupd
```

* 启动负责接收、排队、转发消息到客户端的守护进程, 在不同的shell中

``` bash
nsqd  --lookupd-tcp-address=127.0.0.1:4160
```

* 启动web界面，在不同的shell中

``` bash
nsqadmin --lookupd-http-address=127.0.0.1:4161
```

* 发布初始消息

``` bash
curl -d 'hello world 1' 'http://127.0.0.1:4151/put?topic=test'
```
* 在另外的shell中把消息导入文件

``` bash
nsq_to_file --topic=test --output-dir=/tmp --lookupd-http-address=127.0.0.1:4161
```

* 发布消息到nsq

``` bash
$ curl -d 'hello world 2' 'http://127.0.0.1:4151/put?topic=test'
$ curl -d 'hello world 3' 'http://127.0.0.1:4151/put?topic=test'
```

* 验证
查看 nsqadmin的UI http://127.0.0.1:4171/，也可以检查 /tmp 中的log文件 test.*.log

值得注意的是客户端(nsq_to_file)不知道test主题在哪儿产生，它通过nsqlookupd 获取信息，虽然连接nsqlookupd会耗时，但连接过程中的消息不会丢失。


[原文链接](http://nsq.io/overview/quick_start.html)
