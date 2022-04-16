# Go整合rabbitmq

### 安装

```
docker search rabbitmq // 查找对应的mq版本
```

![](<../.gitbook/assets/image (11).png>)

这里我推荐选择官方的。

```
docker pull rabbitmq
```

默认rabbitmq镜像是开始web管理插件，我们需要进去docker镜像中手动开启，密码也是这个时候设置的。

```
./rabbitmq-plugins enable rabbitmq_management
```

docker运行rabbitmq

```
docker run --name rabbitmq -d -p 15672:15672 -p 5672:5672 rabbitmq
```

输入http://127.0.0.1:15672 账号密码进入管理界面。强的大佬可以通过命令行直接操作，图方便也可以直接界面操作都可。

![](<../.gitbook/assets/image (14).png>)

### 代码整合

完成第一步，我们进行代码整合，由于mq和es有所不同，我们将通过一个test发送消息，再通过一个消费者来消费。

#### 生产消费者搭建

下方为simple模式下的代码逻辑，PublishSimple为发送信息，ConsumeSimple为消费信息。

```go
package common

import (
	"fmt"
	"github.com/streadway/amqp"
	"log"
)

const MQURL = "amqp://penscan:penscan@127.0.0.1:5672/scanner_v2"

// RabbitMQ rabbitMQ结构体
type RabbitMQ struct {
	conn    *amqp.Connection
	channel *amqp.Channel
	//队列名称
	QueueName string
	//交换机名称
	Exchange string
	//bind Key 名称
	Key string
	//连接信息
	Mqurl string
}

// NewRabbitMQ 创建结构体实例
func NewRabbitMQ(queueName string, exchange string, key string) *RabbitMQ {
	return &RabbitMQ{QueueName: queueName, Exchange: exchange, Key: key, Mqurl: MQURL}
}

// Destory 断开channel 和 connection
func (r *RabbitMQ) Destory() {
	r.channel.Close()
	r.conn.Close()
}

// failOnErr 错误处理函数
func (r *RabbitMQ) failOnErr(err error, message string) {
	if err != nil {
		log.Fatalf("%s:%s", message, err)
		panic(fmt.Sprintf("%s:%s", message, err))
	}
}

// NewRabbitMQSimple 创建简单模式下RabbitMQ实例
func NewRabbitMQSimple(queueName string) *RabbitMQ {
	//创建RabbitMQ实例
	rabbitmq := NewRabbitMQ(queueName, "", "")
	var err error
	//获取connection
	rabbitmq.conn, err = amqp.Dial(rabbitmq.Mqurl)
	rabbitmq.failOnErr(err, "failed to connect rabb"+
		"itmq!")
	//获取channel
	rabbitmq.channel, err = rabbitmq.conn.Channel()
	rabbitmq.failOnErr(err, "failed to open a channel")
	return rabbitmq
}

// PublishSimple 直接模式队列生产
func (r *RabbitMQ) PublishSimple(message string) {
	//1.申请队列，如果队列不存在会自动创建，存在则跳过创建
	_, err := r.channel.QueueDeclare(
		r.QueueName,
		//是否持久化
		false,
		//是否自动删除
		false,
		//是否具有排他性
		false,
		//是否阻塞处理
		false,
		//额外的属性
		nil,
	)
	if err != nil {
		fmt.Println(err)
	}
	//调用channel 发送消息到队列中
	r.channel.Publish(
		r.Exchange,
		r.QueueName,
		//如果为true，根据自身exchange类型和routekey规则无法找到符合条件的队列会把消息返还给发送者
		false,
		//如果为true，当exchange发送消息到队列后发现队列上没有消费者，则会把消息返还给发送者
		false,
		amqp.Publishing{
			ContentType: "text/plain",
			Body:        []byte(message),
		})
}

// ConsumeSimple simple 模式下消费者
func (r *RabbitMQ) ConsumeSimple() {
	//1.申请队列，如果队列不存在会自动创建，存在则跳过创建
	q, err := r.channel.QueueDeclare(
		r.QueueName,
		//是否持久化
		false,
		//是否自动删除
		false,
		//是否具有排他性
		false,
		//是否阻塞处理
		false,
		//额外的属性
		nil,
	)
	if err != nil {
		fmt.Println(err)
	}

	//接收消息
	msgs, err := r.channel.Consume(
		q.Name, // queue
		//用来区分多个消费者
		"", // consumer
		//是否自动应答
		true, // auto-ack
		//是否独有
		false, // exclusive
		//设置为true，表示 不能将同一个Conenction中生产者发送的消息传递给这个Connection中 的消费者
		false, // no-local
		//列是否阻塞
		false, // no-wait
		nil,   // args
	)
	if err != nil {
		fmt.Println(err)
	}

	forever := make(chan bool)
	//启用协程处理消息
	go func() {
		for d := range msgs {
			//消息逻辑处理，可以自行设计逻辑
			log.Printf("Received a message: %s", d.Body)

		}
	}()

	log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
	<-forever

}

```

#### 验证代码

test.go

```go
package main

import (
	"es-demo/common"
	"fmt"
	"testing"
)

func TestSend(t *testing.T) {
	rabbitmq := common.NewRabbitMQSimple("test")
	rabbitmq.PublishSimple("Hello test111!") // 发送消息到mq中，等待被消费
	fmt.Println("发送成功！")
}

```

main.go

开始mq对象，开启消费监听。

```go
rabbitmq := common.NewRabbitMQSimple("test")
rabbitmq.ConsumeSimple()
```

![](<../.gitbook/assets/image (9).png>)

![](<../.gitbook/assets/image (2).png>)

到此整合完成！

剩下的模式等实战遇到了，我们再进行操作。
