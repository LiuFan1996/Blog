# kafka学习

## 核心概念

kafka是消息中间件的一种，是一种分布式流平台，适用于构建实时数据管道和流应用程序。具有横向扩展，容错，等优点

## kafka名词解释

### 消息记录（record）

由一个key，一个value和一个时间戳构成，消息最终存储在主题下的分区中，记录在生产者中陈伟生成者记录（ProducerRecord），在消费者中称为消费者记录

（ConsumerRecord），kafka集群保持所有的消息，直到他们过期，无论消息是否被消费了，在一个可配置的时间段内，kafka集群保留所有发布的消息，不管这些消息有没有被消费。比如，如果消息的保存策略设置为2天，那么在一个消息被发布的两天时间里，他都是可以被消费的，之后它将被丢弃以释放空间，kafka的性能是和数据量无关的常量级的，所以保留太多的数据并不是问题。

### 生成者（Producer）

生产者用于发送（send）消息

### 消费者（consumer）

消费者用于订阅（subscribe）消息

### 消费者组（consumer group）

相同的group id的消费者将视为同一个消费组，每个消费组都需要设置一个组id，每条消息只能被consumer group 中的一个 consumer消费，但是可以被多个consumer group消费

### 主题（topic）

消息的一种逻辑分组，用于对消息分门别类，每一类消息称之为一个主题，相同主题的消息放在一个队列中

### 分区（partition）

消息的一种物理分组，一个主题被拆成多个分区，每一个分区就是一个顺序的，不可变的消息队列，并且可以持续添加，分区中的每个消息都被分配了一个唯一的id，称之为偏移量（offset），在每个分区中偏移量都是唯一的。每个分区对应一个逻辑log，又多个segment组成。

### 偏移量（offset）

分区中的每个消息都有一个唯一的id，称之为偏移量，它代表已经消费的位置。可以自动或者手动提交偏移量（即自动或者手动控制一条消息是否已经被成功消费）

### 代理（broker）

一台kafka服务器称之为broker

### 副本（replica）

副本只是一个分区的备份。副本从不读取或写入数据，他们用于防止数据丢失

### 领导者（lender）

leader是负责给定扽去的所有读取和写入的节点。每个分区都有一个服务器充当leader，producer和consumer只跟leader交互

### 追随者（follower）

跟随领导者指令的节点被称为follower，如果领导失败，一个追随者将自动成为新的领导者。跟随者作为正常的消费者，拉去消息并更新其自己的数据存储。replica中一个角色，从leader中复制数据

### zookeeper

kafka代理是无状态的，所以使用zookeeper来维护它们的集群状态，zookerper用于管理和协调kafka代理

## kafka功能

### 发布订阅

生产者生成消息，将消息发送到kafka指定的主题队列中，也可以发送到topic中的指定分区中，消费者从kafka的指定队列中获取消息，然后来处理消息

### 流处理

将输入的topic转换数据流输出到topic

### 连接器

将数据从应用程序中导入到kafka，或者从kafka导出数据到应用程序，例如文件中数据导入到kafka，从kafka中将数据导出到文件中

## kafka中的消息模型

队列：同名的消费者组员瓜分消息

发布订阅：广播消息给多个消费者组



生成者将消息记录发送到kafka中的主题中，一个主题可以有多个分区，消息最终存储在分区中，消费者最终从主题的分区中获取消息。

## 安装和启动

下载windows安装版，解压后

### 启动zookeeper

cd kafka_2.12-2.3.0/

在正常启动zoopkeeper之前需要修改zookeeper.properties的文件内容，将其data的输出目录指定一下

dataDir=D:\kafka_2.12-2.3.0\kafka_2.12-2.3.0\zookerper_data

然后回到上级目录，启动：

bin\windows\zookeeper-server-start.bat config\zookeeper.properties



### 启动kfaka服务：

在启动前，仍然需要修改server.properties中log.dir的配置目录，

logDir=D:\kafka_2.12-2.3.0\kafka_2.12-2.3.0\serber_data

然后回到上级目录，启动：

bin\windows\kafka-server-start.bat config\server.properties

## 使用GO实现消息的发送接收

### 准备

安装依赖sarama

go get github.com/Shopify/sarama

安装sarama-cluster依赖库

go get github.com/bsm/sarama-cluster

### 生产者：

