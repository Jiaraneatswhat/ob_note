# 1. 启动
## 1.1 脚本
### 1.1.1 start-hbase.sh
```shell
# hbase-site.xml将分布式设置为true
distMode=`$bin/hbase --config "$HBASE_CONF_DIR" org.apache.hadoop.hbase.util.HBaseConfTool hbase.cluster.distributed | head -n 1`

# 没有传入启动命令，默认为start
if [ "$1" = "autostart" ]
then
  commandToRun="--autostart-window-size ${AUTOSTART_WINDOW_SIZE} --autostart-window-retry-limit ${AUTOSTART_WINDOW_RETRY_LIMIT} autostart"
else
  commandToRun="start"
fi

if [ "$distMode" == 'false' ] 
then
  "$bin"/hbase-daemon.sh --config "${HBASE_CONF_DIR}" $commandToRun master
else
  "$bin"/hbase-daemon.sh --config "${HBASE_CONF_DIR}" $commandToRun master
  "$bin"/hbase-daemons.sh --config "${HBASE_CONF_DIR}" \
    --hosts "${HBASE_REGIONSERVERS}" $commandToRun regionserver
fi
```
### 1.1.2 hbase-daemon.sh
```shell

```
## 1.2 HMaster.main()
```java
public static void main(String [] args) {
    new HMasterCommandLine(HMaster.class).doMain(args);
}

public void doMain(String args[]) {
    try {
      int ret = ToolRunner.run(HBaseConfiguration.create(), this, args);
    }
}

public static int run(Configuration conf, Tool tool, String[] args) 
    throws Exception{
    GenericOptionsParser parser = new GenericOptionsParser(conf, args);
    //set the configuration back, so that Tool can configure itself
    tool.setConf(conf);
    
    //get the args w/o generic hadoop args
    String[] toolArgs = parser.getRemainingArgs();
    return tool.run(toolArgs);
}

// HMasterCommandLine
public int run(String args[]) throws Exception {
    if ("start".equals(command)) {
      // 启动master
      return startMaster();
    } 
}
```
## 1.2 startMaster()
```java
private int startMaster() {
    try {
      // If 'local', defer to LocalHBaseCluster instance.  Starts master
      // and regionserver both in the one JVM.
      if (LocalHBaseCluster.isLocal(conf)) {...} 
        else {
        logProcessInfo(getConf());
        // 通过反射创建HMaster对象
        HMaster master = HMaster.constructMaster(masterClass, conf);

        // HMaster继承了HRegionServer继承了HasThread
        master.start();
        master.join();
    } 
    return 0;
}
```