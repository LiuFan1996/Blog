# Docker学习笔记

## Docker：镜像管理

### 搜索

搜索    Docker hub中的镜像

格式：docker search [镜像名称]

docker search ubuntu

docker search centos

### 获取

获取DockerHub中的镜像到本地

格式：docker pull [镜像名称]

docker pull ubuntu

### 查看

格式：docker images [镜像名称，可选]

​	   docker images is  [镜像名称，可选]

​	  没有镜像名称则默认查看本地所有的镜像

### 镜像重命名

重命名本地镜像名称和版本

格式：docker tag [老镜像名称]:[老镜像版本] 空格 [新镜像名称]:[新镜像版本]

docker tag  nginx:latest  testNginx:latest

PS:重命名没有新建一个镜像，相当于给同一个镜像创建了一个新的入口...指向的镜像还是同一个

### 删除镜像

格式：docker rmi [命令参数] [镜像id]

​	   docker rmi  [命令参数] [镜像名称]:[镜像版本]

​           docker rmi  [命令参数] [镜像]

命令参数：

​	简写-f 全称 --force 强制删除

实例：docker rmi testnginx:latest

### 导出镜像

将本地一个或多个镜像打包保存成本地的tar文件

格式：docker save [命令参数] [导出镜像名称] [本地镜像名称]

命令参数：-o --output 指定写入的文件名个路径

docker save -o nginx.tar nginx

### 导入镜像

将save命令打包的镜像导入本地镜像库中

格式：docker load [命令参数] [被导入镜像压缩文件的名称]

​	   docker load <  [被导入镜像压缩文件的名称]

​	   docker load --input[被导入镜像压缩文件的名称]

命令参数 ：-i --input 指定要导入的文件名称，如果没有指定，默认的是STDIN

### 历史

查看本地一个镜像的历史信息

格式：docker history [镜像名称]:[镜像版本]

​	    docker history [镜像id]

docker history ubuntu:latest

### 查看镜像详细信息

格式：docker image inspect [命令参数] [镜像名称]:[镜像版本]

​	   docker inspect [命令参数] [镜像id]

docker inspect nginx

### 根据模板创建镜像

格式:cat 模板文件名.tar | docker import -[自定义镜像名]



## Docker：容器管理

容器（Container）：容器事一种轻量级的，可移植，并将应用程序进行打包的技术，使得应用程序可以在几乎任何地方一相同的方式运行

Docker将镜像文件运行起来后，产生的对象就是容器，容器相当与是镜像运行起来的一个实例

容器具备一定的生命周期

另外，可以借助docker ps 命令查看运行的容器，如同在linux上利用ps命令查看运行着的进程那样

我们就可以理解容器就是被封装起来的进程操作，只不过现在的进程可以简单也可以复杂，复杂的话运行一个操作系统，简单的可以运行一个回显字符串。

### 容器与虚拟机的相同点

容器和虚拟机一样，都会对物理硬件资源进行共享使用。

容器和虚拟机的生命周期比较相似（创建，运行，暂停，关闭）

容器中或虚拟机中都可以安装各种应用，如redis，mysql，nginx等，也就是说，在容器中的操作，如同在一个虚拟机的虚拟操作系统中操作一样

### 容器与虚拟机的不同

注意：容器并不是虚拟机，虽然他们有很多相似的地方

虚拟的创建，启动和关闭都是基于一个完整的操作系统，一个虚拟机就是一个完整的操作系统，而容器直接运行在宿主机的内核上，其本质是一系列进程的结合

容器是轻量级的，虚拟机是重量级的

首先容器不需要额外的资源来管理，虚拟机额外更多的性能消耗

其次创建，启动或关闭容器，如同创建，启动或者关闭进程那么轻松，而创建，启动，关闭一个操作系统就没那么方便

因此，意味着在给定的硬件上能运行更多数量的容器，甚至可以直接把docker运行在虚拟机上

### 查看

格式：docker ps 

管理docker容器可以通过名称也可以通过ID

ps 是显示当前正在运行的容器 -a是显示运行过的所有容器，包括已经不运行的容器

### 创建

利用镜像创建出一个Created状态的待启动容器

格式：docker create [命令参数1] 依赖镜像 [容器内命令] [命令参数2]

命令参数1：

​	-t --tty 分配一个伪TTY,也就是分配虚拟终端

​	-i --interactive 即使没有连接,也要保持STDIN打开

​	--name 为容器取名，如果没有指定将会随机产生一个名称

命令参数2：

​	COMMAND 表示容器启动后，需要在容器中执行的命令，如ps，ls等

​	ARG 表示执行COMMAND是需要提供的一下参数，如ps命令的aux，ls命令的-a等

docker create -it --name ubuntu-1 ubuntu ls -a

可以通过执行上述命令给出的sha256的hash 使用docker start xxxx 启动上述容器

### 启动

启动容器有三种方式

1：启动待启动或已关闭容器

2：基于镜像新建一个容器并启动

3：守护进程方式i启动docker

