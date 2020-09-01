

**服务雪崩**：当A调用微服务B，B调C，和其他微服务，这是扇出，当扇出链路上某个微服务调用响应时间过长或者不可用，对微服务的A的调用就会占用越来越多的系统资源，导致系统崩溃，所谓的雪崩效应

**服务熔断：**一般是某个服务异常引起的，相当于“保险丝”，当某个异常条件被触发，直接熔断整个服务，不是等到此服务超时

**服务降级：**降级一般是从整体负荷考虑，当某个服务熔断之后，服务器将不再被调用，客户端可自己准备一个本地的fallback回调，返回一个缺省值，虽然服务水平下降，当能用，比直接挂掉要强



# 服务降级

@EnableCircuitBreaker   开启熔断

```
		<dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>
```



![20200831101617](C:\Users\admin\Documents\Tencent Files\923935136\Image\SharePic\20200831101617.png)



## fallback单独使用（在方法上设置fallback函数）：

此时为服务端

```
@Override
    @HystrixCommand(fallbackMethod = "paymentFailHandler",
            commandProperties = {@HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds",value = "5000")})
    public String paymentFail(int id) {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
       // int i=10/0;
        return Thread.currentThread().getName()+"   "+id+"    "+"fail    "+serverport;
    }


    public String paymentFailHandler(int id){
        return Thread.currentThread().getName()+"   "+id+"    "+"   服务器繁忙或出错 ";
    }
```

如果出现超时或者异常，则直接跳转执行paymentFailHandler函数

```
@SpringBootApplication
@EnableEurekaClient
@EnableCircuitBreaker          //开启服务降级fallback
public class Hystrixpayment8005 {
    public static void main(String[] args) {
        SpringApplication.run(Hystrixpayment8005.class,args);
    }
}
```



## FeignFallback    fallback与openfeign配合使用

## （一般用在客户端，可以配合feign一起使用）：

### 在Feign中使用（Feign中集成了Hystrix和ribbon，所以需要设置一些超时时间、ribbon等待时间）

@EnableFeignClients中已经默认打开了断路器功能，所以这里的启动类上不需要再加@EnableCircuitBreaker注解

只需要在@FeignClient中为fallback参数指定fallback方法

在很早的版本中，Feign的断路器默认是开启的。后来有人提issue，认为这样不方便。一旦使用Feign就默认使用了断路器功能，导致了一些问题。后面从D版本开始断路器就是默认关闭的，需要手动打开。

```
#设置全局feign客户端的超时等待时间（openfeign集成了ribbon）
ribbon:
  ConnectTimeout: 10000        #建立连接所用的时间，
  ReadTimeout: 10000           #建立连接之后从服务端读取资源的时间

 # 设置全局hystrix的超时等待时间
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 10000

#打开feign对于降级处理的设置
feign:
  hystrix:
    enabled: true
```

```
@SpringBootApplication
@EnableFeignClients    //开启feign
@EnableEurekaClient
@EnableHystrix       //开启服务降级（这个注解里面包括了@EnableCircuitBreaker）
public class hystrixorder80 {
    public static void main(String[] args) {
        SpringApplication.run(hystrixorder80.class,args);
    }
}
```





Service接口，里面指明如果异常需要执行的fallback 函数

```
@FeignClient(value = "CLOUD-PROVIDER-SERVER",fallback = orderfallbackService.class)
//@FeignClient(value = "CLOUD-PROVIDER-SERVER")
public interface orderService {

    @GetMapping("/payment/hystrix/ok/{id}")
    public String Ok(@PathVariable("id") int id);

    @GetMapping("/payment/hystrix/fail/{id}")
    public String Fail(@PathVariable("id") int id);
}
```

写出fallback函数的实现类

```
@Service
public class orderfallbackService implements orderService{
    @Override
    public String Ok(int id) {
        return "服务器gg。。。。。。。。。。。o(╥﹏╥)o";
    }

    @Override
    public String Fail(int id) {
        return "服务器gg。。。。。。。。。。。o(╥﹏╥)o";
    }
}
```



