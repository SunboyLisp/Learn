## Springboot3整合Redis&基础应用

### 1.概述

​	SpringBoot是一种用于构建Java应用程序的开发框架，Redis是一个高性能的键值存储数据库，常用于缓存、会话管理、消息队列等应用场景，本文将给大家介绍，基于最新的的SpringBoot3基础上如何集成Redis，并实现Redis基本应用操作。

环境准备

- Java17 或更高
- Redis，版本无要求

> [!NOTE]
>
> https://start.aliyun.com 这个aliyun的脚手架，可以用来创建低版本的项目

### 2.整合过程

a.maven依赖

```java
  <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>

        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>

        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>

        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>3.5.5</version>
        </dependency>

        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis-spring</artifactId>
            <version>3.0.3</version>
        </dependency>
      
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
      
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-annotations</artifactId>
        </dependency>
      
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-core</artifactId>
        </dependency>
```

b.yml依赖，注意创建好一个数据库

```yml
server:
  port: 9999
spring:
  data:
    redis:
      port: 6379
      host: localhost
  datasource:
    username: root
    password: 123456
    url: jdbc:mysql:///springboot
logging:
  level:
    com.lisp: debug		#com.lisp代表包名
```

c.Redis配置类

```java
@Configuration
@EnableCaching
public class RedisConfig {

    @Bean
    public RedisTemplate<String,Object> redisTemplate(RedisConnectionFactory factory){
        RedisTemplate<String,Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(factory);

        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new GenericJackson2JsonRedisSerializer());

        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashValueSerializer(new Jackson2JsonRedisSerializer<Object>(Object.class));

        return redisTemplate;
    }

    @Bean
    public RedisCacheManager redisCacheManager(RedisTemplate redisTemplate){
        RedisCacheWriter redisCacheWriter = RedisCacheWriter.nonLockingRedisCacheWriter(redisTemplate.getConnectionFactory());
        RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(redisTemplate.getKeySerializer()));

        return new RedisCacheManager(redisCacheWriter, redisCacheConfiguration);

    }
}
```



### 3.RedisTemplate操作

#### a.String

①普通字符串存储

```java
public void testString() {
    String key = "user:token:0001";
    //30, TimeUnit.MINUTES 设置有效时间 30分钟
    redisTemplate.opsForValue().set(key, UUID.randomUUID().toString(),30, TimeUnit.MINUTES);
    System.out.println(redisTemplate.opsForValue().get(key));
}
```

②计数器功能

```java
public void testString2() {
    String key = "article:A00001:viewCount";
    redisTemplate.opsForValue().increment(key);
    System.out.println(redisTemplate.opsForValue().get(key));
}
```

③对象存储

```java
public void testString3() {
   Map<String,Object> user = new HashMap<>();
   user.put("id","00001");
   user.put("name","张三");
   user.put("age",18);
   user.put("brithday",new Date(2005-1900,10,03));
   String key = "user:0001";
    //对象存储，会根据序列化配置转换成JSON存储
   redisTemplate.opsForValue().set(key,user);
   System.out.println(redisTemplate.opsForValue().get(key));
  }
```

#### b.Hsah

```java
//Hash是键值类型，类似与map结构，更加适合用来保存对象
public void testHash() {
    String key = "user:0001:cart";
    Map<String,Object> shoppingCart =new HashMap<>();
    shoppingCart.put("cartId","123456789");
    shoppingCart.put("userId","987654321");
    List<Map<String,Object>> items = List.of(
            Map.of( "itemId", "1","itemName","手机","price",999.99,"quantity", 1),
            Map.of( "itemId", "2", "itemName","笔记本电脑","price",1499.99,"quantity",2),
            Map.of( "itemId", "3","itemName","耳机","price",49.99,"quantity", 3)
    );
    shoppingCart.put("items",items);
    shoppingCart.put("totalAmount",3149.92);
    shoppingCart.put("creationTime","2046-03-07T10:00:00:00");
    shoppingCart.put("lastUpdateTime","2046-03-07T12:30:00");
    shoppingCart.put("status","未结账");
    redisTemplate.opsForHash().putAll(key,shoppingCart);
    System.out.println(redisTemplate.opsForHash().get(key, "items"));
}
```

#### c.set

set是Redis数据库中一种无序，不重复的数据结构，用于存储一组唯一的元素。

```java
public void testSet() {
    String key = "author:0001:fans";
    redisTemplate.opsForSet().add(key,"张三","李四","王五");
    System.out.println("粉丝量" + redisTemplate.opsForSet().size(key));
}
```

#### d.zset

Zset是一个有序且唯一的集合，每个元素都与一个浮点数分数相关联，使得集合中的元素可以根据分数进行排序，默认按照分数升序排序。

