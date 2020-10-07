---
layout:         page
title:          【Spring】Introduction to Spring Data MongoDB
date:           2020-09-15
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

> 原文地址：[Introduction to Spring Data MongoDB](https://www.baeldung.com/spring-data-mongodb-tutorial)

#### MongoTemplate and MongoRepository
**MongoTemplate** 依据Spring中标准的 **template** 模式，提供了一些访问底层持久化引擎的基本的、可直接使用的API。

**MongoRepository** 基于众所周知的所有Spring Data项目中的数据访问模式，遵循Spring的 Data-centric 方式，提供了更加灵活、复杂的API。

当然，上述任何一种方式，都需要事先添加响应的依赖项。在Spring框架中访问MongoDB，需要添加如下依赖项：
```java
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-mongodb</artifactId>
    <version>3.0.3.RELEASE</version>
</dependency>

```
在Spring Boot中，也有MongoDB的起步依赖：
```java
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>

```

#### Configuration for MongoTemplate
配置MongoTemplate的方式之一是扩展AbstractMongoClientConfiguration ，但是这里我们先通过直接配置的方式来获取MongoTemplate组件。
```java
@Configuration
public class SimpleMongoConfig {
  private MongoDBProperty mongoDBProperty;

  @Autowired
  public void setEnvConfig(MongoDBProperty mongoDBProperty) {
    this.mongoDBProperty = mongoDBProperty;
  }

  @Bean
  public MongoClient mongo() {
    ConnectionString connectionString = new ConnectionString(mongoDBProperty.getMongoDBUrl());
    MongoClientSettings mongoClientSettings = MongoClientSettings.builder()
            .applyConnectionString(connectionString)
            .build();

    return MongoClients.create(mongoClientSettings);
  }

  @Bean(name = "testMongoDBTemplate")
  public MongoTemplate getTestMongoTemplate() throws Exception {
    return new MongoTemplate(mongo(), mongoDBProperty.getTestDBName());
  }
}

```
如上面配置代码所示，这里通过mongoDBProperty从application-xxxx.properties配置文件中读取MongoDB的连接以及将要访问的数据库名。然后创建了MongoClient、MongoTemplate两个Bean。

#### Data Access with MongoTemplate
接下来看看如何通过MongoTemplate来访问MongoDB数据库。

首先，我们通过set依赖注入获取上面配置好的MongoTemplate:
```java
private MongoTemplate mongoTemplate;

@Autowired
public void setMongoTemplate(@Qualifier("testMongoDBTemplate") MongoTemplate mongoTemplate) {
    this.mongoTemplate = mongoTemplate;
}

```

接下来，定义好实体：
```java

@Document(collection = "customer")
public class Customer {
    @MongoId
    private int id;
    private String name;

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}

```
上面通过在实体类上的注解@Document指定了集合名称，以及通过@MongoId指定了集合中文档的的"_id"属性字段。

接下来就是操作MongoDB了：
```java
Customer c = new Customer();
c.setId(123);
c.setName("Tommy");
mongoTemplate.insert(c);

```
上面的insert操作，会创建数据库test以及集合customer（因为之前它们都不存在），接着向集合中添加了一个文档。该文档有三个属性：
```java
_id: 123,
name: Tommy,
_class: com.example.mmdomain.Customer
```
这个_class字段，是持久化引擎添加的属性，用于后续从MongoDB中反序列化出Customer示例。

上面的示例通过 **insert** 操作插入一条文档。实际上，大部分情况下我们需要先判断集合中是否存在此文档，然后再根据实际情况来插入或者更新文档。而 **save** 操作本身就带有
save-or-update 语义：如果id存在，save执行一个update操作；否则，它执行一个insert操作。

下面是查询MongoDB的示例代码：
```java
Customer c = mongoTemplate.findById(customerId, Customer.class);
...
```

正如前面已经提到过的，MongoTemplate提供了一些基本的访问操作（当然不只是上面代码中的这些，其他的请参考官方文档）。如果业务需要更加灵活的访问方式，那么就可以使用另一种方式：MongoRepository。

#### Configuration for MongoRepository
我们可以为业务实体定义一个自定义的Repository，并扩展MongoRepository：
```java
public interface CustomerRepository extends MongoRepository<Customer, Long> {
  public List<Customer> findByNameIn(List<String> names);
}

```

我们不需要实现这个CustomerRepository的接口。接下来，我们同样要通过set依赖注入CustomerRepository：
```java
private CustomerRepository customerRepository;

@Autowired
public void setCustomerRepository(CustomerRepository customerRepository) {
    this.customerRepository = customerRepository;
}

```

这里有一点需要注意的是，MongoRepository是需要依赖一个名称为"mongoTemplate"的MongoTemplate Bean的。我们上面定义的MongoTemplate bean，名称不同。所以启动时，Spring无法自动装配MongoRepository。
我们可以把上面的MongoTemplate bean 名称改为"mongoTemplate"。不过，也可以通过上面提到过的另一种方式来配置MongoRepository：扩展AbstractMongoClientConfiguration。
```java
@Configuration
@EnableMongoRepositories(basePackages = "com.example.core.db.mgo.repository")
public class MongoRepositoryConfig extends AbstractMongoClientConfiguration {
  private MongoDBProperty mongoDBProperty;

  @Autowired
  public void setEnvConfig(MongoDBProperty mongoDBProperty) {
    this.mongoDBProperty = mongoDBProperty;
  }

  @Override
  protected String getDatabaseName() {
    return  mongoDBProperty.getTestDBName();
  }

  @Override
  public MongoClient mongoClient() {
    ConnectionString connectionString = new ConnectionString(mongoDBProperty.getMongoDBUrl());
    MongoClientSettings mongoClientSettings = MongoClientSettings.builder()
            .applyConnectionString(connectionString)
            .build();

    return MongoClients.create(mongoClientSettings);
  }
}

```

先看看AbstractMongoClientConfiguration的源码：
```java

@Configuration(
  proxyBeanMethods = false
)
public abstract class AbstractMongoClientConfiguration extends MongoConfigurationSupport {
  public AbstractMongoClientConfiguration() {
  }

  public MongoClient mongoClient() {
    return this.createMongoClient(this.mongoClientSettings());
  }

  @Bean
  public MongoTemplate mongoTemplate(MongoDatabaseFactory databaseFactory, MappingMongoConverter converter) {
    return new MongoTemplate(databaseFactory, converter);
  }

  ...
}

```
从源码中可以看到，AbstractMongoClientConfiguration会创建一个名为"mongoTemplate"的 MongoTemplate bean，因此我们的配置类中才不需要再明确配置一个MongoTemplate bean。

其次，上面的配置类MongoRepositoryConfig前面多了一行注解：
```java
@EnableMongoRepositories(basePackages = "com.example.core.db.mgo.repository")

```
这个注解的作用就是表示此配置类是作用于com.example.core.db.mgo.repository包里面的MongoRepository的。因此，如果我们想要定义多个自定义MongoRepository，并且需要让它们访问不同的MongoDB数据库，
那就需要将这多个MongoRepository分开在不同的包中，并且在各自的配置类中通过注解 **@EnableMongoRepositories** 来配置各自MongoRepository所在的包。

#### Data Access with MongoRepository
MongoRepository当然也包含与MongoTemplate相同的insert,save,upsert等等操作。
我们这里通过自定义的CustomerRepository的findByNameIn方法来根据多个名字查找对应的多个Customer实例：
```java
List<String> names = new ArrayList<>();
names.add("Jimmy");
names.add("Tommy");

List<Customer> customers = customerRepository.findByNameIn(names);
return customers;

```
上面的代码从MongoDB中查找名字为"Tommy"或者"Jimmy"的消费者数据，返回结果如下：
```javascript
[
  {
    id: 1234,
    name: "Tommy",
  },
  {
    id: 12345,
    name: "Tommy",
  },
  {
    id: 123456,
    name: "Jimmy",
  },
]

```
上述自定义的CustomerRepository的findByNameIn方法名称不是随意的，需要遵循Spring的一个关键字，具体的关键字请参考[点击这里](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#reference)的“Table 3. Supported keywords inside method names”。