controller

```
@RestController
public class ordercontroller {

    @Autowired
    private orderService orderService;

    @GetMapping("/consumer/order/ok/{id}")
    public String OK(@PathVariable("id") int id)
    {
        return orderService.Ok(id);
    }

    //客户端设置只愿意等待x秒
    @GetMapping("/consumer/order/fail/{id}")
    public String fail(@PathVariable("id") int id)
    {
        return orderService.Fail(id);
    }
}
```





# 服务熔断

服务熔断则是对于目标服务的请求和调用大量超时或失败，这时应该熔断该服务的所有调用，并且对于后续调用应直接返回，从而快速释放资源，确保在目标服务不可用的这段时间内，所有对它的调用都是立即返回，不会阻塞的。再等到目标服务好转后进行接口恢复。

熔断机制是应对雪崩效应的一种微服务链路保户机制，当扇出链路的某个微服务不可用或者响应时间太长时，会进行服务的降级，进而熔断该节点微服务的嗲用，快速返回错误的相应信息。当检测当该节点微服务调用响应正常后恢复调用链路，熔断机制的注解是@HystrixCommand

![20200831141548](C:\Users\admin\Documents\Tencent Files\923935136\Image\SharePic\20200831141548.png)

1. Closed：熔断器关闭状态，调用失败次数积累，到了阈值（或一定比例）则启动熔断机制；
2. Open：熔断器打开状态，此时对下游的调用都内部直接返回错误，不走网络，但设计了一个时钟选项，默认的时钟达到了一定时间（这个时间一般设置成平均故障处理时间，也就是MTTR），到了这个时间，进入半熔断状态；
3. Half-Open：半熔断状态，允许定量的服务请求，如果调用都成功（或一定比例）则认为恢复了，关闭熔断器，否则认为还没好，又回到熔断器打开状态；

![20200831104332](C:\Users\admin\Documents\Tencent Files\923935136\Image\SharePic\20200831104332.png)



```
circuitBreakerEnabled：是否允许熔断，默认允许；
circuitBreakerRequestVolumeThreshold：熔断器是否开启的阀值，也就是说单位时间超过了阀值请求数，熔断器才开；
circuitBreakerSleepWindowInMilliseconds：熔断器默认工作时间，超过此时间会进入半开状态，即允许流量做尝试；
circuitBreakerErrorThresholdPercentage：错误比例触发熔断
```



### 设置服务熔断的核心配置：

1. 启用断路器 `@HystrixProperty(name = "circuitBreaker.enabled", value = "true")`
2. 设置请求次数 `@HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10")`
3. 设置时间窗口期 `@HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "10000")`
4. 设置失败率 `@HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "60")`

如上设置的值，如果在10秒内，失败率达到请求次数（10）的百分之60，也就是6次就会打开断路器；否则断路器依然关闭

**当断路器开启时，所有的请求都不会转发，而是直接调用fallback方法；在一段时间后（默认5秒），断路器处于半开状态，尝试将请求转发，如果得到正确的响应。则将断路器关闭，恢复正常调用；否则断路器状态再次打开，重新计时后再进入半开状态**



### 这个是默认值

![20200831143758](C:\Users\admin\Documents\Tencent Files\923935136\Image\SharePic\20200831143758.png)





```
//------------------------------------下面是服务熔断--------------------
    @HystrixCommand(fallbackMethod = "paymentCircuitBreakHandler",commandProperties = {
            @HystrixProperty(name = "circuitBreaker.enabled",value = "true"),
            @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold",value = "10"),
            @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds",value = "10000"),
            @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage",value = "60")
    })
    public String paymentCircuitBreak(int id){
        if(id<0){
            throw new RuntimeException("不能为负");
        }
        String simpleUUID = IdUtil.simpleUUID();
        return simpleUUID;
    }


    public String paymentCircuitBreakHandler(int id){
        return "sorry，不能为负数啦啦啦啦。。。。。。。。。o(╥﹏╥)o";
    }
```



