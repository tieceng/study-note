### 含义

```
package tcl;

/**
 * start() 启动当前线程，调用当前线程的 run()
 * run()  通常要重写 Thread 类中的词方法，将创建的线程要执行的操作声明在此方法中
 * currentThread() 静态方法，返回当前执行的线程
 * getName() 获取当前线程的名字
 * setName() 设置当前线程名    eg：给主线程命名 Thread.currentThread.setName("主线程")
 * yield() 释放当前 cpu 的执行权
 * join()   在线程 A 中调用线程 B 的 join() 此时线程 A 就进入阻塞状态，知道线程 B 完全执行后，线程 A 才结束阻塞状态
 * stop() 已过时。执行此方法，强制结束当前线程
 * sleep(long millitime) 让当前线程 睡眠，指定 millitime  毫秒，在指定的毫秒时间内，当前线程是阻塞状态
 * isAlive() 判断当前线程是否存活
 *
 *
 */
public class ThreadTests {
    public static void main(String[] args) {

       
    }

}

```

