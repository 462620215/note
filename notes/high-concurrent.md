# 高并发

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