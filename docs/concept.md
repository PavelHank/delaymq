# 基本概念

`v1.0`@[hank]((mailto:pavelhank@outlook.com)
---

## 1 消费模型
turtlemq 由 Producer、Broker 和 Consumer 组成。Producer 负责产生消息，Broker 负责消息状态变换及存储，Consumer 负责消费可处理消息。在业务角度而言，
Broker 就是 turtlemq 服务器，业务中的消息生产者通过 Broker 的接口，将消息发送到指定的 Topic 中。Broker 可以管理多个 Topic，Consumer 消费 topic 中的 ReadyData，每个 Topic 可以有多个消费者，每个消费者都会有一个 ID。如果多个实例使用同一个 ConsumerID，turtlemq 认为是这个 Consumer 是多实例部署的，他们将共同推荐这个 Topic 下消息的消费。

## 2 消息生产者（Producer）
负责产生消息，统一写入到 Broker 管理的 Topic 中，我们的主要责任是管理延时任务，所以生产者产生的消息对消费者而言，实时性要求很低，所以我们采用异步的方式发送给服务端。我们将每秒向服务端是批量提交一次数据，或每消息量达到 1M 之后向服务端同步一次。服务端应该对消息保存成功与否返回对应状态。为了后续对消息可管理，将为每个消息生成一个 ID，可以由业务方指定，也可留空， turtlemq 将自动为其按照 [objecID](https://www.mongodb.com/blog/post/generating-globally-unique-identifiers-for-use-with-mongodb) 格式生成消息ID。

## 3 消费者（Consumer）
消息的处理程序，一般是用来处理业务逻辑后台进程异步消费某一 Topic 下可处理的所有消息。我们采用客户端拉取的方式获得可消费消息，客户端获取数据时，一般每隔 5s 或消息数据量大于 1M 返回一次。客户端在消息消费成功后，也应该上报消费成功的状态。当 Consumer 拉取到消息后，turtlemq 将会把这批消息标记为 `reserve` 状态；当消费成功后，调用 Ack 接口上报，turtlemq 会将其状态标记为 `history`。

## 4 主题（Topic）
主题是一类消息的集合，可以理解为数据库中表的概念。每条消息只能属于一个 Topic，它是消费者订阅消息的基本单位，也是生产者写入消息的主要渠道。

## 5 队列服务（Broker）
对应一个服务器，向外提供队列的功能。turtlemq 的工作节点，负责对 Topic 数据的管理。也会管理服务的元数据，包括消费者信息、生产者信息、消费进度偏移、主题、队列消息等。

## 6 拉取式消费（Pull Consumer）
我们主要管理的延时任务，一般这种场景下都对实时性要求不是很高，所以我们不采用 推动式消费。采取拉取式消费，提供 Http 接口给客户端，这样在不同语言与服务端打交道式也更容易，不用考虑长链接的问题。