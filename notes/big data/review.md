#### 1. Linux 高级命令

- 查看内存: 
	- `top` 
	- `free -m `:  默认 `kb` 使用 `mb` 显示大小 `
	- `java` 进程 `jmap -heap` 
- 查看磁盘
	- `df -h`: -h 格式化输出
	- `du -sh`:  -s summary -h 格式化
- 查看端口 `netstat -anp` 
	- -a all 
	- -n 不显示别名 
	- -p 显示 pid, pname
- 查看进程 `ps -ef`
	- -e 显示全部
	- -f 显示全部字段

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
- 可以提高 copy 阶段 Buffer 占 Reducer 内存的比例，默认 0.7, 可以增加到 0.8
#### 4. Yarn 工作机制

- 客户端向 Yarn 申请提交 Job
- Yarn 向 RM 申请一个 Application，RM 返回资源提交路径
- Yarn 提交 job 运行所需资源(jar 包，切片文件，配置文件)
- Yarn 封装启动 MRAppMaster 的命令，选择 NodeManager 启动作为 AM 的 Container
- RM 将用户请求初始化为 Task, AM 启动后执行任务，去 RM 上申请到资源，在 NodeManager 中启动 Container，开启 YarnChild 进程，运行 MapTask 和 ReduceTask

#### 5. 调度器特点

- FIFO：单队列，先进先出，不用
- Capacity: 多队列，队列内部先进先出，用于并行度不高的场合(离线)
- Fair：多队列，按照缺额分配，并行度高(实时)
- 在 yarn-site.xml 中配置队列
- 队列的设置
	- 框架，业务线，部门，人员

#### 6. Flume 自定义拦截器

- 实现 Interceptor 接口
- 重写四个方法：
	- initialize()
	- intercept(Event e)
	- intercept(List<\Event> events)
	- close()
- 声明一个内部 Builder 类实现 Interceptor.Builder，重写 build 方法返回拦截器实例
- 打包到 Flume 的 lib 目录中
- 在配置信息中配置拦截器$Builder

#### 7. Channel 选择器

- 默认 Replicating，向每个 Channel 都会发送一份数据
- Load Balancing: 实现负载均衡，通过 round robin 或 random 来选择 Channel
- Multiplexing: 配合拦截器一起使用，根据 header 的信息，进行拦截发送给不同的 Channel

#### 8. Flume 优化

- 内存默认 20M，增加至 4-6G
- HDFS Sink 小文件：滚动参数
	- hdfs.rollInterval 30s
	- hdfs.rollCount 1024 bytes
	- hdfs.rollSize 10 event
- 提高吞吐量
	- batchSize：单次发送事件数：默认 100，调整至 2000-3000
- 丢数据
	- 通过监控器观察 Source 成功 Put 的事件数 = Sink 成功 Take 的事件数 + Channel 中现有的事件数
	- 采用 TailDirSource, KakfaChannel, 以及 KafkaSource, FileChannel, HDFSSink 组件，组件间的数据传输有事务保证，没有丢数据