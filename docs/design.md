# 设计

`v1.0`@[hank](mailto:pavelhank@outlook.com)
---

这里描述各个部分的具体实现，其中有部分属于约定，比如 Producer 库逻辑。

## Producer
生产者发布延时消息时，需要批量提交。为了方便消息的追踪，我们将为每个消息生成一个ID，用 UUID4 算法生成。生产者提交到服务端的触发条件应该是 1000 条消息或每秒1次，数据提交成功后，服务端返回成功存入队列的数据记录数，及失败的消息ID。当然，如果客户端提交消息失败，将一直重试，占用客户端缓存区，用户新添加的延时消息会因为缓存区已满而返回错误。选择一直重试是为了阻塞新消息产生，让用户能够及时感知到错误。

## Consumer
消费者从延时队列中获取可消费记录时，使用长轮询模式，每次服务端累计 5s 或 1000 条数据返回一次。返回数据中， 除了原始消息内容外，还包括当前消息所在 ReadyData 中的偏移量。在客户端小服务端发送 Ack 确认信息时，得按顺序提交。比如服务端返回得数据如下：

```
Get /consume
<offset:20,member:0,body:...>
<offset:21,member:0,body:...>
<offset:22,member:0,body:...>
<offset:23,member:0,body:...>
<offset:24,member:0,body:...>
```

客户端也是顺序接收到这些消息，在 Ack 确认时，客户端需要有序提交，比如：

```
Post /ack 
<offset:20,member:0>
<offset:21,member:0>
<offset:22,member:0>
```

如果客户端确认了 `offset = 22,member = 0` 处理成功，将继续等待前面两条得 Ack 调用。如果一直没有调用，将重复投放 `offset = 20` 和 `offset = 21` 的两条。当这几条全部 Ack 后， Consumer 库才应该调用服务端的 Ack 接口，将这部分确认消费信息持久化到服务端。如果客户端在成功消费了这三条数据，还没有调用服务端 Ack 接口就因异常退出了，服务端将重新投放 `offset=20`、`offset=21`、`offset=22` 这几条消息。

## Broker 服务端
服务端主要管理延时数据，在队列中，每条数据均属于一个确定的 Topic, Producer 向 Topic 中写数据，Consumer 从 Topic 中读数据。由于 Topic 需要支持多个 Consumer 的支持，所以需要多个进度表记录每个 Consumer 的消费记录。对 Topic 的数据将分别管理 DelayData 和 ReadyData。所有的 Consumer 都读的是 ReadyData。

### Topic 数据管理
所有的 DelayData 和 ReadyData 均需要持久化的硬盘上。目前采用异步写入的方式持久化内存数据。所有 Producer 写入的数据先写入 CommitLog 中，然后 Index 模块消费 CommitLog 中的数据，通过最小堆计算那些数据可以将其状态改变为 ReadyData。文件持久化路径如下：

```
/{workpath}/{$topic}/commit_0000000000.log
/{workpath}/{$topic}/commit_1073741824.log
/{workpath}/{$topic}/minheap.bson
/{workpath}/{$topic}/ready_0000000000.bson
/{workpath}/{$topic}/ready_1073741824.bson
/{workpath}/{$topic}/metadata.bson
```

其中， commitLog 在成功消费后将尝试删除，而 ReadyData 按照配置保留指定的数据量（比如最近七天），如果总数据量不超过 1G，将不会触发删除逻辑。CommitLog 文件中数据存储格式：
`len:1000,created_at:{CreatedTime},delay_to:{readyAt},body:....`