```java
public void testZset() {
    String key = "user:0001:friends";
    redisTemplate.opsForZSet().add(key,"张三",System.currentTimeMillis());
    redisTemplate.opsForZSet().add(key,"李四",System.currentTimeMillis());
    redisTemplate.opsForZSet().add(key,"王五",System.currentTimeMillis());
    //降序查询
    Set set = redisTemplate.opsForZSet().reverseRange(key, 0, -1);
    System.out.println(set);
}
```

#### e.List

list是Redis中一种简单的字符串列表，按照插入顺序排序，可实现简单的队列功能。

```java
@Test
public void testList() {
    String key = "order:queue";
    HashMap<String, Object> order1 = new HashMap<>();
    order1.put("orderId","1001");
    order1.put("userId","2001");
    order1.put("status","已完成");
    order1.put("amount",500.75);
    order1.put("creationTime","2024-03-07T09:30:0日");
    order1.put("lastUpdateTime","2024-03-07T10:45:00");
    order1.put("paymentMethod","在线支付");
    order1.put("shippingMethod","自提");
    order1.put("remarks","尽快处理");
    Map<String,Object> order2 = new HashMap<>();
    order2.put("orderId","1002");
    order2.put("userId","2002");
    order2.put("status","待处理");
    order2.put("amount",280.99);
    order2.put("creationTime","2024-03-07T11:00:00");
    order2.put("lastUpdateTime","2024-03-07T11:00:00");
    order2.put("paymentMethod","货到付款");
    order2.put("shippingMethod","快递配送");
    order2.put("remarks","注意保鲜");
    //程序A：订单队列接受订单请求
    //存入数据
    redisTemplate.opsForList().leftPushAll(key,order1);
    redisTemplate.opsForList().leftPushAll(key,order2);
    //程序B
    //获取并移除数据
    Object obj = redisTemplate.opsForList().rightPop(key);
    System.out.println(obj);
}
```

### 4.@RediHash注解

a.创建实体类

```java
@RedisHash
@Data
public class User {
    @Id
    private Integer id;
    private String name;
    private Integer age;
    private String phone;
}
```

b.创建接口

```java
public interface UserRedisRepository extends CrudRepository<User, Integer> {}
```

c.使用

```java
public void testRedisHash() {
    User user = new User();
    user.setId(100);
    user.setName("张三丰");
    user.setAge(18);
    user.setPhone("18813085588");
    //保存
    userRedisRepository.save(user);
    //读取
    Optional<User> redisUser = userRedisRepository.findById(100);
    System.out.println(redisUser);
    //更新
    user.setPhone("10086");
    userRedisRepository.save(user);
    Optional<User> redisUser1 = userRedisRepository.findById(100);
    System.out.println(redisUser1);
    //删除
    //userRedisRepository.deleteById(100);
    boolean exists = userRedisRepository.existsById(100);
    System.out.println(exists);
}
```

> [!IMPORTANT]
>
> @RedisHash注解用于将java对象映射到Redis的Hash数据结构中，使得对象的存储和检索变得更加简单。
>
> 使用关键步骤：
>
> 1.@RedisHash注解标注在实体类上
>
> 2.创建接口并继承CrudRepository



### 5.缓存管理注解

SpringBoot的缓存管理功能旨在帮助开发人员轻松地在应用程序中使用缓存，以提高性能和响应速度。它提供了一套注解和配置，使得开发人员可以在方法级别上进行缓存控制，并且支持多种缓存存储提供程序，如Cafeine、EhCache、Redis等。

| 注解        | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| @Cacheable  | 用于声明一个方法的返回值应该被缓存起来，以便下次相同的参数调用时可以直接返回缓存中的值，而不需要执行方法体,。 |
| @CachePut   | 用于更新缓存中的数据，它会在方法执行后，将返回值更新到缓存中 |
| @CacheEvict | 用于清除缓存中的数据，它可以根据条件清除指定的缓存项         |

#### 1.准备表

```mysql
CREATE TABLE product(
id INT AUTO_INCREMENT PRIMARY KEY,
name VARCHAR(255) NOT NULL,
description TEXT,
Price DECIMAL(10,2) NOT NULL,
stock INT NOT NULL
);
```

```mysql
NSERT INTO product(name,description,price,stock) VALUES
 ('iPhone 15','最新的iPhone型号',8999.99,100),
 ('三星Galaxy s24','旗舰安卓手机',7899.99,150),
 ('MacBook Pro','专业人士的强大笔记本电脑',15999.99,58),
 ('iPad Air','性能强劲的便携式平板电脑',5599.99,200),
 ('索尼playstation 6','下一代游戏机',4499.99,75);
```

#### 2.实体类

```java
@Data
@TableName
public class Product implements Serializable {

    @TableId
    private Integer id;
    private String name;
    private String description;
    private Double price;
    private Integer stock;
}
```

### 结束语