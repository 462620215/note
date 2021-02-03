<span style="font-family: Monaco">

# 高并发

## 概念
### 同步(Synchronous)和异步(Asynchronous)
- 同步：调用方法的时候需要等待方法执行完毕后才能继续后续操作；
- 异步：调用方法的时候，被调用方法会直接返回响应给调用方，然后另起一个线程来执行逻辑；

### 并发(Concurrency)和并行(Parallelism)
- 并发：单核中多线程之间的交替执行关系（中断），例如四个柜台只有1个人来轮流处理业务；
- 并行：多核中不同核的线程间的执行关系，真实，例如四个柜台同时有4个人来处理业务；

### 临界区
- 用来表示一种公共资源或者说共享数据，可以被多个线程使用，但是每一次只能有一个线程使用它，一旦临界区资源被占用，其他线程要想使用这个资源就必须等待；例如

### 阻塞(Blocking)和非阻塞(Non-Blocking)
- 阻塞：
- 非阻塞：

### 死锁(Deadlock)、饥饿(Starvation)和活锁(Livelock)

## 线程的状态
   - NEW `初始化`
     - 触发方法
       - 实例化一个Thread对象
   - RUNNABLE `运行`
     - 触发方法
       - 调用Thread实例的start()方法
   - WAITING `等待`
   - TIMED_WAITING `超时等待`
   - BLOCKED `阻塞`
   - TERMINATED `终止`
## 面临的问题
    - 可见性
    - 原子性
    - 顺序性
## Happens-Before原则

</span>