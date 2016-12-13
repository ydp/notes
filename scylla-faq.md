---
title: ScyllaDB FAQ
date: 2016-09-08 13:12:44
categories: Scylla
tags: Scylla
---

#### 性能

###### 一个8核CPU服务器上运行scylla，cpu利用率到800%是怎么回事？

对于scylla来说，等待网络数据包的到达、或者CPU核之间交换消息都是非常耗时的操作。为了提高效率，scylla没有采用让操作系统调度这些事件的方式来处理，而是主动地对这些事件进行轮训，这就会导致scylla吃掉所有可用CPU，但提高了吞吐量、降低了延迟。注意，从scylla 0.14开始，只有在处理请求的时候CPU才会到100%，其他时候(空闲)不会。

###### 一个8核CPU上运行的scylla不应该把所有可用CPU都消耗掉？为什么我看到的明显不到800%?

有可能你的数据或日志文件不在XFS格式的分区上，想充分利用scylla，数据和日志文件都必须存放在XFS格式的分区上。

###### scylla吃掉了所有内存！！！为什么会这样？如果服务器的内存耗光了会怎么办？

Scylla把所有可用的内存都用来缓存数据了，它自身知道如何动态地管理内存以达到最优化的性能，举个栗子，如果很多客户端连到scylla上，它就会从缓存里让出一部分内存给这些客户端连接(连接消耗内存)，当请求处理结束，连接数掉下去的时候，这些内存又会还给缓存使用。

###### 可以限制scylla使用CPU和内存的数量吗？

--smp 选项可以限制scylla使用更少的CPU，比如 --smp 2，被使用CPU的利用率仍然会到100%，但空出的CPU可以让系统有一部分空闲；内存也有类似的选项，-m。

###### 我需要禁用scylla的缓存功能减少内存开销吗？

scylla的缓存功能是无法关掉的。普通的操作不会导致缓存问题，scylla会用到50%的内存作为缓存，如果已用空间变得很大，它会动态放出一部分内存。所以唯一可能发生内存溢出的情况就是，对一个scylla分片来说单行数据大小超过了50%的内存，也就是 (total_machine_memory / (number_of_lcores * 2))。

一个64G内存，16核且开启超线程的机器，每个分片有2G内存，所以每个分片的缓存可以用到1G内存，如果真的有这么大的单行数据，你一定会遇到其他的问题（不仅仅是内存问题）。我们建议单行数据的大小保持在几M就OK了。

##### scylla使用了哪些技巧来达到现在的性能？

scylla尽可能地利用所有可用资源（CPU核、内存、存储、网络），使用的方法就是所有操作都并行处理，绝不阻塞。如果scylla需要读一个磁盘块，读取操作开始后会立即返回去做其他任务，稍后，当读操作完成时，它又会回到这个任务接着进行下一个操作。采用非阻塞的方式，实现高并发，把所有资源都利用到极限。

###### 我知道scylla底层使用了seastar框架，每CPU核一个线程，但我看到每个核不止一个线程是怎么回事？

seastar对阻塞的系统调研(open()/fsync()/close())seastar为每个核创建了一个额外的线程，如此，当阻塞调用发生时，seastar框架的reactor就可以继续执行其他操作。这些线程通常都是空闲的，因此不会带来线程频繁切换的开销。

#### 特性

###### 下个release，scylla会有什么新功能？

可以从这里查到： http://www.scylladb.com/technology/status/

###### scylla和cassandra兼容性如何？

参考：http://www.scylladb.com/technology/status/#cassandra-compatibility

###### scylla 以后对Cassandra的兼容策略如何？

参考：http://www.scylladb.com/technology/status/#cassandra-compatibility

我们会支持2.2和3.0的feature，包括materialized view.

3.0以后，我们会评估新的feature，看是否合适scylla，但总的目标是保持兼容。

我们还打算引入一些扩展和增强功能，和Cassandra社区一起开发新feature，因此我们不会打破Cassandra已经建立起来的(NOSQL)生态，支持在同样的驱动和应用程序代码下可以来回切换这两套(NOSQL)方案。


#### UBUNTU

###### 检查更新UBUNTU 14.04内核

检查内核
``` bash
uname -a
```
如果内核版本低于3.15，执行:
``` bash
sudo apt-cache search linux-image
sudo apt-get install linux-image-your_version_choice
sudo reboot now
```
先检查可用的内核镜像，然后安装，重启


#### 安装

###### scylla可以安装在一个Cassandra服务器上吗？

scylla有自己的Cassandra客户端工具，在scylla-tools包里，在一个已经安装Cassandra的服务器上安装它可能会导致下面的问题：

``` bash
Unpacking scylla-tools (1.0.1-20160411.b9fe89b-ubuntu1) ...
dpkg: error processing archive /var/cache/apt/archives/scylla-tools_1.0.1-20160411.b9fe89b-ubuntu1_all.deb (--unpack):
trying to overwrite '/usr/bin/nodetool', which is also in package cassandra 2.1.4
```
我们建议在安装scylla-tools前先卸载Cassandra。


#### 帮助

哪儿可以问问题？

邮件列表：
* [scylla开发组](https://groups.google.com/forum/#!forum/scylladb-dev)：开发scylla
* [scylla用户组](https://groups.google.com/forum/#!forum/scylladb-users)：使用scylla和开发客户端



原文链接： [scylla faq](http://www.scylladb.com/doc/faq/)
