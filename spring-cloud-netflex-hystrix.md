# 熔断限流实现

## 配置环境

```xml
 <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
     <version>2.5.5.RELEASE</version>
</dependency>
```

```java
@EnableHystrix
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients(basePackages = {"org.order.api"})
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}

```



## 熔断降级配置

```java

    @HystrixCommand(
            commandProperties = {
                    //打开熔断降级
                    @HystrixProperty(name="circuitBreaker.enabled",value="true"),
                    //默认值为　10秒内　达到　20　次请求，并且失败次数达到　50％　则触发熔断　为　5000毫秒
                    //20 次请求
                    @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold",value = "5"),
                    //停止服务　5 秒
                    @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds",value = "5000"),
                    //请求失败次数大于　50％
                    @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage",value = "50")
            }
        	//回调方法
            ,fallbackMethod ="ordersFallback"
    )
    @GetMapping("/orders/{num}")
    public String orders(@PathVariable int num) throws Exception {
        if (num == 1){
            throw new Exception();
        }
        //其他服务
        return order.getOrders();
    }
   public String ordersFallback(int num){
        return "熔断降级";
    }
```

## 请求超时降级

```java
    //超时降级
    @HystrixCommand(
            commandProperties = {
                //超时时间　默认开启
                    @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds",value="1000"),
            }
            ,fallbackMethod ="ordersFallback"
    )
    @GetMapping("/orders1")
    public String orders1() throws Exception {
        return order.getOrders();
    }
    public String ordersFallback1(){
        return "超时降级";
    }
```

## 限流降级

### 线程

java使用

```java
	//线程池限流
    @HystrixCommand(
            commandProperties = {
                    @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds",value="10000"),
                   //配置限流策略　value = THREAD 则为线程池方案　SEMAPHORE　则为计数器方案
                	@HystrixProperty(name="execution.isolation.strategy",value="THREAD"),
            }
            ,fallbackMethod ="ordersFallback2",threadPoolKey = "default"
    )
    @GetMapping("/orders2")
    public String orders2() throws Exception {
        return order.getOrders();
    }
    public String ordersFallback2(){
        return "访问人数过多";
    }
```

系统配置

```yml
#fegin熔断配置
hystrix:
  threadpool:
    default:
      #线程数
      coreSize: 2
      #等待队列大小
      maxQueueSize: 4
      #拒绝策略启动大小
      queueSizeRejectionThreshold: 3
```



### 访问量

```java
  //线程池限流
    @HystrixCommand(
            commandProperties = {
                    @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds",value="10000"),
                    @HystrixProperty(name="execution.isolation.strategy",value="SEMAPHORE"),
                	//设置接口同时访问人数
                    @HystrixProperty(name="execution.isolation.semaphore.maxConcurrentRequests",value="3")
            }
            ,fallbackMethod ="ordersFallback3"
    )

    @GetMapping("/orders3")
    public String orders3() throws Exception {
        return order.getOrders();
    }
    public String ordersFallback3(){
        return "访问人数过多-访问数";
    }
```

## 容量监控

引入

```xml
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-annotations</artifactId>
      <version>2.11.2</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
      <version>2.2.2.RELEASE</version>
    </dependency>
```

启动类加入　@EnableHystrixDashboard注解

加入以下配置

```properties
server.port=8083
spring.application.name=spring-cloud-hystrix-dashboard
hystrix.dashboard.proxy-stream-allow-list=127.0.0.1
management.endpoints.jmx.exposure.include=*
management.endpoints.web.exposure.include=*
management.endpoint.health.show-details=always
```



## 熔断原理