# 1 基础知识
- 进程和线程
	- 进程由指令和数据组成，程序被运行，加载代码至内存，就开启了一个进程
	- 进程可以视为程序的一个实例
	- 一个进程可以分为多个线程
	- 线程是最小的调度单位
	- 线程是进程的子集，进程拥有共享的资源共内部线程共享
- 并发和并行
	- `OS` 通过任务调度器将 `CPU` 的时间片分给不同的线程使用，微观上串行，宏观上并行。这种轮流使用 `CPU` 的方式叫做并发
	- 多核 `CPU` 可以真正实现并行，同一时间运行多个线程
- 栈与栈帧
	- 每个线程启动后，虚拟机会为其分配内存
	- 每个栈由多个栈帧组成，每个线程只有一个活动栈帧，对应着正在执行的方法
- 线程上下文切换
	- `CPU` 不再执行当前线程，转而执行另一个线程
		- 线程 `CPU` 时间片用完
		- `GC`
		- 优先级更高的线程出现
		- 线程调用了 `sleep(), yield(), wait()` 等方法
	- 上下文切换时，程序计数器会记住下一条 `JVM` 指令的地址，保存当前线程的信息，方便下次继续执行
	- 状态包括程序计数器，虚拟机栈中每个帧的信息，局部变量，返回地址等
# 2 java 线程
## 2.1 创建运行线程
### 2.1.1 Thread
```java
// 直接创建 Thread 对象，重写 run() 方法
new Thread(() -> ...)

new Thread() {
	@Override  
	public void run() {...}
}
```
### 2.1.2 Thread & Runnable
- Runnable.java
```java
@FunctionalInterface  
public interface Runnable {  
	public abstract void run();  
}
```
- `Runnable` 作为 `Thread` 要执行的任务传入
```java
Runnable r = () -> ...;  
  
Thread t = new Thread(r, "thread_name");

// Thread 的构造方法中可以传入 Runnable
public Thread(Runnable target, String name) {  
    init(null, target, name, 0);  
}
```
### 2.1.3 Callable & FutureTask
- `Callable` 有返回值
- Callable.java
```java
@FunctionalInterface  
public interface Callable<V> {  
	V call() throws Exception;  
}
```
- `FutureTask` 实现了 `RunnableFuture`
- `RunnableFuture` 继承了 `Runnable` 和 `Future`
```java
// 定义一个 FutureTask
FutureTask<Integer> task = new FutureTask<>(new Callable<Integer>() {  
    @Override  
    public Integer call() throws Exception {...}  
});

// FutureTask 的构造器可以传入 Callable 对象
public FutureTask(Callable<V> callable) {  
    if (callable == null)  
        throw new NullPointerException();  
    this.callable = callable;  
    this.state = NEW; 
}
// 将 FutureTask 传入 Thread 构造方法中执行
new Thread(task);

```
