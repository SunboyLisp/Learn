# MybatisPlus---青青菜鸟课程笔记

### 1.特性

![image-20241203153942791](C:\Users\lisp\AppData\Roaming\Typora\typora-user-images\image-20241203153942791.png)

### 2.框架结构

![image-20241203153557381](C:\Users\lisp\AppData\Roaming\Typora\typora-user-images\image-20241203153557381.png)

### 3.使用步骤

#### a.导入依赖

```
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.2</version>
</dependency>
```

#### b.配置文件配置数据源

```
spring:
  datasource:
    username: root
    password: 123456
    url: jdbc:mysql:///springboot
```

#### c.实体类

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User implements Serializable {
    private Long id;
    private String name;
    private Integer age;
    private String email;
}
```

> [!IMPORTANT]
>
> public class User implements Serializable  注意这里要进行序列化



#### d.创建Mapper接口，继承BaseMapper

```
public interface UserMapper extends BaseMapper<User> {}
```

> [!IMPORTANT]
>
> 只要继承了这个接口，我们自己写的UserMapper就有对这个User实体的增删改查的功能。

#### e.添加注解

```
//第一种方式，在这里加这个注解
@Mapper
public interface UserMapper extends BaseMapper<User> {}

//第二种方式是在启动类上面加一个注解 @MapperScan("com.lisp.mapper")
@SpringBootApplication
public class MybatisPulsLearnApplication {
    public static void main(String[] args) {      SpringApplication.run(MybatisPulsLearnApplication.class, args);
    }
}
```

### 4.常用注解

#### @TableName

表明注解，标识实体类对应的表。即类名和表名的映射。

不加这个注解，默认数据库表中的表名和实体类名一样

在实体类上加这个注解，例如TableName("user_info"),代表这个实体类可对应user_info这个数据库表。

#### @Tableld

主键注解
其中，ldType主键类型说明

| 值          | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| AUTO        | 数据库 ID 白增                                               |
| NONE        | 无状态，该类型为未设置主键类型(注解里等于跟随全局，全局里约等于INPUT) |
| INPUT       | insert 前自行 set 主键值                                     |
| ASSIGN ID   | 分配 ID(主键类型为 Number(Long 和 Integer)或 String)(since 3.3.0),使用接口IdentifierGenerator 的方法 nextId (默认实现类为 DefaultIdentifierGenerator 雪花算法) |
| ASSIGN UUID | 分配 UUID,主键类型为 String(since 3.3.0),使用接口 IdentifierGenerator 的方法nextuuID (默认 default 方法) |

```java
//可以实现id自增（基于最大的id进行自增）
@TableId(type = IdType.AUTO)
private Long id;
```



#### @TableFileld

字段注解

| 属性   | 默认值 | 描述                 |
| ------ | ------ | -------------------- |
| value  | **     | 数据库字段名         |
| exist  | true   | 是否为数据库表字段   |
| select | true   | 是否进行 select 查询 |

```java
//可能会存在数据库表中字段名和实体类名不一样的情况，这个时候就需要用这个注解的value字段
@TableField(value = "email")
private String mail;
```

```Java
//这个属性在数据表中不存在
@TableFiled(exist="false")
private String status
```

```java
//这个属性在查询的时候不想查
@TableFiled(select="false")
private String age;
```

#### @TableLogic

表字段逻辑处理注解（逻辑删除）

| 属性   | 默认值 | 描述         |
| ------ | ------ | ------------ |
| value  | String | 逻辑未删除值 |
| delval | String | 逻辑删除值   |

> [!IMPORTANT]
>
> 数据量小的话比较适合这种

使用步骤：

1.yml

```yml
mybatis-plus :
  global-config:
    db-config:
      logic-delete-field: deleted          #全局逻辑删除的实体字段名,配置后可不用@TableLogic注解
      logic-delete-value: 1                #逻辑已删除
      logic-not-delete-value: 0            #逻辑未删除
```

2.entity

```java
@TableLogic
private String deleted;		//建议使用deleted，如果设置这个可以不写注解
```

#### @Version

乐观锁注解，当要更新一条记录的时候，希望这条记录没有被别人更新。

乐观锁实现方式:

> - 取出记录时，获取当前 version
> - 更新时，带上这个 version
> - 执行更新时，set version =newVersion where version = oldVersion
> - 如果 version 不对，就更新失败

使用步骤：

0.为表新增字段：version，balance

1.config

```Java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
    return interceptor;
}
```

2.entity

```Java
 @Version
 private Integer version;
