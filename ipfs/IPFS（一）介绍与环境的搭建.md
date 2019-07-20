# IPFS（一）介绍与环境的搭建



###  1：What is IPFS

星际文件系统(InterPlanetary File System). IPFS 是一个分布式的web, 点到点超媒体协议. 可以让我们的互联网速度更快, 更加安全, 并且更加开放

ps：这是官方的解释

在我看来IPFS就是一个分布式文件系统



### 2：为什么会有IPFS

众所周知, 互联网是建立在HTTP协议上的. HTTP协议是个伟大的发明, 让我们的互联网得以快速发展.但是互联网发展到了今天HTTP逐渐出来了不足.



**HTTP的中心化是低效的, 并且成本很高.**

使用HTTP协议每次需要从中心化的服务器下载完整的文件(网页, 视频, 图片等), 速度慢, 效率低. 如果改用P2P的方式下载, 可以节省近60%的带宽. P2P将文件分割为小的块, 从多个服务器同时下载, 速度非常快.



**Web文件经常被删除**

回想一下是不是经常你收藏的某个页面, 在使用的时候浏览器返回404(无法找到页面), http的页面平均生存周期大约只有100天. Web文件经常被删除(由于存储成本太高), 无法永久保存. IPFS提供了文件的历史版本回溯功能(就像git版本控制工具一样), 可以很容易的查看文件的历史版本, 数据可以得到永久保存



**中心化限制了web的成长**

我们的现有互联网是一个高度中心化的网络. 互联网是人类的伟大发明, 也是科技创新的加速器. 各种管制将对这互联网的功能造成威胁, 例如: 互联网封锁, 管制, 监控等等. 这些都源于互联网的中心化.而分布式的IPFS可以克服这些web的缺点.



**现在的互联网应用高度依赖互联网主干网**

主干网受制于诸多因素的影响, 战争, 自然灾害, 互联网管制, 中心化服务器宕机等等, 都可能是我们的互联网应用中断服务. IPFS可以是互联网应用极大的降低互联网应用对主干网的依赖.



### 3：下载最新的IPFS

下载go-ipfs

##### 3.1：官网下载

https://dist.ipfs.io/#go-ipfs

ps：这个网址打开以后会出现很多的下载选项，我们需要下载对应的go-ipfs，会自动识别对应系统的下载链接

![1545638831518](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545638831518.png)

##### 3.2：网盘下载

链接: https://pan.baidu.com/s/1SVe0zEY_x4cduDOAdO2MoQ 提取码: 7dtz



### 4：安装IPFS

1）将下载好的ipfs文件夹移动的创建好的文件夹解压，进入解压后的文件夹，目录结构如下：

![1545639059457](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545639059457.png)



2）运行install.sh

![1545639170869](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545639170869.png)



3)   测试安装是否成功

![1545639243436](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545639243436.png)



4）查看命令帮助

![1545639290200](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545639290200.png)

ps:在安装执行./install.sh之后可能会出现执行ipfs version 或ipfs help 无法找到命令的错误，这个时候需要全局代理翻墙，翻墙后重新执行./install.sh



### 5：初始化配置信息与启动守护进程

1）初始化ipfs

在命令行执行ipfs init

![1545639566631](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545639566631.png)

ps：初始化时会默认初始化在你的用户目录下新建一个.ipfs

​	会生成你的节点id：/QmS4ustL54uo8FzR9455qaxZwuMiUhyvMcX9Ba8nUH4uVv ps:这是这是我生成的id，每个人都是不一样的



2）查看安装信息

![1545640064164](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545640064164.png)



3）启动守护进程

​	ipfs daemon

![1545640180914](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545640180914.png)

这条命令是启动一条守护进程运行你的ipfs节点

可以通过ipfs swarm peers来查看链接的节点



### 6：上传下载

ipfs提供了两种方式对文件进行操作

1）weiui 方式

浏览器输入localhost:5001/webui进入浏览器文件操作页面这里不做演示傻瓜式操作

ps：在节点加载页面特别的耗费资源，电脑可能会产生卡顿，关掉就好了



2）命令行方式

我在F:盘新建了一个1.txt内容为：123456789

上传1.txt

ipfs add 1.txt

![1545641283086](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545641283086.png)

这个 QmbbHQPfRcXmZMgwFbu8wiaA1oG3NRQcni7zDQXbuvVaXB就是文件的hash值，通过它可以找到这个文件

通过hash查看1.txt

ipfs cat  QmbbHQPfRcXmZMgwFbu8wiaA1oG3NRQcni7zDQXbuvVaXB

![1545641339705](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545641339705.png)



下载1.txt

ipfs get QmbbHQPfRcXmZMgwFbu8wiaA1oG3NRQcni7zDQXbuvVaXB

![1545641442107](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545641442107.png)

ps：下载到当前目录下，且文件名为默认为hash值