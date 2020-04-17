# volatitle 使用：		
通过一段代码了解为什么要用 volatitle 		
 ```java
public class Volatitle implements Runnable {
    private static volatile Boolean count = false;
    @Override
    public void run() {
        int i = 0;
        while (!count){
            i++;
        }
    }
    public static void main(String[] args) {
        Volatitle v1 = new Volatitle();
        Thread t1 = new Thread(v1);
        t1.start();
        Thread.sleep(1000);
        count=true;
    }
}
```

理论上 在程序执行 1s 后便会结束，如果不加 volatile 程序则会一直循环下去。由此可见 volatile 是将主线程更改 count =true 的结果 通知给 t1 线程。		

# 可见性问题历史：

## cpu 高速缓存出现场景：

	    硬件层面由于 cpu 性能 > 内存 > io设备，所以为了提高计算机性能做了以下操作：		
	       1、为 cpu 添加高速缓存
	       2、操作系统增加了线程和进程，通过 cpu 时间切换，使 cpu 利用最大化
	       3、编译器指令优化。 		
	    定义：		
	       将数据保存到主n内存中，莫个 cpu 访问共享数据时，将主内存共享数据拷贝到自己的工作缓存中，以达到让运算快速执行，运
        算结束后将修改值返回到主内存中。
        模型：
            内存：
                    cpu1 工作缓存 => cpu1
                    cpu2 工作缓存 => cpu2

## cpu 高速缓存带来问题:			

	   缓存一致性问题：		
	       由定义可知，cpu处理过程为，先将 cpu 计算所需要的数据加载到高速缓存中，处理完成后再将结果返回到主内存中。	
	   由于存在多个 cpu ,同一份数据可能会储存在到多个 cpu 内存中，如果某个 cpu 的线程修改这份数据，并储存到了主内		
	   存中，这是主内存数据与其他 cpu 缓存数据便会出现缓存不一致问题。			

## cpu 高速缓存问题解决方案:			

	   总线锁：
            当一个 cpu 修改数据时，向主内存区发送一个信号，终止其他 cpu 与主内存的通信，无法通过主内存访问共享元素，这样
        性能开销显然不符合我们的预期。
	   缓存锁：
            于是通过减小锁的粒度，保证共享数据一致性，于是引入了缓存锁，核心是基于缓存一致性协议实现的。
            缓存一致性协议:
                MESI 协议:
                    modify: 共享数据只缓存到了当前 cpu ,并且已被修改，与主内存数据不一致
                    exclusive:独占，共享数据只缓存在当前cpu ，并未进行修改
                    shared:共享数据缓存到了各个 cpu 并且每个 cpu 数据一致
                    invaid:共享数据cpu缓存失效
                MESI原则:
                    cpu 读请求: 缓存处于 m e s状态是可读取的，处于 i状态无法读取
                    cpu 写请求: 缓存处于 m e 状态是可以写入的，但处于 s 状态需要将其他 cpu 缓存设置为 i 状态才可以及进行写入
                MESI保证共享数据一致性原理:
                    现有 cpu0,cpu1 两个cpu。cpu0 需要对共享数据进行写入，首先 cpu0需要发送一个缓存失效通知给 cpu1,
                cpu1 收到后发送一个确实回执，然后 cpu0 才可以对数据进行修改。
                MESI 缺陷：
                    在 cpu0 发送通知到收到 cpu1 回执期间，cpu0 处于阻塞状态，严重浪费 cpu 资源。 

# storebuffers：			

## 出现场景：		

        由于 mesi 一致性协议出现的 cpu 资源浪费问题。为了避免 cpu 阻塞，在 cpu 与其高速缓存区之间加入了 storebuffers 缓存区，
    当 cpu 执行写入操作时，会将所要修改的数据放入 storebuffers 中并向其他 cpu 发送缓存失效通知，然后继续执行其他指令，当其
    他 cpu 收到通知，并确认以后，再将 storebuffers修改的内容写入cpu 高速缓存中，再由 cpu 高速缓存写入到主内存中。这样便出现了
    异步操作。
