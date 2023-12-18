#### 1. Linux 高级命令

- 查看内存: 
	- `top` 
	- `free -m` 
	- `java` 进程 `jmap -heap` 
- 查看磁盘
	- `df -h`
	- `du -sh`
- 查看端口 `netstat -anp`
- 查看进程 `ps -ef`

#### 2. 启动停止脚本
```shell
#!/bin/sh
case $1 
"start")
	for i in hadoop102 hadoop103 hadoop104
	do
		ssh $i "xx-start.sh"
   done
;;
"stop")
	for i in hadoop102 hadoop103 hadoop104
	do
		ssh $i "xx-stop.sh"
   done
;;
*)
;;
esac
```
#### 3. HDFS 读写流程
- 写流程
	- 客户端首先创建 DataStreamer 用于后续数据传输
	- 通过 RPC 向 NN 发起读请求，NN 检查客户端权限，检查目录树中是否存在文件，返回文件所在块地址
	- 客户端开始写，产生 packet 后，NN 通过机架感知定位要写入的块所在的位置，优先同机架，其次另一个机架，以及另一个机架的另一个节点，将块所在的 DN 通知给客户端
	- 客户端与 DN 建立连接，写入数据，DN 会通过 DataXceiver 线程返回 ack，将数据持久化到磁盘
- 读流程
	- 客户端创建输入流，向 NN 建立连接获取块地址
	- NN 检查权限和目录树后，返回块地址
	- 客户端和 DN 建立连接读取数据

#### 4. 小文件危害
- 存储：每个块对应 150 byte 的元数据信息，占用 NN 内存
- ji'su