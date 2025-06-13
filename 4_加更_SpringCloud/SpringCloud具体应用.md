# SpringCloud具体应用



## Nacos-注册/配置中心

***注册中心：服务注册/发现功能，能够协助服务之间的调用***
***配置中心：存放服务所需配置，方便统一管理***

### 启动功能

在对应服务启动类上添加**@EnableDiscoveryClient**

### 服务发现

可用**DiscoveryClient**或NacosServiceDiscovery，这里引入DiscoveryClient类比较通用
```java
@Autowired
DiscoveryClient discoveryClient;
//获取全部service名
discoveryClient.getServices();
//获取某个service的实例
discoveryClient.getInstances(service)
```

### 远程调用（实际只解决了服务发现与低级别调用）

```java
1.根据已知服务名获取实例信息
discoveryClient.getInstances(service);
2.根据实例信息拼接URL
String url = "http://"+choose.getHost() +":" +choose.getPort() +"/product/"+productId;
3.以对象形式传回
RestTemplate restTemplate = new RestTemplate();
restTemplate.getForObject(url,ServiceModel.class)
```

此处可以看到要使用获取实例对象信息比较麻烦，而且需要拼装url。后续做了一些改进，也引入了更适合RPC的Openfeign。

### 实现远程调用的负载均衡

1. 手动创建处理类。和上述流程相同，只是获取实例信息时用**LoadBalancerClient**
   ```java
   @Autowired
   LoadBalancerClient loadBalancerClient;
   loadBalancerClient.choose(service);
   //后续相同，先拼接后返回
   ```

2. 使用注解开发（**@LoadBalanced**）

    ```java
    在RestTemplate类中使用@LoadBalanced
    无需获取实例信息拼接字符串，只放入服务名即可
    String url = "http://service-product/product/"+productId
    ```

### 配置中心

1. 在对应服务下指定nacos配置地址
   ```
   spring.cloud.nacos.server-addr=localhost:8848
   spring.config.import=nacos:service-order.properties
   ```

2. 打开之前运行的Nacos网页，填写DataID与需要配置内容

3. 获取配置

   1. 使用依赖注入获取
      ```java
      @Value("${要填的属性}")
      String propretyOne;
      建议在使用配置的类上加上@RefreshScope注解，能够实时刷新
      ```

   2. 直接在model层使用**@ConfigurationProperties**(更推荐)
      ```java
      抽象上述需要配置的属性为properties类，然后在该类使用@ConfigurationProperties(prefix = "具体属性")
      ```

### 补充

```
namespace：命名空间，可以写开发、测试、生产
dataID：填名字.后缀
groupID：用微服务名称即可
```

==TODO监听配置变化：暂时不学，等开发实际阶段用。==



## OpenFeign-远程调用

远程调用（RPC）：能够实现服务之间的调用，有时候是在不同节点情况下调用。而OpenFeign比RestTemplate更优雅简单地处理了调用。

### 启动功能

在对应服务启动类上添加**@EnableFeignClients**

### 远程调用

```java
@FeignClient("stores")  // 指定调用的微服务名称（注册中心中的服务ID）
public interface StoreClient {
	// 定义HTTP请求（路径、方法等）
    @GetMapping("/stores")  
    List<Store> getStores();
	//注意，此处和MVC有点不同。Feign中的@Get和Post是发送而不是接收
    @PostMapping("/stores")
    Store createStore(Store store);
}
```

### FallBack兜底回调

需要整合下面的Sentinel实现[导入Maven + 开启熔断]，当远程调用失败时，能够返回兜底数据

```java
//需要传入实现fallback的类
@FeignClient(value = "service-product",fallback = ProductFeignClientFallback.class) 
public interface ProductFeignClient {
    @GetMapping("/product/{id}")
 	Product getProductById(@PathVariable("id") Long id);
}
```

```java
//实现fallback的类
@Component
 public class ProductFeignClientFallback implements ProductFeignClient {
    @Override
 	public Product getProductById(Long id) {
 	System.out.println("兜底回调....");
 	Product product = new Product();
	product.setId(id);
 	product.setPrice(new BigDecimal("0"));
 	return product;
 	}
}
```



## Sentinel-服务保护（服务熔断）

### 资源与规则

为资源设立规则，比如某个web接口的流量控制，熔断降级，系统保护，来源控制，热点参数。因此需要定义资源和规则，如果用户访问资源时就判断是否符合规则，若不符合则看是否有兜底处理/兜底回调。

使用:`@SentinelResource`定义资源，可在Sentinel网页设置流量控制等服务。

当对资源进行规则配置后，若不符合规则的请求就会异常。Sentinel有默认的异常处理，我们也可以通过对异常进行重写来实现更合理的返回机制。

