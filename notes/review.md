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
- 读流程
	- 客户端向 NN 发起读请求，NN 检查客户端权限，检查目录树中是否存在文件，返回文件元数据地址
	- 