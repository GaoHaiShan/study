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
		将内存中的一部分数据保存到 cpu中，让运算快速执行，运算结束后将修改值返回到内存中		

## cpu 高速缓存带来问题:			

	缓存一致性问题：		
	   由定义可知，cpu处理过程为，先将 cpu 计算所需要的数据加载到高速缓存中，处理完成后再将结果返回到主内存中。	
	由于存在多个 cpu ,同一份数据可能会储存在到多个 cpu 内存中，如果某个 cpu 的线程修改这份数据，并储存到了主内		
	存中，这是主内存数据与其他 cpu 缓存数据便会出现缓存不一致问题。			

## cpu 高速缓存问题解决方案:			

	总线锁：		
	缓存锁：		

# storebuffers：			

## 出现场景：		

## 出现的问题：		

# cpu 内存屏障：		

#JMM Java 内存模型：		