![20200831143722](C:\Users\admin\Documents\Tencent Files\923935136\Image\SharePic\20200831143722.png)



### 更多的属性设置查看HystrixCommandProperties类

```
protected HystrixCommandProperties(HystrixCommandKey key, HystrixCommandProperties.Setter builder, String propertyPrefix) {
        this.key = key;
        this.circuitBreakerEnabled = getProperty(propertyPrefix, key, "circuitBreaker.enabled", builder.getCircuitBreakerEnabled(), default_circuitBreakerEnabled);
        this.circuitBreakerRequestVolumeThreshold = getProperty(propertyPrefix, key, "circuitBreaker.requestVolumeThreshold", builder.getCircuitBreakerRequestVolumeThreshold(), default_circuitBreakerRequestVolumeThreshold);
        this.circuitBreakerSleepWindowInMilliseconds = getProperty(propertyPrefix, key, "circuitBreaker.sleepWindowInMilliseconds", builder.getCircuitBreakerSleepWindowInMilliseconds(), default_circuitBreakerSleepWindowInMilliseconds);
        this.circuitBreakerErrorThresholdPercentage = getProperty(propertyPrefix, key, "circuitBreaker.errorThresholdPercentage", builder.getCircuitBreakerErrorThresholdPercentage(), default_circuitBreakerErrorThresholdPercentage);
        this.circuitBreakerForceOpen = getProperty(propertyPrefix, key, "circuitBreaker.forceOpen", builder.getCircuitBreakerForceOpen(), default_circuitBreakerForceOpen);
        this.circuitBreakerForceClosed = getProperty(propertyPrefix, key, "circuitBreaker.forceClosed", builder.getCircuitBreakerForceClosed(), default_circuitBreakerForceClosed);
        this.executionIsolationStrategy = getProperty(propertyPrefix, key, "execution.isolation.strategy", builder.getExecutionIsolationStrategy(), default_executionIsolationStrategy);
        this.executionTimeoutInMilliseconds = getProperty(propertyPrefix, key, "execution.isolation.thread.timeoutInMilliseconds", builder.getExecutionIsolationThreadTimeoutInMilliseconds(), default_executionTimeoutInMilliseconds);
        this.executionTimeoutEnabled = getProperty(propertyPrefix, key, "execution.timeout.enabled", builder.getExecutionTimeoutEnabled(), default_executionTimeoutEnabled);
        this.executionIsolationThreadInterruptOnTimeout = getProperty(propertyPrefix, key, "execution.isolation.thread.interruptOnTimeout", builder.getExecutionIsolationThreadInterruptOnTimeout(), default_executionIsolationThreadInterruptOnTimeout);
        this.executionIsolationThreadInterruptOnFutureCancel = getProperty(propertyPrefix, key, "execution.isolation.thread.interruptOnFutureCancel", builder.getExecutionIsolationThreadInterruptOnFutureCancel(), default_executionIsolationThreadInterruptOnFutureCancel);
        this.executionIsolationSemaphoreMaxConcurrentRequests = getProperty(propertyPrefix, key, "execution.isolation.semaphore.maxConcurrentRequests", builder.getExecutionIsolationSemaphoreMaxConcurrentRequests(), default_executionIsolationSemaphoreMaxConcurrentRequests);
        this.fallbackIsolationSemaphoreMaxConcurrentRequests = getProperty(propertyPrefix, key, "fallback.isolation.semaphore.maxConcurrentRequests", builder.getFallbackIsolationSemaphoreMaxConcurrentRequests(), default_fallbackIsolationSemaphoreMaxConcurrentRequests);
        this.fallbackEnabled = getProperty(propertyPrefix, key, "fallback.enabled", builder.getFallbackEnabled(), default_fallbackEnabled);
        this.metricsRollingStatisticalWindowInMilliseconds = getProperty(propertyPrefix, key, "metrics.rollingStats.timeInMilliseconds", builder.getMetricsRollingStatisticalWindowInMilliseconds(), default_metricsRollingStatisticalWindow);
        this.metricsRollingStatisticalWindowBuckets = getProperty(propertyPrefix, key, "metrics.rollingStats.numBuckets", builder.getMetricsRollingStatisticalWindowBuckets(), default_metricsRollingStatisticalWindowBuckets);
        this.metricsRollingPercentileEnabled = getProperty(propertyPrefix, key, "metrics.rollingPercentile.enabled", builder.getMetricsRollingPercentileEnabled(), default_metricsRollingPercentileEnabled);
        this.metricsRollingPercentileWindowInMilliseconds = getProperty(propertyPrefix, key, "metrics.rollingPercentile.timeInMilliseconds", builder.getMetricsRollingPercentileWindowInMilliseconds(), default_metricsRollingPercentileWindow);
        this.metricsRollingPercentileWindowBuckets = getProperty(propertyPrefix, key, "metrics.rollingPercentile.numBuckets", builder.getMetricsRollingPercentileWindowBuckets(), default_metricsRollingPercentileWindowBuckets);
        this.metricsRollingPercentileBucketSize = getProperty(propertyPrefix, key, "metrics.rollingPercentile.bucketSize", builder.getMetricsRollingPercentileBucketSize(), default_metricsRollingPercentileBucketSize);
        this.metricsHealthSnapshotIntervalInMilliseconds = getProperty(propertyPrefix, key, "metrics.healthSnapshot.intervalInMilliseconds", builder.getMetricsHealthSnapshotIntervalInMilliseconds(), default_metricsHealthSnapshotIntervalInMilliseconds);
        this.requestCacheEnabled = getProperty(propertyPrefix, key, "requestCache.enabled", builder.getRequestCacheEnabled(), default_requestCacheEnabled);
        this.requestLogEnabled = getProperty(propertyPrefix, key, "requestLog.enabled", builder.getRequestLogEnabled(), default_requestLogEnabled);
        this.executionIsolationThreadPoolKeyOverride = HystrixPropertiesChainedProperty.forString().add(propertyPrefix + ".command." + key.name() + ".threadPoolKeyOverride", (Object)null).build();
    }

```





