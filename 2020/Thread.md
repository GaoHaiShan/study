线程使用

>不需要返回值：

>>继承 Thread 类

>>实现 Runable 接口

>需要返回值：

>>1、实现 Callable<T> 接口

>>2、使用线程池 ExecutorService 初始化线程

>>3、将 Callable 实现类实例化

>>4、Future<T> f 进行接收,f = executorService.submit(callanle) 

>>5、 f.get()获取返回值

线程状态

>new

>>初始状态：线程已经被初始化，还未执行 start 方法

>runnable

>>运行状态：线程准备就绪和线程正在执行统称

>blocked

>>阻塞状态：

>>>等待阻塞：线程调用 wait() 方法引起的阻塞

>>>同步阻塞：线程竞争锁失败后引起的阻塞

>>>其他阻塞: 由于 sleep join io操作引起的阻塞

>waiting\time_waiting 

>>等待\超时等待状态：满足某种条件\超过时间自动返回

>timeinated：

>>终止状态：线程执行完毕 

线程状态转换：

>1、线程执行 start 方法进入初始状态，等待 cpu 调度。

>2、线程准备就绪或 cpu 调度该线程：进入运行状态。

>3、运行状态中状态转换：

>>条件：需要竞争锁

>>过程：

>>>竞争锁成功：

>>>>执行代码:

>>>>>条件：执行 wait()、join() 等方法

>>>>>过程: 进入阻塞状态，

>>>竞争锁失败：

>>>>进入同步阻塞状态。

>4、线程执行完毕进入终止状态

线程启动原理

>start() 调用 native start0() 来启动一个线程,

>start0（） 在 Thread 类中的静态代码块中进行注册，执行registerNatives()

>[registerNatives()方法位置](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/00cd9dc3c2b5/src/share/native/java/lang/Thread.c)

>``` c

>{"start0","()V",(void *)&JVM_StartThread},

>```

>JVM_StartThread 在jvm.cpp(需下载host源码)

>之后操作大致就是 JVM_StartThread 调用 JavaThread thread() 方法

>thread（） 调用 os::create_thread() 方法创建线程

>线程启动会调用 Thread.cpp Thread::start() 启动线程

>最终会调到 Thread.cpp JavaThread::run()

线程中断\复位

>中止：调用 Tread.interrupt() 方法

>>interrupt 只是通知线程可以中止，并不会使线程立即中止，线程何时中止由线程自身决定(线程可以通过 Thread.currentThread.isInterrupted()判断自身是否被中断过)。

>>通常我们是 通过 标识符或中断操作将线程停止，而不是用 stop() 方法武断终止，这种方法可以使线程有时间清理资源。

>复位：调用 Thread.interrupted() 方法

>>线程复位是对外届 interrupt() 的响应，表示已收到中断信号，但不会立即中断，具体什么时候中断，由线程自己绝定

线程终止原理：

>jvm 内设置一个标识符 interrupted = true 中断线程 调用线程的interrupted() interrupted= false



线程竞争锁失败,wait()实现大致逻辑：

>将线程加入到同步阻塞队列中

>调用 thread.interrupt()中断线程 将 interrupted = true

>正在执行的线程调用 notify 或执行完毕后调用ParkEvent part 清除中断标识 interrupted = false

>调用 thread.interrupted() 进行复位 

>继续竞争锁，失败则继续以上操作。



Object.wait(),Thread.sleep()等抛出InterruptedException理解

>这个异常是告诉调用者，调用这些方法会执行线程的中断方法，这些方法回事线程尝试中断，并提前返回

>抛出这个异常并不是线程立即停止，接下来的操作取决线程自身：

>>例如直接捕获异常不做任何处理、将异常外抛、停止线程打印异常信息等。





