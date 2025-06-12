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
监听配置变化：暂时不学，等开发实际阶段用。
namespace：命名空间，可以写开发、测试、生产
dataID：填名字.后缀
groupID：用微服务名称即可
```



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

