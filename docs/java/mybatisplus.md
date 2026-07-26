# MyBatis-Plus

MyBatis 最佳搭档，只做增强不做改变，为简化开发、提高效率而生


- 官方网址：https://baomidou.com/


## 1. 快速入门

### 1.1	默认配置

```yaml
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/mp?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai&rewriteBatchedStatements=true
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: root
    password: 123456

logging:
  level:
    net.xzh: debug
    net.xzh.generator.mapper: debug
  pattern:
    dateformat: HH:mm:ss

mybatis-plus:
  # Mapper.xml 文件位置
  mapper-locations: classpath:/mapper/*Mapper.xml
  # 实体类扫描包（用于在XML中直接使用类名）
  type-aliases-package: net.xzh.generator.model
  global-config:
    db-config:
      # 主键策略：ASSIGN_ID 为雪花算法ID
      id-type: ASSIGN_ID
      # 逻辑删除字段名
      logic-delete-field: del_flag
      # 逻辑已删除值
      logic-delete-value: 1
      # 逻辑未删除值
      logic-not-delete-value: 0
  configuration:
    # 开启驼峰命名自动映射
    map-underscore-to-camel-case: true
```

### 1.2	常用注解

| 注解名称             | 描述                                                         | 关键属性与说明                                               |
| :------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| `@TableName`         | 作用于**实体类（类级别）**，用于指定实体类对应的数据库表名。 | `value`：表名<br>`schema`：schema名称<br>`autoResultMap`：是否自动构建resultMap（配合TypeHandler建议设为true）<br>`excludeProperty`：需排除的属性名数组。 |
| `@TableId`           | 作用于**实体类的主键字段**，标识主键并定义主键生成策略。     | `value`：主键列名<br>`type`：生成策略。（如：`AUTO`自增、`ASSIGN_ID`雪花算法（推荐）、`INPUT`手动输入、`ASSIGN_UUID`分配32位UUID。） |
| `@TableField`        | 作用于**实体类的普通字段**，映射非主键字段，控制插入/更新行为及类型处理。 | `value`：列名<br>`exist`：是否为表字段（false则忽略）<br>`fill`：自动填充策略（如：INSERT/UPDATE/INSERT_UPDATE）<br>`typeHandler`：自定义类型处理器<br>`jdbcType`：指定JDBC类型。 |
| `@TableLogic`        | 作用于**实体类的逻辑删除字段**，标识逻辑删除字段，删除操作转为UPDATE，查询自动追加未删除条件。 | `value`：未删除值（如"0"）<br>`delval`：删除后值（如"1"）。    |
| `@Version`           | 作用于**实体类的版本号字段**，标识乐观锁字段<br>更新时自动校验旧版本号并自增。 | 前置条件：需在配置中注册`OptimisticLockerInnerInterceptor`插件。 |
| `@EnumValue`         | 作用于**枚举类的具体属性上**，指定枚举在数据库中实际存储的值，MP自动双向转换。 | 标注在枚举属性（如`code`）上，数据库存储该属性的值，而非枚举的`name`。 |
| `@OrderBy`           | 作用于**实体类的字段上**，为内置SQL指定默认排序规则（未显式指定Wrapper排序时生效）。 | `asc`：是否升序（默认true）<br>`sort`：排序优先级（数字越小越靠前）。 |
| `@KeySequence`       | 作用于**实体类（类级别）**，指定Oracle等数据库中用于生成主键的序列名称。 | `value`：序列名称<br>`dbType`：数据库类型。                    |
| `@InterceptorIgnore` | 作用于**Mapper接口（方法或类上）**，执行SQL时忽略特定的全局拦截器插件。 | `tenantLine`：是否忽略多租户拦截器<br>`blockAttack`：忽略防全表更新/删除攻击拦截器<br>`illegalSql`：忽略非法SQL拦截器。 |


## 2. 核心功能


### 2.1	条件构造器

![](../../assets/_images/java/mybatisplus/1.png)

1. QueryWrapper查询风格

```java
@Test
void testQueryWrapper() {
	// 1.构建查询条件
	QueryWrapper<User> wrapper = new QueryWrapper<User>()
			.select("id", "username", "info", "balance")
			.like("username", "o")
			.ge("balance", 1000);
	// 2.查询
	List<User> users = userMapper.selectList(wrapper);
	users.forEach(System.out::println);
}
```

2. LambdaQueryWrapper查询风格

```java
@Test
void testLambdaQueryWrapper() {
	// 1.构建查询条件
	LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<User>()
			.select(User::getId, User::getUsername, User::getInfo, User::getBalance)
			.like(User::getUsername, "o")
			.ge(User::getBalance, 1000);
	// 2.查询
	List<User> users = userMapper.selectList(wrapper);
	users.forEach(System.out::println);
}
```

3. UpdateWrapper更新风格

```java
@Test
void testUpdateByQueryWrapper() {
	// 1.要更新的数据
	User user = new User();
	user.setBalance(2000);
	// 2.更新的条件
	QueryWrapper<User> wrapper = new QueryWrapper<User>().eq("username", "jack");
	// 3.执行更新
	userMapper.update(user, wrapper);
}

@Test
void testUpdateWrapper() {
	List<Long> ids = List.of(1L, 2L, 4L);
	UpdateWrapper<User> wrapper = new UpdateWrapper<User>()
			.setSql("balance = balance - 200")
			.in("id", ids);
	userMapper.update(null, wrapper);
}
```

### 2.2	自定义SQL

```java
@Test
void testCustomSqlUpdate() {
	// 1.更新条件
	List<Long> ids = List.of(1L, 2L, 4L);
	int amount = 200;
	// 2.定义条件
	QueryWrapper<User> wrapper = new QueryWrapper<User>().in("id", ids);
	// 3.调用自定义SQl方法
	userMapper.updateBalanceByIds(wrapper, amount);
}
```

```java
public interface UserMapper extends BaseMapper<User> {
    void updateBalanceByIds(@Param(Constants.WRAPPER) QueryWrapper<User> wrapper, @Param("amount") int amount);
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="net.xzh.mp.mapper.UserMapper">
    <update id="updateBalanceByIds">
        UPDATE user SET balance = balance - #{amount} ${ew.customSqlSegment}
    </update>
</mapper>

```

### 2.4 静态工具

![](../../assets/_images/java/mybatisplus/2.png)


```java
@Test
void testDbGet() {
    User user = Db.getById(1L, User.class);
    System.out.println(user);
}

@Test
void testDbList() {
    // 利用Db实现复杂条件查询
    List<User> list = Db.lambdaQuery(User.class)
            .like(User::getUsername, "o")
            .ge(User::getBalance, 1000)
            .list();
    list.forEach(System.out::println);
}

@Test
void testDbUpdate() {
    Db.lambdaUpdate(User.class)
            .set(User::getBalance, 2000)
            .eq(User::getUsername, "Rose");
}
```

## 3. 插件

### 3.1	分页插件

PaginationInnerInterceptor

### 3.2	乐观锁

OptimisticLockerInnerInterceptor

###	3.3	多租户插件

TenantLineInnerInterceptor

###	3.4	数据权限插件

DataPermissionInterceptor 

### 3.5	动态表名插件

DynamicTableNameInnerInterceptor

### 3.6	数据变动记录插件

DataChangeRecorderInnerInterceptor