```

3.断点测试

```Java
@SpringBootTest
public class LockerTest {

    @Resource
    private UserMapper userMapper;

    @Test
    public void testLockeA(){
        User user = userMapper.selectById(1);
        user.setBalance(user.getBalance() + 20);
        userMapper.updateById(user);
    }
    
    @Test
    public void testLockeB(){
        User user = userMapper.selectById(1);
        user.setBalance(user.getBalance() + 30);
        userMapper.updateById(user);
    }
}

```

## 5.查询

### a.分页查询

1.配置分页拦截器

```Java
//可以在配置类那种加上这个
interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
```

2.查询前设置分页参数

```Java
IPage page = new Page<>(2,2);
userMapper.selectPage(page,null);
System.out.println(page.getTotal());
page.getRecords().forEach(System.out::println);
```

### b.条件构造器

1.常规使用

```Java
//Wrapper 查询封装器
//查询年龄>20，姓赵，降序
QueryWrapper<User> wrapper = new QueryWrapper<>();
//有一个弊端，字段名可能会写错，这样的话发现时期比较晚
wrapper.gt("age",20);
wrapper.likeRight("name","赵");
wrapper.orderByAsc("id");
List<User> users = userMapper.selectList(wrapper);
users.forEach(System.out::println);
```

2.推荐使用

```Java
//Wrapper 查询封装器
//查询年龄>20，姓赵，降序
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.gt(User::getAge,20);
wrapper.likeRight(User::getName,"赵");
wrapper.orderByAsc(User::getId);
List<User> users = userMapper.selectList(wrapper);
users.forEach(System.out::println);
```

## 6.代码生成器

##### a.导入依赖

```Java
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-generator</artifactId>
    <version>3.5.2</version>
</dependency>
<dependency>
    <groupId>org.freemarker</groupId>
    <artifactId>freemarker</artifactId>
</dependency>
```

##### b.代码生成器代码

①快速生成（推荐使用这个）

```java 
public class CodeGenerator {
    public static void main(String[] args) {
        String url = 
        String username = 
        String password = 
        String author = 
        String outputDir = 
        String basePackage = 
        String moduleName = 
        String mapperLocation = 
        String tableName = 
        FastAutoGenerator.create(url, username, password)
                .globalConfig(builder -> {
                    builder.author(author) // 设置作者
                            .enableSwagger() // 开启 swagger 模式
                            //.fileOverride() // 覆盖已生成文件
                            .outputDir(outputDir); // 指定输出目录
                })
                .packageConfig(builder -> {
                    builder.parent(basePackage) // 设置父包名
                            .moduleName(moduleName) // 设置父包模块名
                            .pathInfo(Collections.singletonMap(OutputFile.xml, mapperLocation)); // 设置mapperXml生成路径
                })
                .strategyConfig(builder -> {
                    builder.addInclude(tableName) // 设置需要生成的表名
                            .addTablePrefix(tablePrefix); // 设置过滤表前缀
                })
                .templateEngine(new FreemarkerTemplateEngine()) // 使用Freemarker引擎模板，默认的是Velocity引擎模板
                .execute();
    }

}

```

②交互式生成

```Java
public static void main(String[] args) {
    FastAutoGenerator.create("url", "username", "password")
            // 全局配置
            .globalConfig((scanner, builder) -> builder.author(scanner.apply("请输入作者名称？")))
            // 包配置
            .packageConfig((scanner, builder) -> builder.parent(scanner.apply("请输入包名？")))
            // 策略配置
            .strategyConfig((scanner, builder) -> builder.addInclude(getTables(scanner.apply("请输入表名，多个英文逗号分隔？所有输入 all")))
                    .entityBuilder()
                    .enableLombok()
                    .addTableFills(
                            new Column("create_time", FieldFill.INSERT)
                    )
                    .build())
            // 使用Freemarker引擎模板，默认的是Velocity引擎模板
            .templateEngine(new FreemarkerTemplateEngine())
            .execute();
}

// 处理 all 情况
protected static List<String> getTables(String tables) {
    return "all".equals(tables) ? Collections.emptyList() : Arrays.asList(tables.split(","));
}
```

