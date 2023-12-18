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
- 危害
	- 存储：每个块对应 150 byte 的元数据信息，占用 NN 内存
	- 计算：默认每个文件切一片
- 解决
	- 源头
		- Flume 通过 HDFS Sink 向 HDFS 写入时，通过设置滚动参数来避免小文件
		- ADS 层 SQL 今天的数据 union all 昨天之前的数据进行 overwrite
		- 手动合并
		- har 归档
	- 计算
		- JVM 重用(只适用于 MR 进程)
		- CombineTextInputFormat

#### 3. Shuffle 优化
- map()方法后，通过 collect() 将数据写入到环形缓冲区中，根据分区号进行快速排序，分区内有序后，达到 80% 的阈值后，进行归并排序生成单个大文件，向磁盘溢写
- reduce()方法有 copy, merge, reduce 三个阶段，首先去拉取对应的分区的数据，进行 merge 合并相同分区的数据，最后通过 reduce 方法写出
- 可以增大环形缓冲区的大小到 200M，因为切片在最后一片时，如果剩余的数据量 < 块大小的 1.1 倍，会将剩余部分与上一片合在一起，这样就可以减少溢写的次数，以及防止归并排序。若溢写的次数仍然较多，说明文件大小很大且不能切分
- merge 默认的合并次数是 10，可以提高到 20
- 在不影响业务结果的条件下，可以在归并后使用 Combiner
- 为了减少磁盘 IO，可以使用压缩(IO 遇到瓶颈时)
- 可以增加 Container 的内存和 CPU 资源
- reduce 阶段，可以提高 Reduce 去 Map 拉取数据的并行度，默认 5，可以提高到 10
- 