# Hystrix工作流程

![20200831144804](C:\Users\admin\Documents\Tencent Files\923935136\Image\SharePic\20200831144804.png)

整个流程可以大致归纳为如下几个步骤：

1. 创建HystrixCommand或者HystrixObservableCommand对象
2. 执行 Command
3. 检查请求结果是否被缓存
4. 检查是否开启了短路器
5. 检查 线程池/队列/semaphore 是否已经满
6. 执行 HystrixObservableCommand.construct() or HystrixCommand.run()
7. 计算短路健康状况
8. 调用fallback降级机制
9. 返回依赖请求的真正结果



**一、创建Command对象**

HytrixCommand和HystrixObservableCommand包装了对外部依赖访问的逻辑。整个流程的第一个步骤就是实例化HystrixCommand或者HystrixObservableCommand对象。在构造这两个Command对象时，可以通过构造方法传递任何执行过程中需要的参数。

如果对外部依赖调用只返回一个结果值，那么可以实例化一个HystrixCommand对象

HystrixCommand command = newHystrixCommand(arg1, arg2);

如果在调用外部依赖时需要返回多个结果值时，可以实例化一个HystrixObservableCommand对象

HystrixObservableCommand command = newHystrixObservableCommand(arg1, arg2);

**二、执行 Command**

Hystrix API提供了四个方法来执行Command，分别如下：

1. execute()方法，调用后直接block住，属于同步调用，直到依赖服务返回单条结果，或者抛出异常
2. queue()方法，返回一个Future，属于异步调用，后面可以通过Future获取单条结果
3. observe()方法，订阅（subscribe）一个Observable对象，Observable代表的是依赖服务返回的结果，获取到一个那个代表结果的Observable对象的拷贝对象
4. toObservable()方法，返回一个Observable对象，如果我们订阅（subscribe）这个对象，就会执行command