```go
package main

import (
   "fmt"
   "github.com/Shopify/sarama"
   "log"
   "os"
   "time"
)

var Address = []string{"192.168.1.114:9092"}
func main() {
   syncProducer(Address)
}

//同步消息模式
func syncProducer(address []string){
   config := sarama.NewConfig()
   config.Producer.Return.Successes = true
   config.Producer.Timeout = 5 * time.Second
   producer, err := sarama.NewSyncProducer(address, config)
   if err!=nil{
      log.Println("sarama.NewAsyncProducer err",err)
      return
   }
   defer producer.Close()

   topic := "test"
   srcValue := "sync: this is a message. index=%d"
   for i:=0;i<10 ;i++  {
      value := fmt.Sprintf(srcValue, i)
      message := &sarama.ProducerMessage{
         Topic: topic,
         Value: sarama.ByteEncoder(value),
      }
      part, offset, err := producer.SendMessage(message)
      if err != nil {
         log.Printf("send message(%s) err=%s \n", value, err)
      }else {
         fmt.Fprintf(os.Stdout, value + "发送成功，partition=%d, offset=%d \n", part, offset)
      }
      time.Sleep(2*time.Second)
   }
}
//异步消息模式
func SaramaProducer() {
   config := sarama.NewConfig()
   //等待服务器所有副本都保存成功后的响应
   config.Producer.RequiredAcks = sarama.WaitForAll
   //随机向partition发送消息
   config.Producer.Partitioner = sarama.NewRandomPartitioner
   //是否等待成功和失败后的响应，只有上面的RequireAcks设置不是NoReponse这里才有用
   config.Producer.Return.Successes = true
   config.Producer.Return.Errors = true
   //设置使用的kafka版本，如果低于V0_10_0_0版本，消息中的timestrap没有作用，需要消费和生成同时
   //注意设置版本不对的话，kafka会返回很奇怪的错误，并且无法成功发送消息
   config.Version = sarama.V2_2_0_0
   fmt.Println("start make producer")
   //使用配置,新建一个异步生产者
   producer, e := sarama.NewAsyncProducer([]string{"182.61.9.153:6667","182.61.9.154:6667","182.61.9.155:6667"}, config)
   if e != nil {
      fmt.Println(e)
      return
   }
   defer producer.AsyncClose()

   //循环判断哪个通道发送过来数据.
   fmt.Println("start goroutine")
   go func(p sarama.AsyncProducer) {
      for{
         select {
         case  <-p.Successes():
            //fmt.Println("offset: ", suc.Offset, "timestamp: ", suc.Timestamp.String(), "partitions: ", suc.Partition)
         case fail := <-p.Errors():
            fmt.Println("err: ", fail.Err)
         }
      }
   }(producer)

   var value string
   for i:=0;;i++ {
      time.Sleep(500*time.Millisecond)
      time11:=time.Now()
      value = "this is a message 0606 "+time11.Format("15:04:05")

      // 发送的消息,主题。
      // 注意：这里的msg必须得是新构建的变量，不然你会发现发送过去的消息内容都是一样的，因为批次发送消息的关系。
      msg := &sarama.ProducerMessage{
         Topic: "0606_test",
      }

      //将字符串转化为字节数组
      msg.Value = sarama.ByteEncoder(value)
      //fmt.Println(value)

      //使用通道发送
      producer.Input() <- msg
   }

}
```

### 消费者：

```go
package main

import (
   "fmt"
   "github.com/Shopify/sarama"
   "github.com/bsm/sarama-cluster"
   "log"
   "os"
   "os/signal"
   "sync"
)
var Address1 = []string{"192.168.1.114:9092"}
func main()  {
   topic := []string{"test"}
   var wg = &sync.WaitGroup{}
   wg.Add(2)
   //广播式消费：消费者1
   go clusterConsumer(wg, Address1, topic, "group-1")
   //广播式消费：消费者2
   go clusterConsumer(wg, Address1, topic, "group-2")

   wg.Wait()
}

// 支持brokers cluster的消费者
func clusterConsumer(wg *sync.WaitGroup,brokers, topics []string, groupId string)  {
   defer wg.Done()
   config := cluster.NewConfig()
   config.Consumer.Return.Errors = true
   config.Group.Return.Notifications = true
   config.Consumer.Offsets.Initial = sarama.OffsetNewest

   // init consumer
   consumer, err := cluster.NewConsumer(brokers, groupId, topics, config)
   if err != nil {
      log.Printf("%s: sarama.NewSyncProducer err, message=%s \n", groupId, err)
      return
   }
   defer consumer.Close()

   // trap SIGINT to trigger a shutdown
   signals := make(chan os.Signal, 1)
   signal.Notify(signals, os.Interrupt)

   // consume errors
   go func() {
      for err := range consumer.Errors() {
         log.Printf("%s:Error: %s\n", groupId, err.Error())
      }
   }()

   // consume notifications
   go func() {
      for ntf := range consumer.Notifications() {
         log.Printf("%s:Rebalanced: %+v \n", groupId, ntf)
      }
   }()

   // consume messages, watch signals
   var successes int
Loop:
   for {
      select {
      case msg, ok := <-consumer.Messages():
         if ok {
            fmt.Fprintf(os.Stdout, "%s:%s/%d/%d\t%s\t%s\n", groupId, msg.Topic, msg.Partition, msg.Offset, msg.Key, msg.Value)
            consumer.MarkOffset(msg, "")  // mark message as processed
            successes++
         }
      case <-signals:
         break Loop
      }
   }
   fmt.Fprintf(os.Stdout, "%s consume %d messages \n", groupId, successes)
}
```