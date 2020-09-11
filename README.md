# turtlemq
开源的高并发延时消息队列（Delay message queue）, 提供消息持久化、高可用等特性

# Why
延时队列在工作中经常需要用到，比如用户注册后，如果三天内没有第二次登陆，则发一份唤回邮件；再比如下订单半小时如果没退款，需要将订单关闭，释放商品给其他用户购买。

查阅了一些现有对延时队列的实现方案，基本借助于 redis 的有序集合，以消息被执行时间作为 score，消息内容作为内容存储在 redis 中，redis 提供排序功能，轮询的检查有序集合中最前面的那条任务是否到达处理时间，以达到延时功能，但这对于延时消息数据巨大的应用来说，很考验内存。

turtlemq 项目受到 kafka 的启发，希望能够合理利用巨大的磁盘空间，提供延时队列的功能。

# 设计
turtlemq 将由 Producer/Consumer、 Index 模块、Broker 模块、Server 模块组成，Producer/Consumer 负责延时消息的产生和消费，Index 和 Broker 负责管理数据，Server 负责暴露服务，将 delaymq 的服务能力。
为了方便描述，我们在这里统一几个关键词：
- topic: 对延时消息的归类，生产者写指定的 topic 写入数据，服务端以 topic 维度管理数据，消费者只能处理 topic 中 readyData；
- sleepData: 延时消息，添加到了队列中，还没到指定处理时间的数据，比如一条消息指定在时间节点 `time1` 之后再处理，在 `time1` 之前的时间内，称它为 sleepData；
- readyData: 就绪数据，可以通知消费者；
- reserveData: 已消费、未确认消费成功的数据，如果一定时间内未确认成功消费，将重新变成 readyData，可再次消费；
- historyData: 历史数据，已成功消费的数据，只有这部分数据才会定期删除；

## Producer
延时消息的产生者，每个消费者只能向一个 topic 写入数据，但一个 topic 可接受多个 Producer 的写入请求；由于消息是延时的，Producer 应该定时批量提交数据，尽量避免单条数据提交；

## Consumer
readyData 的消费者，可以有多个消费者批量消费同一个 topic 下的数据。因为所有的 readyData 均可消费，所以消费者也是通过 API 批量获取数据。已处理消息的 ack 信号分组确认，每次提供一组消息ID及成功与否的标识，如：`["MsgID1","-MsgID2"]`，MsgID1 成功处理，MsgID2 处理失败。

## Index 模块
负责延时数据的管理，所有 Producer 产生的消息，将直接存储到 Index 模块，Index 内部通过排序，将 readyData 转给 Broker ，内部通过最小堆或环形计时器检查消息是否可以从 `sleep` 状态转变为 `ready`。将会把所有的消息和索引持久化到文件系统，同时需要实现文件合并、释放磁盘空间的逻辑，当某个文件内的延时消息超过 90% 已转移给 Broker 的文件，将会把其他 sleepData 重新 Index 一次，然后将当前文件的磁盘空间释放。索引数据结构采用 BTree+？

## Broker 模块
负责管理 readyData、reserveData 及 historyData，需要在内部维护删除 historyData 的逻辑。 

## Server 模块
提供客户端访问的接口，包括添加延时消息、消费消息、确认消费成功等接口

## 数据持久化存储
数据持久化不依赖于第三方存储软件，
延时消息,文件名是文件 1G=1073741824  
有且仅有每个文件中的消息全部被转移到 ready 队列中之后才可以清除该部分数据
清除方式，当每个文件中的内容少于一定比例后（比如 <=5%），将追加到后面文件中，为了释放磁盘空间。
索引不做更新，如果根据索引检索数据转移去 ready queue 时，如果主文件不存在，忽略错误，我们认为它是转移到新的 delay 文件中
 /hank/data/topic/{$name}/delay_0000000000.bson
 /hank/data/topic/{$name}/delay_1073741824.bson

已到处理时间的消息
超过一定时间的消息将被删除，从 delay queue 到 ready queue，落盘需要定期刷新，比如 5s 刷一次
 /hank/data/topic/{$name}/ready_0000000000.bson
 /hank/data/topic/{$name}/ready_1073741824.bson

延时消息的索引，用以计算哪些消息可正常消费了
在这个索引中，也记录了哪些消息已经转移到 ready 队列中了
 /hank/data/topic/{$name}/delay.index

偏移量，记录各个消费者对消息的消费进度，及最后更改时间
 /hank/data/topic/{$name}/offset



高可用：
 1. 数据全量备份
 2. 服务器主备
 3. 实现 name server，集中存储 offset 信息，主备服务器信息，健康检查