## 出现的问题：		
        1、由于 storebuffers 是在其他 cpu 确认以后才将数据同步，这样无法确定数据何时去同步.
        2、指令重排序问题：下面通过例子演示
            共享变量 boolean b = false; int a = 3
            条件: b 在 cpu 中处于 e 独占状态，a 处于 s 状态
            cpu0 执行代码：

```java
    void execCpu0(){
        a = 10;
        b = true;
    }    
```

        cpu1 执行代码：

```java
    void execCpu1(){
        if(b){
           boolean c = a == 10;
        }
    }
```

            cpu0 执行代码的时候 由于 a 处于 s 状态，所以先向其他 cpu 发送缓存失效通知，将 a=10 放入storebuffers 中，假设
        其他 cpu 响应很慢，所以 cpu0 会继续执行 b = true ,由于 b 处于独占状态可以直接写入， 当 cpu1 执行时 b 为 true 往下
        执行获取 a 的时候由于这是  a=10 还存储在 storebuffs 中，内存中的 a 依旧为 3 所以 c 最后的值为 false 。所以本应该
        先执行 a = 10 指令 在执行 b = true 指令 最后返回 c = true,结果却是先写入了 b = true 指令,a 的写入阻塞导致 c = flase。   

# cpu 内存屏障：	

        由于上面的重排序问题导致的可见性问题，所以引入了内存屏障，有软件自己决定什么时候加入屏障，保证数据的可见性。

## 读屏障 ifence
    
            读屏障之后的读操作，全在读屏障之后执行，配合写屏障使得写屏障之前的操作对于读屏障之后的操作可见。
            个人理解：
                读屏障就是强制更新当前cpu高速缓存中的数据，保证缓存中的数据是最新的。

## 写屏障 sfence

            写屏障就是在写屏障之前，将所有已经储存在 storebuffes中的数据都同步主内存中之后 在进行屏障后操作。也就是说，
        写屏障之前的指令，对写屏障之后的指令完全可见。（这里的指令包括读指令和写指令）
            个人理解：
                写屏障就是强行将 storebuffers 中的数据写入到内存中。

## 全屏障 mfence 

            确保屏障之前的读写操作完全同步到内存之后，在进行全屏障之后的读写操作。

#JMM Java Memory Model 内存模型：		

        jmm 属于语言级别的对硬件模型的抽象，它定义了多线程程序的读写操作的行为规范：在jvm 虚拟机中 将共享变量储存到内存，
    以及从内存中取出共享变量的底层实现细节。注意: jmm 依旧存在重排序以及缓存不一致问题。
## JMM内存模型

        主内存：（共享资源）
            工作内存1 （线程自身可见）：线程1
            工作内存2 （线程自身可见）：线程2

## JMM 如何解决可见性以及有序性问题

        volatitle \ synchronized \ final

## JMM 内存屏障
    
        LoadLoad Barriers：
            Load1; LoadLoad; Load2  确保load1数据装载 优先于 load2以及后指令的数据装载
            个人理解：load1 数据装载完 在进行 load2数据装载。
        StoreStore Barriers：
            Store1; StoreStore; Store2; 确保Store1数据储存 优先于 Store2以及后指令的数据存储
            个人理解：Store1数据存储完 在进行 Store2数据存储
        LoadStore Barriers：
            Load; LoadStore; Store; 确保 Load 数据装载优先于 Store以及以后的数据存储
            个人理解：load数据装载完 在进行 Store 数据存储
        StoreLoad Barriers：
            Store; StoreLoad; Load;确保 Store 数据存储优先于 Load以及以后的数据装载
            个人理解: Store数据存储完 在进行load 数据装载

## happenBefore

        a happenBefore b 就是 a 指令执行完 在执行 b 指令

# Volatitle

        对于 volatitle 修饰的变量，变量的写操作一定 happenbefore 读操作。
