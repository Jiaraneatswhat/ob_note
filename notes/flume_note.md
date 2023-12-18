# 1. 启动
## 1.1 flume-ng
```shell
# agent主类
FLUME_AGENT_CLASS="org.apache.flume.node.Application"
run_flume() {
  # 执行java -cp $FLUME_CLASSPATH
  $EXEC $JAVA_HOME/bin/java $JAVA_OPTS $FLUME_JAVA_OPTS "${arr_java_props[@]}" -cp "$FLUME_CLASSPATH" \
      -Djava.library.path=$FLUME_JAVA_LIBRARY_PATH "$FLUME_APPLICATION_CLASS" $*
}

# flume-ng agent
case "$mode" in
  agent)
    opt_agent=1
    ;;
    ...
esac

# 最终的执行逻辑
if [ -n "$opt_agent" ] ; then
  run_flume $FLUME_AGENT_CLASS $args
exit 0
```
## 1.2 Application.java
```java
public static void main(String[] args) {  
  try {  
    // Options对象底层维护了LinkedHashMap和ArrayList
    Options options = new Options();
    // 将参数封装在Option对象后添加到Options对象中
	Option option = new Option("n", "name", true, "the name of this agent");
	option.setRequired(true);
	options.addOption(option);

	option = new Option("f", "conf-file", true,
  "specify a config file (required if -c, -u, and -z are missing)");

	
```