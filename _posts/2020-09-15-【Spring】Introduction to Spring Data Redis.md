---
layout:         page
title:          【Spring】Introduction to Spring Data MongoDB
date:           2020-09-15
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

关于在Spring中使用Redis，这篇文档有比较详细的介绍：[Click Here](https://docs.spring.io/spring-data/data-redis/docs/current/reference/html/#redis:setup)。不过这里，我依然会直接依赖Spring Boot的起步依赖。除了依赖配置不同，其他的相关配置，其实都基本一样。

#### 依赖
因为我这里的项目是基于Spring Boot，所以项目配的是Spring Boot的起步依赖：
```java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

```

#### RedisTemplate
类似MongoTemplate, RedisTemplate 是 Spring Data Redis 提供出来的抽象客户端。只不过，RedisTemplate还把对Redis的访问操作根据操作类型不同，分别封装成了以下几种Operations接口：
```java
ValueOperations：简单K-V操作
SetOperations：set类型数据操作
ZSetOperations：zset类型数据操作
HashOperations：针对map类型的数据操作
ListOperations：针对list类型的数据操作

```
我们就可以通过这些Operations接口来访问Redis服务中不同类型的数据。

#### 1. Connecting to Redis
在Spring中使用Redis的第一步就是通过Spring IoC容器连接Redis。为了达到这个目的，我们需要一个Java连接器（connector）。不论底层使用什么库，我们只需使用Spring提供的那套API（“Spring Data Redis APIs”），其中包括 RedisConnection 和 RedisConnectionFactory 这两个接口，通过它们，我们可以获取到Redis的活动连接。

**RedisConnection** 提供了跟Redis通信的核心构建模块，因为正是它在处理与Redis后台服务的通信。同时，RedisConnection也自动将底层连接库的异常转化为一致的[Spring DAO 异常体系](https://docs.spring.io/spring-framework/docs/5.2.9.RELEASE/spring-framework-reference/data-access.html#dao-exceptions)。所以，我们可以在不同的connectors之间切换，而不需要对上层代码做任何改动。

**RedisConnectionFactory** 负责创建活动的RedisConnection 对象。

创建RedisTemplate时，需要传一个RedisConnectionFactory bean，因此首先我们就需要创建一个RedisConnectionFactory bean，至于创建RedisConnection对象的细节，就由框架内部实现了。

```java

// SimpleRedisConfig.java

private RedisProperty redisProperty;

@Autowired
public void setCoreConfig(RedisProperty redisProperty) {
  this.redisProperty = redisProperty;
}

@Bean(name = "CustomerLettuceConnectionFactory")
LettuceConnectionFactory getCustomerLettuceConnectionFactory() {
  /**
   * these methods have been deprecated.
   * We now need to configure using RedisStandaloneConfiguration
   */
//    factory.setHostName(redisHost);
//    factory.setPort(redisPort);

  RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration(
          redisProperty.getCustomerRedisHostName(),
          redisProperty.getCustomerRedisPort());
  String password = redisProperty.getCustomerRedisPassword();
  if (!StringUtils.isEmpty(password)) {
    redisStandaloneConfiguration.setPassword(RedisPassword.of(password));
  }
  redisStandaloneConfiguration.setDatabase(redisProperty.getCustomerRedisDB());
  LettuceConnectionFactory factory = new LettuceConnectionFactory(redisStandaloneConfiguration);

  return factory;
}

```

如上面代码所示，创建LettuceConnectionFactory对象时，传入一个RedisStandaloneConfiguration配置对象，该对象包含了Redis服务地址，端口，密码等等信息。实际上，我们可以传入不同的配置对象，来达到与不同的Redis服务通信的目的。例如，我们还可以给LettuceConnectionFactory构造方法传入一个RedisClusterConfiguration对象，来实现与Redis Cluster集群通信的目的。如：

```java
/**
   * Redis Cluster Connection
   */
@Bean(name = "CustomerClusterLettuceConnectionFactory")
LettuceConnectionFactory getCustomerClusterLettuceConnectionFactory() {
  List<String> clusterNodes = redisProperty.getCustomerClusterNodes();
  RedisClusterConfiguration redisClusterConfiguration = new RedisClusterConfiguration(clusterNodes);
  String password = redisProperty.getCustomerRedisPassword();
  if (!StringUtils.isEmpty(password)) {
    redisClusterConfiguration.setPassword(RedisPassword.of(password));
  }
  // TODO:
  // redisClusterConfiguration.setMaxRedirects();
  LettuceConnectionFactory factory = new LettuceConnectionFactory(redisClusterConfiguration);

  return factory;
}

```

有了上面的RedisConnectionFactory bean，就可以创建RedisTemplate bean。正如上面的两个不同的bean所示，RedisTemplate不管RedisConnectionFactory bean 创建的RedisConnection 对象是与Redis Master-Slave通信，还是与Redis Cluster通信，这些细节是底层Redis库的问题，RedisTemplate封装了一致性的Redis操作。

也就是说，我们使用RedisTemplate访问Redis时，如果需要从Master-Slave切换至Redis Cluster，或者反过来，只需要创建不同的RedisConnectionFactory bean（给它传入不同的Redis Configuration 对象）。上层业务代码是不需要做任何改动的。这就是RedisTemplate封装出的一致性操作的好处之一。

有了上面的RedisConnectionFactory bean之后，我们就可以来创建RedisTemplate bean了：
```java

/**
 * RedisAutoConfiguration:
 * @ConditionalOnMissingBean(
 *     name = {"redisTemplate"}
 *   )
 */
@Bean(name = "redisTemplate")
RedisTemplate< String, Customer> getCustomerRedisTemplate(@Qualifier("CustomerLettuceConnectionFactory") LettuceConnectionFactory lettuceConnectionFactory) {
  final RedisTemplate<String, Customer> template =  new RedisTemplate<>();
  template.setConnectionFactory(lettuceConnectionFactory);

  /**
   * 使用Jackson2JsonRedisSerializer来序列化和反序列化value（默认使用JDK的序列化方式）
   */
  Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Customer.class);
  ObjectMapper om = new ObjectMapper();

  /**
   * 指定要序列化的域，field,get和set,以及修饰符范围，ANY是都有包括private和public
   */
  om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.PUBLIC_ONLY);

  /**
   * 指定序列化输入的类型，类必须是非final修饰的，final修饰的类，比如String,Integer等会跑出异常
   */
  om.activateDefaultTyping(LaissezFaireSubTypeValidator.instance ,
          ObjectMapper.DefaultTyping.NON_FINAL,
          JsonTypeInfo.As.WRAPPER_ARRAY);
  jackson2JsonRedisSerializer.setObjectMapper(om);

  template.setValueSerializer(jackson2JsonRedisSerializer);
  template.setKeySerializer(new StringRedisSerializer() );
  template.setHashKeySerializer(new StringRedisSerializer());
  template.setHashValueSerializer(jackson2JsonRedisSerializer);

  return template;
}
```

如上面代码所示，getCustomerRedisTemplate bean方法，通过@Qualifier注解注入了"CustomerLettuceConnectionFactory" bean，而不是"CustomerClusterLettuceConnectionFactory" bean。因此它创建的RedisTemplate bean会负责与Redis Master-Slave通信。另外，上面代码里还设置了一些序列化和反序列化Redis数据（key和value）的规则。

另外，上面的RedisTemplate bean被命名为"redisTemplate"，这个名字当然可以自定义。只不过，框架的RedisAutoConfiguration类也有bean方法，并且在判断没有一个名称为"redisTemplate"的 bean的时候，会创建一个出来：
```java

@Configuration(
  proxyBeanMethods = false
)
@ConditionalOnClass({RedisOperations.class})
@EnableConfigurationProperties({RedisProperties.class})
@Import({LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class})
public class RedisAutoConfiguration {
  public RedisAutoConfiguration() {
  }

  @Bean
  @ConditionalOnMissingBean(
    name = {"redisTemplate"}
  )
  public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
    RedisTemplate<Object, Object> template = new RedisTemplate();
    template.setConnectionFactory(redisConnectionFactory);
    return template;
  }

  @Bean
  @ConditionalOnMissingBean
  public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory) throws UnknownHostException {
    StringRedisTemplate template = new StringRedisTemplate();
    template.setConnectionFactory(redisConnectionFactory);
    return template;
  }
}

```
同样，RedisAutoConfiguration还会注册一个stringRedisTemplate bean（如果它不存在），并且，这个bean方法也依赖RedisConnectionFactory对象。以我们上面的例子为例，我们没有创建一个stringRedisTemplate bean，却创建了两个RedisConnectionFactory bean（一个单点、一个Cluster）。这就有问题了，RedisAutoConfiguration的stringRedisTemplate  @Bean 方法无法判断出来应该用哪个RedisConnectionFactory bean的，因此就会报错：
```java
Parameter 0 of method stringRedisTemplate in org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration required a single bean, but 2 were found:
	- EntityLettuceConnectionFactory: defined by method ...
	- ABTestLettuceConnectionFactory: defined by method ...


Action:

Consider marking one of the beans as @Primary, updating the consumer to accept multiple beans, or using @Qualifier to identify the bean that should be consumed
```

如错误后的提示，解决这个问题就需要我们使用@Primary注解了，通过在某一个创建RedisConnectionFactory bean的 @Bean方法前面添加一个@Primary注解，来告诉框架应该优先用哪个RedisConnectionFactory bean。


#### 序列化、反序列化
如创建RedisTemplate的@Bean方法中代码所示，我们需要根据实际情况，设置Redis的Key 和 Value的序列化、反序列化方式：
```java

template.setValueSerializer(jackson2JsonRedisSerializer);
template.setKeySerializer(new StringRedisSerializer() );
template.setHashKeySerializer(new StringRedisSerializer());
template.setHashValueSerializer(jackson2JsonRedisSerializer);

```

#### Operations接口
上面已经提过，RedisTemplate将对Redis的数据操作封装为了几个Operations接口，上层业务代码需要通过这几个Operations接口来与Redis服务器通信。以Key-Value数据类型为例：
```java
/**
 * 对redis字符串类型数据操作
 *
 * @param redisTemplate
 * @return
 */
@Bean
public ValueOperations<String, Customer> getEntityValueOperations(@Qualifier("redisTemplate") RedisTemplate<String, Customer> redisTemplate) {
  return redisTemplate.opsForValue();
}

```
通过在SimpleRedisConfig配置类中再添加一个getEntityValueOperations @Bean方法，向 [IoC容器](https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/beans.html) 中注入一个ValueOperations<String, Customer>类型的bean。接着业务代码即可通过这个bean访问Redis。

#### Define a Redis Client
为了业务代码与底层Redis细节的隔离，最好再自定义一个Redis Client来封装Redis操作。如：
```java

@Component
public class CustomerRedisClient {
  private ValueOperations<String, Customer> valueOperations;

  @Autowired
  public void setValueOperations(ValueOperations<String, Customer> valueOperations) {
    this.valueOperations = valueOperations;
  }

  public void setCustomer(Customer c, int expire) {
    valueOperations.set("customer:" + c.getId(), c, expire, TimeUnit.SECONDS);
  }

  public Customer getCustomer(int id) {
    return valueOperations.get("customer:" + id);
  }
}
```

上面的示例中，通过 ValueOperations<String, Customer>类型的bean，CustomerRedisClient类型的bean对外提供了两个方法setCustomer和getCustomer。下面是一个简单的使用自定义的这个Redis Client的代码：
```java
@GetMapping("customer/{customerId}")
public Customer getCustomer(@PathVariable Integer customerId) {
    Customer c = customerRedisClient.getCustomer(customerId);
    if (c == null) {
        c = mongoTemplate.findById(customerId, Customer.class);
        customerRedisClient.setCustomer(c, 300);
    }

    return c;
}

```
上面这个简单的示例中，一个Controller的getCustomer方法先试图从Redis中拿出缓存数据作为响应，没有缓存时则从MongoDB中查询，并且设置缓存。