![](C:\Users\HP\Documents\Obsidian Vault\Summary\img\微服务Sentinel异常处理.png)

### 流控模式

- 直接策略：流量直接访问资源A
- 链路策略：访问资源C，通过A访问的不限流，通过B访问的限流。（比如秒杀系统）
- 关联策略：关联读写。优先写入资源，当写入频繁时读取限流。

### 流控效果

- 快速失败：其他请求被丢弃
- WarmUP：逐步增加直至阈值
- 匀速排队：流量进入队列，超时会被丢弃

### 熔断降级

因为微服务之间的调用具有放大作用，任何一方的不稳定都可能引发雪崩。因此出现熔断降级切断不稳定调用，一般用于调用方。

引入断路器：
![微服务Sentinel断路器原理](C:\Users\HP\Documents\Obsidian Vault\Summary\img\微服务Sentinel断路器原理.png)

原理：断路器具有open，closed，half-open三个状态。

- 断路器默认为closed：设置流量和规则，若符合某种规则则断路器open，断路器open时，就不执行调用直接返回失败
- 断路器open时将会有一个timeWindow，也就是熔断时长。当超过熔断时长后变为half-open
- 断路器half-open时，会先放一个调用进来测试，若成功则断路器closed，失败则继续open，熔断时长重新计算

配置熔断能够直接进行兜底回调，而不用重走大概率失败的路。这样能使得需要大量被调用的服务更稳定。

### 热点规则

实现更加精确的管控。可以通过Sentinel页面中对某个参数的访问阈值设置来实现。
```
//例子
普通用户->QPS:1
VIP用户->QPS:10
保密信息->QPS:0
```

***Sentinel的配置默认保存在内存中，其实我们可以将其存在Nacos。***

***小结：配置+兜底回调一定要做好，尤其是兜底回调的工作量可能非常大***



## Gateway 网关

原来：前端需要记住所有微服务的地址
现在：前端只需要将请求发送给网关，让网关和注册中心连接，获取所有服务信息。同时还可以和Sentinel等等联动，共同保障服务稳定。

==TODO：断言机制（用到时再进行书写）==

![微服务Gateway断言机制](C:\Users\HP\Documents\Obsidian Vault\Summary\img\微服务Gateway断言机制.png)

基础规则：

1. 路由规则（id，url，predicate断言规则）
2. Filter过滤器[传入和传出]



## Seata-分布式事务

保证多个数据库连接一起提交回滚，保证数据一致性。

将原来的多个事务性操作分别写到对应的FeignClient接口中
（一般写在和service同级的feign文件夹中）

- TC事务协调者：需要驱动全局事务和分支事务的状态
- TM事务管理器：定义全局事务的范围，即调用者
- RM资源管理器：管理分支事务的资源，需要和TC进行交互

### 步骤

1. 安装Seata服务器并启动
2. 引入相应配置文件，以及在数据库创建undo_log表（看文档）
3. 为全局事务，即调用入口方法标注`@GlobalTransactional`注解即可，其余不用动。

### 原理

#### 第一阶段：本地事务

各微服务执行修改之前先查一遍获得前镜像，执行修改后又创建一次后镜像，并一起插入undo_log日志。

注册分支事务，并**申请记录的全局锁**。（防止并发数据异常）

将业务数据 + undo_log日志上传到数据库

汇报上传状态

#### 第二阶段：（如果成功）分支提交

TC发起提交请求，那么各微服务添加异步请求将undo_log删除

#### 第二阶段：（如果失败）分支回滚

TC发起回滚请求，各微服务找到undo_log
先进行后镜像校验，看当前数据和后镜像（修改后的值）是否一致。
如果不一致说明被外界修改了（比如手动修改等等），那么需要配置响应策略。
如果一致则直接回滚数据至前镜像状态，也就是数据修改前的样子。修改完成后删除undo_log

最后释放数据的全局锁。

#### 四种事务模式

1. AT模式（即上述原理描述阶段。默认）
2. XA：第一阶段并不提交数据，性能比较低下
3. TCC：全手动模式，需要手动实现各阶段代码（比如夹杂非数据库操作，比如上传文件-发送邮件-确定）
4. Saga：长事务模式，使用消息队列比Seata更适合。



## 总结

1. 为什么需要分布式和微服务-->应用与用户量庞大，单服务器已无法支持
2. 微服务怎么分-->注册中心/配置中心
3. 微服务之间如何调用-->OpenFeign
4. 调用链如何维护-->Sentinel
5. 微服务的统一入口-->Gateway
6. 微服务的出口，如何协调各微服务的数据事务-->Seata
