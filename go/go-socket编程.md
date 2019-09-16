# socker编程

socker编程是网络常用的编程，我们通过在网路中创建socker关键字来实现网络间的通信

### socket大致介绍

​    socket编程时一门技术，它主要时在网络通信中进场用到

​    既然是一门技术，由于现在是面向对象的编程，一些计算机行业的大神通过抽象的理念，在现实中通过反复的理论或者实际的推导，提出了抽象的一些通信协议，基于tcp/ip协议，提出大致的构想，一些泛型的程序大牛在这个协议的基础上，将这些抽象化的理念接口化，针对协议提出的每个理念，专门的编写制定的接口，与其协议一一对应，形成了现在的socket标准规范，然后将其接口封装成可以调用的接口，供开发者使用

​     目前，开发者开发出了很多封装的类来完善socket编程，都是更加方便的实现刚开始socket通信的各个环节，所以我们首先必须了解socket的通信原理，只有从本质上理解socket的通信，才可能快速方便的理解socket的各个环节，才能从底层上真正的把握

### TCP/IP协议

要理解socket必须得理解tcp/ip。

TCP/IP协议不同于OSI模型得7个分层，它是根据这7个分层，将其重新划分。

原本得OSI模型重上到下为：应用层，表示层，会话层，传输层网络层，数据链路层，物理层

TCP/IP协议参考模型把所有得TCP/IP协议类规到四个抽象层中

应用层：TFTP，HTTP，SNMP，FTP，SMTP，DNS，Telnet等待

传输层：TCP，UDP

网络层：IP，ICMP，OSPF，EIGRP，IGMP

数据链路层：SLIP，CSLIP，PPP，MTU

每一抽象层建立在低一层提供的服务上，并为上一层提供服务

### 理解socket

我们可以发现socket就应用程序得传输层与应用层之间，设计了一个socket抽象层.

tcp连接得三次握手：

​	第一次握手：客户端尝试连接服务器，向服务器发送syn包，syn=j，客户端进入SYN_SEND状态等待服务器确认

​	第二次握手：服务器结束客户端syn包并确认（ack=j+1），同时向客户端发送一个SYN包（syn=k），即SYN+ACK包，此时服务器进入SYN_RECV状态

​	第三次握手：客户端收到服务器得SYN+ACK包，向服务器发送确认包ACK（ack=k+1）,此包发送完毕，客户端和服务器端进入ESTABLISHED状态，第三次握手完成

![1568607988734](C:\Users\PC-003\AppData\Roaming\Typora\typora-user-images\1568607988734.png)

根据tcp得三次握手，socket也定义了三次握手

![1568607977355](C:\Users\PC-003\AppData\Roaming\Typora\typora-user-images\1568607977355.png)

### socket接口函数原理

socket三次握手详解：
	第一次握手：客户端发送一个syn j包，试着去链接服务器端，于是客户端需要一个提供一个链接函数

​	第二次握手：服务器端需要接受客户端发送来得syn j+1包，然后再发送ack包，所以我们需要又服务器端得接收处理函数

​	第三次握手，客户端得处理函数和服务器端得处理函数

三次握手只是一个数据传输得过程，但是，我们传输器需要一些准备过程，比如创建一个套接字，手机一些计算机得资源，将一些资源绑定套接字里面，以及接收和发送数据得函数。

TCP Server               TCP Client

socket()			socket()

bind()			    connnect()

listen()			    send()

accept()			   close()

recv()

close()

服务器端得工作流程为：创建一个socket，通过bind绑定socket和端口号，listen监听端口，accept接收来自客户端得链接请求，recv从socket读取数据，close关闭socket

客户端得工作流程为：传教socket，链接指定ip得指定端口，也就是服务器，send向 socket中写入信息，close关闭socket。

### go-socket案例

Socket_Server 

```go
package main

import (
   "fmt"
   "net"
)

func main() {
    // net包中封装好的接口，内部包括了 新建一个socket，绑定端口等操作
   listen, err := net.Listen("tcp", "127.0.0.1:8000")
   if err != nil{
      fmt.Println("socket_server is final",err)
      return
   }
	//默认程序结束时关闭链接
   defer  listen.Close()

   for{
       //从监听中接收一个消息
      conn, err := listen.Accept()
      if err!= nil{
         fmt.Println("socket_server connect is err:",err)
         continue
      }
       //如果获取成功，开启一个协程处理这个数据
      go func(conn net.Conn) {
         defer conn.Close()
          // 获取链接地址
         ipAddr := conn.RemoteAddr().String()
         fmt.Println("链接成功IP：",ipAddr)

         buf := make([]byte,1024)
         n, err := conn.Read(buf)
         if err != nil{
            fmt.Println("socket_server read is err:",err)
            return
         }
         bytes := buf[:n]
         fmt.Println("服务端接收IP：",ipAddr," 数据为：",string(bytes))

         conn.Write([]byte("success"))
      }(conn)
   }
}
```

Socket_Client

```go
package main

import (
   "fmt"
   "net"
)

func main(){
	//链接服务器
   conn, err := net.Dial("tcp", "127.0.0.1:8000")
   if err != nil{
      fmt.Println("socket_client conenct server is err：",err)
      return
   }
	//结束时关闭链接
   defer  conn.Close()

   buf := make([]byte,1024)
    //循环发送
   for{
      fmt.Println("请输入：")
      fmt.Scan(&buf)
      fmt.Println("发送的内容为："+string(buf))

      conn.Write(buf)

      n, err := conn.Read(buf)
      if err!=nil {
         fmt.Println("socket_client write buf to server ,result is err :",err)
         return
      }

      result := buf[:n]
      fmt.Println("服务器回执为：",n,string(result))
   }
}
```