#### 启动容器：

将一个或多个容器处于创建状态或关闭状态的容器启动起来

格式: docker start [容器名称]或[容器id]

命令参数

-a  -- attach 将当前shell的STDOUT/STDERR连接到容器上

-i   --interactive 将当前shell的STDIN连接到容器上

启动上面创建的容器

docker start -a ubuntu-1 

#### 创建新容器并启动

利用镜像创建并启动一个容器

格式：docker run [命令参数] [镜像名称] [执行的命令]

命令参数

​	-t --tty 分配一个TTY，也就是分配虚拟终端

​	-i -interactive 即使没有连接，也要保持STDIN打开

​	--name 为容器起名

​	-d --detach 在后台运行容器并打印出容器id

​	--rm 当前容器退出运行后，自动删除容器	

启动一个镜像输出内容并删除容器

docker run --rm --name naginx1 nginx /bin/echo 'hello world'

docker run 其实 是两个命令的集合体 docker create + docker start 

#### 守护进程方式启动容器

格式：docker run -d [image_name] command ...

docker run -d nginx

### 暂停

暂停一个或多个处于运行状态的容器

格式：docker pause [容器名称]或[容器id]

docker pause 9690aba0bf05 

### 取消暂停

取消一个或多个处于暂停状态的容器，恢复运行

格式：docker unpause [容器名称]或[容器id]

docker unpause 9690aba0bf05 

### 重启

重启一个或多个处于运行状态，暂停状态，关闭状态或者新建状态的容器

该命令相当与stop和start命令的结合

格式：docker restart [容器名称]或[容器id]

命令参数

​	-t --time 重启前等待的时间，单位秒

docker restart 9690aba0bf05 

### 关闭

延迟关闭一个或多个处于暂停或者运行状态的容器

格式：docker stop  [容器名称]或[容器id]

docker stop  9690aba0bf05 

### 终止

强制并立即关闭一个或多个处于暂停或运行状态的容器

格式：docker kill  [容器名称]或[容器id]

docker kill  9690aba0bf05 

### 删除

删除容器有三种方法：

正常删除--删除已关闭的

强制删除--上出正在运行的

强制批量删除--删除全部容器

#### 正常删除容器

格式：docker rm  [容器名称]或[容器id]

docker rm  9690aba0bf05 

#### 强制删除运行容器

格式：docker rm -f [容器名称]或[容器id]	

docker start -d 9690aba0bf05 

docker rm -f  9690aba0bf05 

#### 批量关闭容器

格式：docker rm -f $(docker ps -a -q)

### 进入，退出容器

#### 创建并进入容器

格式：docker run --name [容器名称] -it [镜像名称] /bin/bash

docker run -it --name ubuntu2 ubuntu /bin/bash

#### 退出容器

exit

ctrl+D

#### 手动进入容器

格式：docker exec -it 容器id /bin/bash

docker exec -it 9690aba0bf05 /bin/bash 

#### 生产方式进入容器

定义脚本

```bash
#!/bin/bash 

\#定义进入仓库函数 

docker_in(){ 

NAME_ID=$1 

PID=$(docker inspect --format {{.State.Pid}} $NAME_ID) 

nsenter --target $PID --mount --uts --ipc --net --pid 

}

docker_in $1 
```

直接执行没有权限，需要赋值权限

chmod +x docker_in.sh

./docker_in.sh 9690aba0bf05 

### 基于容器创建镜像



### 查看日志

格式：docker logs [容器id]

docker logs 9690aba0bf05 

### 查看容器端口信息

格式：docker port [容器id]

docker port 9690aba0bf05 

### 容器重命名

格式：docker rename[容器id]

docker rename 9690aba0bf05 ubuntu3

## Docker数据管理

生产环境使用Docker的过程中，往往需要对数据进行持久化保存，或者需要更多容器之间进行数据共享，那我们需要怎么样的操作呢？

答案就是：数据卷和数据卷容器

### 数据卷

数据卷就是将宿主机的某个目录，映射到容器中，作为数据存储的目录，我们就可以在宿主机对对数据进行存储

数据卷：容器内数据直接映射到本地主机环境

#### 数据卷特性

1 数据卷可以在容器之间共享和重用，本地与容器间传递数据更高效；

2 对数据卷的修改会立马有效，容器与本地目录均可；

3 对数据卷的更新，不会影响镜像，对数据与应用进行了解耦操作；

4 卷会一直存在，直到没有容器使用

命令格式：docker run -v/--volume 

在docker run 创建容器时添加一个-v参数，就可以创建并挂载一个到多个数据卷到当前运行的容器中，-v参数的作用时将宿主机的一个目录作为容器的数据卷挂载到docker容器中，使宿主机和容器之间可以共享一个目录，如果本地路径不存在，docker也会自动创建

### 数据卷实战

#### 数据卷之目录

命令格式：docker run -itd --name [容器名字] -v [宿主机目录]:[容器目录] [镜像名称] [命令(可选)] 



#### 数据卷之文件