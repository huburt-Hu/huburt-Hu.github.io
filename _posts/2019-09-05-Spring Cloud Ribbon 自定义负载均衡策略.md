---
layout:     post
title:      Spring Cloud Ribbon 自定义负载均衡策略
date:       2019-09-05
author:     胡奚冰
header-img: img/post-bg-coffee.jpeg
catalog: 	 true
tags:
    - Java
    - Spring Cloud
---


## 原由

公司项目使用Spring Cloud微服务架构，随着服务的增加，开发调试变得有些麻烦。有些同事的电脑配置不高，无法在本地启动这么多的服务。公司有自己的dev环境，对于开发当前修改的服务可以直接注册到dev环境，使用其他未修改的服务，如Eureka，config等。但是，如果这个时候有前端正在dev调试，则会出现网关转发到本地开发中的服务，出现异常。

出现上述情况的原因是因为Ribbon默认负载均衡策略是轮询，当一个服务出现多个实例的时候，网关转发或者服务消费时就会采用Ribbon的负载均衡策略，出现指向开发本地实例的情况。

知道原因之后解决方法就呼之欲出了：自定义负载均衡策略，使dev环境中的微服务消费或转发都指定到固定dev环境中服务，不让其指定到开发本地起的服务。

## 实现策略

`com.netflix.loadbalancer.IRule`是Ribbon的负载均衡策略接口：
```
public interface IRule{
    /*
     * choose one alive server from lb.allServers or
     * lb.upServers according to key
     * 
     * @return choosen Server object. NULL is returned if none
     *  server is available 
     */

    public Server choose(Object key);
    
    public void setLoadBalancer(ILoadBalancer lb);
    
    public ILoadBalancer getLoadBalancer();    
}
```

Ribbon自带有几种策略实现：

+ RandomRule：随机选取负载均衡策略，随机Random对象，在所有服务实例中随机找一个服务的索引号，然后从上线的服务中获取对应的服务。
+ RoundRobinRule：线性轮询负载均衡策略。
+ WeightedResponseTimeRule：响应时间作为选取权重的负载均衡策略，根据平均响应时间计算所有服务的权重，响应时间越短的服务权重越大，被选中的概率越高。刚启动时，如果统计信息不足，则使用线性轮询策略，等信息足够时，再切换到WeightedResponseTimeRule。
+ RetryRule：使用线性轮询策略获取服务，如果获取失败则在指定时间内重试，重新获取可用服务。
+ ClientConfigEnabledRoundRobinRule：默认通过线性轮询策略选取服务。通过继承该类，并且对choose方法进行重写，可以实现更多的策略，继承后保底使用RoundRobinRule策略。
+ BestAvailableRule：继承自ClientConfigEnabledRoundRobinRule。从所有没有断开的服务中，选取到目前为止请求数量最小的服务。
+ PredicateBasedRule：抽象类，提供一个choose方法的模板，通过调用AbstractServerPredicate实现类的过滤方法来过滤出目标的服务，再通过轮询方法选出一个服务。
+ AvailabilityFilteringRule：按可用性进行过滤服务的负载均衡策略，会先过滤掉由于多次访问故障而处于断路器跳闸状态的服务，还有并发的连接数超过阈值的服务，然后对剩余的服务列表进行线性轮询。
+ ZoneAvoidanceRule：本身没有重写choose方法，用的还是抽象父类PredicateBasedRule的choose。

其中没有我们需要的策略，那我们就自己实现一个IRule。我们参照默认的RoundRobinRule，继承`AbstractLoadBalancerRule `（实现了IRule的loadBanlancer相关方法）：

```

import com.netflix.client.config.IClientConfig;
import com.netflix.loadbalancer.AbstractLoadBalancerRule;
import com.netflix.loadbalancer.Server;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import java.util.List;

/**
 * @author hubert
 * @version 1.0
 * @date 2019/9/5 上午10:22
 */
@Slf4j
public class MyRule extends AbstractLoadBalancerRule {
    @Override
    public void initWithNiwsConfig(IClientConfig iClientConfig) {
    }

    @Override
    public Server choose(Object o) {
        log.info("key:" + o);
        List<Server> allServers = getLoadBalancer().getAllServers();
        log.info(allServers.toString());
        return allServers.get(0);
    }
}
```

我们需要实现choose方法来完成我们自己的策略，`getLoadBalancer()`可以获取当前服务的所有实例`Server`的信息，我们需要从中挑选一个作为choose方法的返回。这里就简单地返回列表第一个`Server`。


## 配置

自定义策略实现之后需要配置，我们要在服务调用方（使用@FeignClient注解的类的方法）进行配置。

### 简单配置

如果所有调用服务的策略是相同的，我们最简单的配置就是在`MyRule`类上添加`@Component`注解，让Spring发现并注入该类。Ribbon会优先使用我们实现的策略。

如果针对不同的服务需要不同的策略，则可以参考官方实例的配置。

###  `@RibbonClient`配置

这种方式需要在启动类中添加注解：
```
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
@RibbonClient(value = "service-hi", configuration = RuleConfig.class)
public class ServiceFeignApplication {
```
RuleConfig如下：
```
import com.netflix.loadbalancer.IRule;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author hubert
 * @version 1.0
 * @date 2019/9/5 上午10:40
 */
@Configuration
public class RuleConfig {
    @Bean
    public IRule ribbonRule() {
        return new MyRule();
    }
}
```
简单来说就是注入我们自己实现的IRule，然后配置给RioonClient。

`@RibbonClient`的`value/name`属性设置的是被调用的服务名（不是当前正在配置的服务名），也即使声明`@FeignClient`是指定的`value/name`。如果当前服务调用多个其他服务，可以用`@RibbonClient`给每个被调用服务设置不同的策略。

需要注意的是`RuleConfig`类需要放在启动类的上层（或者不同包名），避免Spring默认扫描到。否则会出现“简单配置”效果，即所有服务都使用这个策略，无法实现不同服务不同策略的效果。

### 配置文件配置

与`@RibbonClient`类似，可以为每个微服务单独配置策略。我们在yml配置文件中添加`<serverName>.ribbon.NFLoadBalancerRuleClassName`：

```
service-hi:
  ribbon:
    NFLoadBalancerRuleClassName: com.hubert.feign.MyRule
```

`<serverName>`也就是声明`@FeignClient`是指定的`value/name`。

## 实验

配置完成之后就是启动实验了，我们依次启动2个被调用服务，以及一个调用服务：
```
2019-09-05 14:31:45.408  INFO 1143 --- [ix-service-hi-2] com.hubert.feign.MyRule                  : key:null
2019-09-05 14:31:45.409  INFO 1143 --- [ix-service-hi-2] com.hubert.feign.MyRule                  : [192.168.31.244:8882, 192.168.31.244:8881]
```
发现日志打印，说明使用了我们自定义的策略，并且效果也是始终调用第一个服务。