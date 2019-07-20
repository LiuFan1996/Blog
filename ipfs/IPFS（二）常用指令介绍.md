# IPFS （二）常用指令介绍

### 1、基本命令

|    命令    |           介绍           |
| :--------: | :----------------------: |
| add <path> |    添加一个文件到IPFS    |
| cat <ref>  |   预览文件内容在控制台   |
| get <ref>  |       下载获取文件       |
|  ls <ref>  |   从一个对象中列出链接   |
| refs <ref> | 从一个对象中列出链接hash |
|    init    |    初始化IPFS本地配置    |

##### 1.1 ipfs add

1）先创建一个2.txt的文件 vi 2.txt 内容为123456789987654321 

​      使用ipfs add <path> 也就是文件路径将文件上传到ipfs

![1545792948125](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545792948125.png                        )

2）新建一个文件ipfs-add-dir 在文件夹中创建文件3.txt

​      使用 ipfs add -r ipfs-add-dir 递归上传目录和目录下所有文件

****![1545793197962](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545793197962.png                                 )    

3）创建一个隐藏的文件夹 ./list 在ipfs-add-dir 下

​	-r：递归上传文件目录

​	-w：用目录对象包裹文件

​	-H：上传隐藏的文件或文件夹

![1545793859372](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545793859372.png)

##### 1.2 ipfs cat

选项有两个

-o int显示时去掉前面的int个字节

-l int 总共显示int个字节

用来查看ipfs中存储的文件内容

例如我们查看之前上传的3.txt （注意：不能直接查看文件夹）

![1545794812919](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545794812919.png)



##### 1.3 ipfs get

选项有四个

-o path本地保存路径

-a 保存为.tar格式的压缩包

-C保存为.gzip格式的压缩包

-l int 指定压缩等级

1）使用get下载存储在ipfs中的文件例如3.txt（注意：下载默认位置是当前路径，默认文件名是文件的hash）

![1545794987613](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545794987613.png)

2）下载文件夹也是一样例如下载之前的ipfs-add-dir 这个时候下载的默认文件夹名也是hash

![1545795316126](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545795316126.png)

3）使用ipfs get <ref> -o 指定文件名or文件夹名

![1545795495230](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545795495230.png)

![1545795542294](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545795542294.png)



##### 1.4 ipfs ls

-v 在输出结果里面添加一个表头

1）ipfs pin ls 列出当前节点的所有文件

![1545796431586](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545796431586.png)

2）ipfs  ls <ref>列出当前目下的所有内容

![1545796470010](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545796470010.png)

##### 1.5 ipfs refs

refs命令用于列出某个文件的相关分片。格式如下：

ipfs refs  [选项] 文件hash

选项有四个

--format  指定输出格式，默认为只输出各分片

-e 输出格式为源文件->分片的格式

-u输出结果去重

-r 将子节点的分片也列出



### 2、数据结构命令

|  命令  |                  介绍                  |
| :----: | :------------------------------------: |
| block  |        与数据存储中的原始块交互        |
| object |           与原始DAG节点交互            |
| files  | 将对象抽象成uinx文件系统，并与对象交互 |
|  dag   |             与IPLD文件交互             |

##### 2.1 ipfs block

1）ipfs block get    

  获取原始ipfs块信息



2）ipfs block put <data> 

   把输入作为一个ipfs块

![1545799654397](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545799654397.png)

3）ipfs block stat <key> 

  打印ipfs 块统计信息

![1545799636606](C:\Users\liufan\AppData\Roaming\Typora\typora-user-images\1545799636606.png)

##### 2.2 ipfs object

##### 2.3 ipfs files

##### 2.4 IPfs dag



### 3、高级命令

|   命令    |           介绍           |
| :-------: | :----------------------: |
|  daemon   |  开启一个节点的守护进程  |
|   mount   |  挂载一个IPFS只读挂载点  |
|  resolve  |    解析任何类型的名字    |
|   name    |    发布并解析IPNS名字    |
|    key    | 创建并列出IPNS名字密钥对 |
|    dns    |       解析DNS链接        |
|    pin    |   将对象锁定到本地仓库   |
|   repo    |       操作IPFS仓库       |
|   stats   |       各种操作状态       |
| filestore |       管理文件仓库       |

##### 3.1 ipfs daemon

##### 3.2 ipfs mount

##### 3.3 ipfs resove

##### 3.4 ipfs name

##### 3.5 ipfs key

##### 3.6 ipfs dns

##### 3.7 ipfs pin

##### 3.8 ipfs repo

##### 3.9 ipfs stats

##### 3.10 ipfs filestore

### 4、网络命令

|   命令    |              介绍              |
| :-------: | :----------------------------: |
|    id     |        展示IPFS节点信息        |
| bootstrap |       添加或删除引导节点       |
|   swarm   |        管理p2p网络连接         |
|    dht    | 请求有关值或节点的分布式hash表 |
|   ping    |       测量一个链接的延迟       |
|   diag    |          打印诊断信息          |

##### 4.1 ipfs id

##### 4.2 ipfs bootstrap

##### 4.3 ipfs swarm

##### 4.4 ipfs dht

##### 4.5 ipfs ping

##### 4.6 ipfs diag

### 5、工具命令

​	

|   命令   |            介绍             |
| :------: | :-------------------------: |
|  config  |          管理配置           |
| version  |      展示IPFS版本信息       |
|  update  | 下载并应用go-ipfs更新并应用 |
| commands |      列出所有可用命令       |

##### 5.1 ipfs config

##### 5.2 ipfs version

##### 5.3 ipfs update

##### 5.4 ipfs commands 

