---
layout:     post
title:      Spring Cloud Oauth2 + Security 填坑记
date:       2019-07-11
author:     胡奚冰
header-img: img/post-bg-coffee.jpeg
catalog: 	 true
tags:
    - Java
    - Spring Cloud
---

教程：

[Spring Cloud下基于OAUTH2认证授权的实现](https://wiselyman.iteye.com/blog/2379419)

[Spring cloud微服务实战——基于OAUTH2.0统一认证授权的微服务基础架构](https://blog.csdn.net/w1054993544/article/details/78932614)

[Spring Cloud OAuth2（一） 搭建授权服务](https://www.cnblogs.com/fp2952/p/8973613.html)

demo: 

https://gitee.com/xingfly/Spring-CloudJiYuZuulDeTongYiShouQuanRenZheng

https://gitee.com/log4j/pig

搜索到一些文章，就照着搭，但是由于本人使用Spring Cloud 是Finchley.RELEASE版本，而教程或者demo通常是Dalston.SR3版本的。由于版本差异出现了一些问题。

## 版本问题

### Dalston.SR3版本Redis集群Pipeline问题

Dalston.SR3版本spring-boot-starter-data-redis默认实现是Jedis，security oauth使用Redis存储token的实现类RedisTokenStore中使用Pipeline的功能。Jedis对于单节点可以支持Pipeline，但是集群则没有支持Pipeline，最终导致服务无法使用。
```
UnsupportedOperationException, Pipeline is currently not supported for JedisClusterConnection.
```
![JedisClusterConnection.png](https://upload-images.jianshu.io/upload_images/4657803-139f6b18d6e7b966.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

要解决这个问题，要么自己实现Jedis对Pipeline的支持，这比较复杂。
另一种就是使用lettuce代替Jedis，luttuce对于集群使用Pipeline有很好的支持。而且Dalston.SR3版本spring-boot-starter-data-redis也有相关的配置类，我们只需要引入luttuce的依赖：
```
        <dependency>
            <groupId>biz.paluch.redis</groupId>
            <artifactId>lettuce</artifactId>
            <version>4.4.2.Final</version>
        </dependency>
```
注意luttuce的版本与spring的对应关系，错误的版本也导致无法启动。查找的时候发现不同的lettuce，groupId有不同的，连里面的包名也不同。

然后配置redisConnectionFactory：
```
    @Primary
    @Bean
    public LettuceConnectionFactory redisConnectionFactory(RedisProperties redisProperties) {
        RedisProperties.Cluster cluster = redisProperties.getCluster();
        if (cluster != null) {
            RedisClusterConfiguration configuration = new RedisClusterConfiguration(cluster.getNodes());
            return new LettuceConnectionFactory(configuration);
        } else {
            LettuceConnectionFactory connectionFactory = new LettuceConnectionFactory(redisProperties.getHost(), redisProperties.getPort());
            connectionFactory.setPassword(redisProperties.getPassword());
            return connectionFactory;
        }
    }

    @Bean
    public TokenStore tokenStore(RedisConnectionFactory connectionFactory) {
        RedisTokenStore redisTokenStore = new RedisTokenStore(connectionFactory);
        redisTokenStore.setPrefix(Const.REDIS_PREFIX);
        return redisTokenStore;
    }
```

### Finchley.RELEASE版本 配置ClientDetailsServiceConfigurer时，secret需要使用passwordEncoder加密

```
.secret(passwordEncoder.encode("android"))
```

## 坑

各微服务单独做token校验时遇到了一些token传递的问题

### zuul网关过滤Authorization请求头

发起请求时，需要添加Authorization请求头设置token完成校验，直接访问微服务没有问题，但是经过zuul网关转发时发现返回401。
排查发现在经过zuul转发到微服务时丢失了Authorization请求头，查找资料才知道zuul默认会过滤一些敏感header。
如果需要将token信息传递给微服务，则需要配置zuul关闭默认过滤：
```
zuul:
  # 默认会过滤authorization、set-cookie、cookie、host、connection、content-length、content-encoding、server、transfer-encoding、x-application-context
  sensitive-headers:
```
这里冒号之后是空，表示没有需要过滤的header。

### 使用Feign进行服务消费时，没有将Authorization请求头传递给下游服务

Feign本质上是一个新的请求，与进入这个微服务的请求并不是同一个。如果我们需要token传递给下游微服务，需要自己取出token设置给新请求。
先定义一个Feign的请求拦截器：
```
@Component
public class FeignOauth2RequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate requestTemplate) {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        log.info("authentication:" + authentication);
        if (authentication != null && authentication.getDetails() instanceof OAuth2AuthenticationDetails) {
            OAuth2AuthenticationDetails auth2AuthenticationDetails = (OAuth2AuthenticationDetails) authentication.getDetails();
            String token = String.format("%s %s", auth2AuthenticationDetails.getTokenType(), auth2AuthenticationDetails.getTokenValue());
            log.info("feign header token:" + token);
            requestTemplate.header(HttpHeaderConst.AUTHORIZATION, token);
        }
    }
}
```
这里完成了取出token，设置的操作。注意`@Component`注解，然后将这个拦截器配置给FeignClient：
```
@Component
@FeignClient(value = ServiceNameConst.SERVICE_USER, fallback = AccessApiImpl.class,
        configuration = FeignOauth2RequestInterceptor.class
)
public interface AccessApi {
```
这样在构造Feign请求时会执行拦截器的操作，完成token的传递。

### SecurityContextHolder无法获取到token

当使用hystrix时，你会发现上面Feign的拦截器中并不能获取到token。问题原因来自hystrix隔离策略，默认是线程隔离，也就是说Feign的请求执行在一个新的线程中。而SecurityContextHolder获取token对象是通过ThreadLocal存储的，也即是说与线程绑定，因此无法在新线程中获取token。

解决方法是将hystrix隔离策略（strategy）修改为：SEMAPHORE。

> 参考
[Spring Cloud fegin 在oauth2 中无法传递授权信息](https://blog.csdn.net/u010176542/article/details/79122038)
[feign调用session丢失解决方案](https://blog.csdn.net/zl1zl2zl3/article/details/79084368)
[关于springcloud 使用oauth2 和 feign 的坑](http://blog.51cto.com/9795602/2177730)


## 扩展

[自定义oauth2验证失败出参](https://my.oschina.net/merryyou/blog/1819572)

[hasRole hasAuthority 区别](https://stackoverflow.com/questions/19525380/difference-between-role-and-grantedauthority-in-spring-security)

 [Spring Security OAuth2 授权失败（401) 问题整理](https://www.cnblogs.com/mxmbk/p/9782409.html)