**其中execute()和queue()仅仅对HystrixCommand适用，适用于外部依赖只返回一个结果的情况下**。execute()方法调用的是HystrixCommand.queue().get()方法获取最终的返回值。而queue()方法调用的是HystrixCommand.toObservable().toBlocking().toFuture() 方法来实现的。也就是说，无论是哪种执行command的方式，最终都是依赖toObservable()去执行的。只不过是HystrixCommand这两个方法返回的结果只有一个值。

**三、检查请求结果是否被缓存**

如果开启了Hystrix请求结果缓存功能，并且这个调用的结果在缓存中存在，那么hystrix会直接将缓存的结果返回。在使用HystrixCommand和HystrixObservableCommand类封装依赖请求逻辑时，可以通过重载getCacheKey()实现请求结果的缓存。通过实现同类请求结果的缓存，可以在同一个请求Context中有效降低对外部依赖的实际调用次数。

以下是在两个线程中对外部依赖http请求的流程示意图：

![img](https://img-blog.csdn.net/20180519183950129)

结果缓存的好处：

1、在同一个请求上下文中，可以减少使用相同参数请求原始服务的开销。

2、请求缓存在 run() 和 construct() 执行之前生效，所以可以有效减少不必要的线程开销。

**四、检查是否开启了短路器**

接下来需要判断熔断器的开启状态。如果断路器被打开了，那么hystrix就不会执行这个command，而是直接去执行fallback降级机制。HystrixCircuitBreaker是Hystrix中实现熔断器功能的核心类，HystrixCommand和HystrixObservableCommand通过与HystrixCircuitBreaker交互实现了熔断器功能的开启和关闭。

下面是HystrixCircuitBreaker和Command的交互流程和内部开关控制逻辑：

![img](https://img-blog.csdn.net/20180519184011783)

熔断器开关控制条件：

- 对外部依赖调用的次数满足配置的阈值
- 对外部依赖调用发生错误的比率满足配置的阈值

在满足以上两个条件时，熔断器打开熔断开关，之后所有对外部依赖调用都将被直接断开。在开关打开时长超过试探窗口期后，熔断器将尝试放行部分外部依赖的调用，根据试探的结果决定重新开启或则关闭熔断开关。

**五、检查 线程池/队列/semaphore 是否已经满**

Hystrix引入了ThreadPool和semaphore两种方式实现资源隔离机制。如果command对应的线程池/队列/semaphore已经满了，那么也不会执行command，而是直接去调用fallback降级机制

**六、执行 HystrixObservableCommand.construct() or HystrixCommand.run()**

HystrixCommand和HystrixObservableCommand封装了外部依赖调用。通过重写HystrixCommand的run()方法和HystrixObservableCommand类的construct()方法实现外部依赖的调用。

- HystrixCommand的run()方法只返回一个请求结果
- HystrixObservableCommand的construct()方法返回一个Observable对象，通过订阅该Observerable可以获取多个返回结果

如果run()方法或则construct()方法在执行外部依赖请求出现超时，那么请求线程或者调用线程将抛出TimeOutException。Hystrix接下来直接返回调用降级方法获得的值而不管超时后外部依赖实际返回结果。除了通过抛出一个InterruptException外没有更好的方式中断外部调用线程的执行，所以建议调用外部依赖逻辑添加对InterruptException的处理以便在必要时中断请求线程的执行，而目前很多Http Client的实现并没有很好的支持该中断的响应，所以在使用Http Client发送http请求时需要合理设置超时时间以免长时间阻塞执行线程。

如果外部依赖调用正常返回，那么Hystrix metrics将进行必要的数据统计，这些统计数据关系到HystrixCircuitBreaker的正确运转。

**七、计算短路健康状况**

Hystrix会将每一个依赖服务的调用成功，失败，拒绝，超时，等事件，都会发送给circuit breaker断路器，HystrixCircuitBreaker通过维护一系列的counter记录外部依赖请求的执行情况。断路器根据维护的这些信息，在符合触发条件下开启断路功能，在条件合适的时候关闭断路开关。如果打开了断路器，那么在一段时间内就会直接短路，然后如果在之后第一次检查发现调用成功了，就关闭断路器。

**八、调用fallback降级机制**

在以下几种情况中，hystrix会调用fallback降级机制：run()或construct()抛出一个异常，短路器打开，线程池/队列/semaphore满了，command执行超时了。

一般在降级机制中，都建议给出一些默认的返回值，比如静态的一些代码逻辑，或者从内存中的缓存中提取一些数据，尽量在这里不要再进行网络请求了；**即使在降级中，一定要进行网络调用，也应该将那个调用放在一个HystrixCommand中，进行隔离**。

HystrixCommand通过重载getFallback()方法或则HystrixObservableCommand的resumeWithFallback()方法实现服务的降级。通过getFallback()方法只能返回一个结果值，而通过resumeWithFallback()方法返回的Observable对象可以给请求线程返回多个结果值。

如果fallback返回了结果，那么hystrix就会返回这个结果。对于HystrixCommand，会返回一个Observable对象，其中会发返回对应的结果； 对于HystrixObservableCommand，会返回一个原始的Observable对象。

如果没有实现fallback，或者是fallback抛出了异常，Hystrix会返回一个Observable，但是不会返回任何数据。

不同的command执行方式，其fallback为空或者异常时的返回结果不同

- 对于execute()，直接抛出异常
- 对于queue()，返回一个Future，调用get()时抛出异常
- 对于observe()，返回一个Observable对象，但是调用subscribe()方法订阅它时，理解抛出调用者的onError方法
- 对于toObservable()，返回一个Observable对象，但是调用subscribe()方法订阅它时，理解抛出调用者的onError方法

**九、返回依赖请求的真正结果**

外部依赖调用成功时将通过Observable对象返回结果值，鉴于构造Command对象的不同和调用执行方法的不同，结果值可能经过一些列的转化过程，不过从本质上讲都是通过Observable对象返回结果值。

外部依赖结果值返回流程图：

![img](https://img-blog.csdn.net/20180519184032705)

不同执行方式：

- execute()，获取一个Future.get()，然后拿到单个结果
- queue()，返回一个Future
- observer()，立即订阅Observable，然后启动8大执行步骤，返回一个拷贝的Observable，订阅时理解回调给你结果
- toObservable()，返回一个原始的Observable，必须手动订阅才会去执行8大步骤





# Hystrix图形化界面Dashboard

http://localhost:9002/hystrix    访问此网址，设置的端口号为9002

Dashboard需要自己写一个启动类

```
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
        </dependency> 
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
```

```
@SpringBootApplication
@EnableHystrixDashboard             //开启Dashboard的图形化监控
public class HystrixDashboard9002 {
    public static void main(String[] args) {
        SpringApplication.run(HystrixDashboard9002.class,args);
    }
}
```





提示：需要在被监控的启动类进行配置（由于SpringCloud升级导致的）

假设此时监控的是8005

```
@SpringBootApplication
@EnableEurekaClient
@EnableCircuitBreaker          //开启服务降级fallback
public class Hystrixpayment8005 {
    public static void main(String[] args) {
        SpringApplication.run(Hystrixpayment8005.class,args);
    }
    
    //需要进行配置这段话
    @Bean
    public ServletRegistrationBean getServlet(){
        HystrixMetricsStreamServlet hystrixMetricsStreamServlet = new HystrixMetricsStreamServlet();
        ServletRegistrationBean servletRegistrationBean = new ServletRegistrationBean(hystrixMetricsStreamServlet);
        servletRegistrationBean.setLoadOnStartup(1);
        servletRegistrationBean.setName("HystrixMetricsStreamServlet");
        servletRegistrationBean.addUrlMappings("/hystrix.stream");
        return servletRegistrationBean;
    }

}
```



![20200831153925](C:\Users\admin\Documents\Tencent Files\923935136\Image\SharePic\20200831153925.png)

![20200831154028](C:\Users\admin\Documents\Tencent Files\923935136\Image\SharePic\20200831154028.png)

