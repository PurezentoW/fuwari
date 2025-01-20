---
title: jooq.md
published: 2025-01-20
description: ''
image: ''
tags: []
category: ''
draft: false 
lang: ''
---
# JOOQ



jOOQ（**Java Object Oriented Querying**）是一个开源框架，它可以把数据库模型的基本信息，比如表名，字段名自动生成相应的Java类；并在此基础上提供了一整套数据处理的API。

## 简单对比

### 1. 原生sql

```sql
SELECT * FROM kj_merchant WHERE city = '厦门市' ORDER BY create_time 
```

### 2. **MyBatis**

MyBatis 是一个半自动化的ORM框架，需要手动编写SQL语句或通过XML配置。

#### 方式1：XML配置

在 `Mapper.xml` 中定义SQL：

```xml
<select id="selectMerchantsByCity" resultType="KjMerchant">
    SELECT * FROM kj_merchant
    WHERE city = #{city}
    ORDER BY create_time
</select>
```

在 `Mapper` 接口中调用：

```java
public interface KjMerchantMapper {
    List<KjMerchant> selectMerchantsByCity(@Param("city") String city);
}
```

#### 方式2：注解方式

直接在 `Mapper` 接口中使用注解：

```java
public interface KjMerchantMapper {
    @Select("SELECT * FROM kj_merchant WHERE city = #{city} ORDER BY create_time")
    List<KjMerchant> selectMerchantsByCity(@Param("city") String city);
}
```

------

### 3. **MyBatis-Plus**

MyBatis-Plus 是 MyBatis 的增强工具，提供了更简洁的API。

#### 方式1：使用 `QueryWrapper

```java
QueryWrapper<KjMerchant> queryWrapper = new QueryWrapper<>();
queryWrapper.eq("city", "厦门市")
            .orderByAsc("create_time");

List<KjMerchant> merchants = kjMerchantMapper.selectList(queryWrapper);
```

#### 方式2：使用 Lambda 表达式

```java
LambdaQueryWrapper<KjMerchant> lambdaQueryWrapper = new LambdaQueryWrapper<>();
lambdaQueryWrapper.eq(KjMerchant::getCity, "厦门市")
                  .orderByAsc(KjMerchant::getCreateTime);

List<KjMerchant> merchants = kjMerchantMapper.selectList(lambdaQueryWrapper);
```

------

### 4. **Hibernate**

Hibernate 是一个全自动化的ORM框架，支持通过HQL或Criteria API进行查询。

#### 方式1：使用 HQL

```java
String hql = "FROM KjMerchant WHERE city = :city ORDER BY createTime";
Query<KjMerchant> query = session.createQuery(hql, KjMerchant.class);
query.setParameter("city", "厦门市");

List<KjMerchant> merchants = query.getResultList();
```

#### 方式2：使用 Criteria API

```java
CriteriaBuilder cb = session.getCriteriaBuilder();
CriteriaQuery<KjMerchant> cq = cb.createQuery(KjMerchant.class);
Root<KjMerchant> root = cq.from(KjMerchant.class);

cq.select(root)
  .where(cb.equal(root.get("city"), "厦门市"))
  .orderBy(cb.asc(root.get("createTime")));

List<KjMerchant> merchants = session.createQuery(cq).getResultList();
```

------

### 5. **jOOQ**

jOOQ 是一个以SQL为中心的框架，支持类型安全的SQL查询。

#### 方式1：使用 jOOQ 的 DSL

假设你已经通过 jOOQ 代码生成器生成了 `KjMerchant` 表和字段的Java类：

```java
// 导入生成的表类
import static com.example.generated.Tables.KJ_MERCHANT;

// 创建 DSLContext
DSLContext create = DSL.using(connection, SQLDialect.MYSQL);

// 执行查询
List<KjMerchantRecord> merchants = create.selectFrom(KJ_MERCHANT)
    .where(KJ_MERCHANT.CITY.eq("厦门市"))
    .orderBy(KJ_MERCHANT.CREATE_TIME.asc())
    .fetchInto(KjMerchantRecord.class);
```

#### 方式2：直接使用SQL

如果你不想使用代码生成器，也可以直接编写SQL：

```java
Result<Record> result = create.fetch("SELECT * FROM kj_merchant WHERE city = ? ORDER BY create_time", "厦门市");
List<KjMerchant> merchants = result.into(KjMerchant.class);
```

------

### 6. 总结

以下是四种框架的实现对比：

| 框架             | 实现方式                                                     |
| ---------------- | ------------------------------------------------------------ |
| **MyBatis**      | 手动编写SQL（XML或注解），适合需要精细控制SQL的场景。        |
| **MyBatis-Plus** | 提供更简洁的API（如 `QueryWrapper`），适合快速开发。         |
| **Hibernate**    | 使用HQL或Criteria API，适合面向对象的操作，隐藏SQL细节。     |
| **jOOQ**         | 类型安全的SQL查询，适合需要直接控制SQL但又希望避免手动拼接SQL的场景。 |

## 简介

[jOOQ](https://jooq.org/)，是一个ORM框架，利用其生成的Java代码和流畅的API，可以快速构建有类型约束的安全的SQL语句

jOOQ的核心优势是可以将数据库表结构映射为Java类，包含表的基本描述和所有表字段。通过jOOQ提供的API，配合生成的Java代码，可以很方便的进行数据库操作

生成的Java代码字段类型是根据数据库映射成的Java类型，在进行设置和查询操作时，因为是Java代码，都会有强类型校验，所以对于数据的输入，是天然安全的，极大的减少了SQL注入的风险

jOOQ的代码生成策略是根据配置全量生成，任何对于数据库的改动，如果会影响到业务代码，在编译期间就会被发现，可以及时进行修复。

## 配置

### Maven 配置

```xml
<properties>
    <jooq.version>3.14.15</jooq.version>
</properties>

<dependencies>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>8.0.18</version>
    </dependency>

    <!-- base jooq dependency -->
    <dependency>
        <groupId>org.jooq</groupId>
        <artifactId>jooq</artifactId>
        <version>${jooq.version}</version>
    </dependency>
</dependencies>

<plugins>
   <plugin>
        <groupId>org.jooq</groupId>
        <artifactId>jooq-codegen-maven</artifactId>
        <version>3.14.15</version> <!-- 请使用最新版本 -->
        <executions>
            <execution>
                <goals>
                    <goal>generate</goal>
                </goals>
            </execution>
        </executions>
        <configuration>
            <jdbc> <!-- 数据库连接信息 -->
                <driver>com.mysql.cj.jdbc.Driver</driver>
                <url>jdbc:mysql://localhost:3306/learn-jooq</url>
                <user>root</user>
                <password>root</password>
            </jdbc>
            <generator> <!-- 代码生成器配置 -->
                <database>
                    <name>org.jooq.meta.mysql.MySQLDatabase</name> <!-- 数据库类型 -->
                    <includes>.*</includes> <!-- 表名匹配规则 -->
                    <inputSchema>learn-jooq</inputSchema> <!-- 输入数据库的模式 -->
                </database>
                <generate>
                    <pojos>true</pojos> <!-- 启用POJO生成 -->
                    <daos>true</daos> <!-- 可选：启用DAO生成 -->
                </generate>
                <target>
                    <packageName>cc.wcxian.jooq.codegen</packageName> <!-- 生成代码的包名 -->
                    <directory>src/main/java</directory> <!-- 生成代码的目录 -->
                </target>
            </generator>
        </configuration>
   </plugin>
</plugins>
```

### 代码生成

代码生成的原理就是通过读取数据库的元数据，将其转换为Java代码，并生成指定的文件，存放到配置好的指定目录

jOOQ的生成代码的目标路径建议配置单独的子包，因为每次代码生成都是全量的，如果和其他业务代码混合在一起，会被生成器误删

```shell
# 通过此命令里可以调用 jooq-codegen-maven 插件进行代码生成
mvn jooq-codegen:generate
```

或者

![image-20250119163038565](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202501192010267.png)

代码生成器执行完成后，会生成以下目录:

```
├─src/main/java/.../codegen ---- // 生成路径
│ ├─tables --------------------- // 表定义目录
│ │ ├─S1User ------------------- // s1_user 表描述包含: 字段，主键，索引，所属Schema
│ │ └─records ------------------ // 表操作对象目录
│ │   └─S1UserRecord ----------- // s1_user 表操作对象，包含字段get,set方法
│ ├─DefaultCatalog ------------- // Catalog对象，包含Schema常量
│ ├─Indexes -------------------- // 当前数据库所有的所有常量
│ ├─Keys ----------------------- // 当前数据库所有表主键，唯一索引等常量
│ ├─LearnJooq ------------------ // 数据库`learn-jooq`常量，包含该库所有表描述常量
│ └─Tables --------------------- // 所有数据库表常量
```

## 写个接口

```java
@RestController
@RequestMapping("/jooq")
public class JooqController {

    @Autowired
    private DSLContext dslContext;

    @GetMapping("/select/{id}")
    public List<S1UserDTO> select(@PathVariable Integer id) {
        List<S1UserRecord> s1UserRecords = dslContext.selectFrom(S1_USER)
                .where(S1_USER.ID.eq(id))
                .fetch();
        List<S1UserDTO> dtos = s1UserRecords.stream().map(s1UserRecord -> {
            S1UserDTO dto = new S1UserDTO();
            dto.setId(s1UserRecord.getId());
            dto.setUsername(s1UserRecord.getUsername());
            dto.setEmail(s1UserRecord.getEmail());
            dto.setAddress(s1UserRecord.getAddress());
            dto.setCreateTime(s1UserRecord.getCreateTime());
            dto.setUpdateTime(s1UserRecord.getUpdateTime());
            return dto;
        }).collect(Collectors.toList());
        return dtos;
    }
}


```

```json
[
	{
		"id": 1,
		"username": "demo1",
		"email": "demo1@diamondfds.com",
		"address": "China Guangdong Shenzhen",
		"createTime": "2019-12-27T16:41:42",
		"updateTime": "2019-12-27T16:41:42"
	}
]
```

## CRUD

### Insert

```java
// 类SQL语法 insertInto 方法第一个参数通常是表常量
dslContext.insertInto(S1_USER, S1_USER.USERNAME, S1_USER.ADDRESS, S1_USER.EMAIL)
    .values("王呈现", "厦门", "aa@gmail.com")
    .values("小王", "泉州", "bb@gmail.com")
    .execute();

// newRecord() 方法标识添加一条记录，通过链式调用，支持批量插入
dslContext.insertInto(S1_USER)
    .set(S1_USER.USERNAME, "雷军")
    .set(S1_USER.EMAIL, "diamondfsd@gmail.com")
    .newRecord()
    .set(S1_USER.USERNAME, "马云")
    .set(S1_USER.EMAIL, "diamondfsd@gmail.com")
    .execute();


//Record API
S1UserRecord record = dslContext.newRecord(S1_USER);
record.setUsername("usernameRecord1");
record.setEmail("diamondfsd@gmail.com");
record.setAddress("address hello");
record.insert();

//批量插入
List<S1UserRecord> recordList = IntStream.range(0, 10).mapToObj(i -> {
    S1UserRecord s1UserRecord = new S1UserRecord();
    s1UserRecord.setUsername("usernameBatchInsert" + i);
    s1UserRecord.setEmail("diamondfsd@gmail.com");
    return s1UserRecord;
}).collect(Collectors.toList());
dslContext.batchInsert(recordList).execute();

//插入后获得自增主键
Integer userId = dslContext.insertInto(S1_USER,
    S1_USER.USERNAME, S1_USER.ADDRESS, S1_USER.EMAIL)
    .values("username1", "demo-address1", "diamondfsd@gmail.com")
    .returning(S1_USER.ID)
    .fetchOne().getId();
```

### Update

```java
//类SQL
dslContext.update(S1_USER)
    .set(S1_USER.USERNAME, "RUOK")
    .set(S1_USER.EMAIL, "diamondfsd@gmail.com")
    .where(S1_USER.ID.eq(347))
    .execute();

//Record API
S1UserRecord record = dslContext.newRecord(S1_USER);
record.setId(1);
record.setUsername("usernameUpdate-2");
record.setAddress("record-address-2");
record.update();
// 生成SQL:  update `learn-jooq`.`s1_user` set `learn-jooq`.`s1_user`.`id` = 1, `learn-jooq`.`s1_user`.`username` = 'usernameUpdate-2', `learn-jooq`.`s1_user`.`address` = 'record-address-2' where `learn-jooq`.`s1_user`.`id` = 1

//批量更新
S1UserRecord record1 = new S1UserRecord();
record1.setId(1);
record1.setUsername("batchUsername-1");
S1UserRecord record2 = new S1UserRecord();
record2.setId(2);
record2.setUsername("batchUsername-2");

List<S1UserRecord> userRecordList = new ArrayList<>();
userRecordList.add(record1);
userRecordList.add(record2);
dslContext.batchUpdate(userRecordList).execute();
```

### Select

```java
//单表查询
Result<Record> fetchResult = dslContext.select().from(S1_USER)
        .where(S1_USER.ID.in(1, 2)).fetch();
List<S1UserRecord> result = fetchResult.into(S1UserRecord.class);

//联表查询
esult<Record3<String, String, String>> record3Result =
        dslContext.select(S1_USER.USERNAME,
        S2_USER_MESSAGE.MESSAGE_TITLE,
        S2_USER_MESSAGE.MESSAGE_CONTENT)
        .from(S2_USER_MESSAGE)
        .leftJoin(S1_USER).on(S1_USER.ID.eq(S2_USER_MESSAGE.USER_ID))
        .fetch();
List<UserMessagePojo> userMessagePojoList = record3Result.into(UserMessagePojo.class);
//into方法可以将结果或结果集转换为任意类型，jOOQ会通过反射的方式，将对应的字段值填充至指定的POJO中。通过关联查询的结果集，可以使用此方法将查询结果转换至指定类型的集合。
```

### Delete

```java
//类SQL方式
int i = dslContext.delete(S1_USER).where(S1_USER.USERNAME.eq("王呈现")).execute();

//Record API
S1UserRecord record = dslContext.newRecord(S1_USER);
record.setId(2);
int deleteRows = record.delete();

//批量删除
S1UserRecord record1 = new S1UserRecord();
record1.setId(1);
S1UserRecord record2 = new S1UserRecord();
record2.setId(2);
dslContext.batchDelete(record1, record2).execute();

List<S1UserRecord> recordList = new ArrayList<>();
recordList.add(record1);
recordList.add(record2);
dslContext.batchDelete(recordList).execute();
```

## POJO

代码生成器可以配置在生成代码时，同时生成和表一一对应的POJO，只需要在生成器配置`generator`块中，加上相关配置即可:

```XML
<generator>
    <generate>
        <pojos>true</pojos>
    </generate>
    <!-- ...  -->
</generator>
```

我们如果要基于原有的某个POJO添加其他字段，那么我们可以自己创建一个和表名一致的类，然后继承该POJO对象，在添加上我们需要的字段，例如本篇代码实例中的`s2_user_message`表，在这里我们需要将这张表和`s1_user`表进行关联查询，为了是查出用户ID对应的用户名

```JAVA
public class UserMessagePojo extends S2UserMessage {
    private String username;

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }
}
```

## SpringBoot 配置

```xml
<generator>
    <strategy>
        <name>com.diamondfsd.jooq.learn.CustomGeneratorStrategy</name>
    </strategy>
    <generate>
        <pojos>true</pojos>
        <daos>true</daos>
        <interfaces>true</interfaces>
        <springAnnotations>true</springAnnotations>
    </generate>
</generator>
```

```java
class S1UserDaoTest extends BaseTest {
    @Autowired
    S1UserDao s1UserDao;

    Integer insertUserId = null;

    @Test
    public void findAll() {
        List<S1UserPojo> userAll = s1UserDao.findAll();
        Assertions.assertTrue(userAll.size() > 0);
    }

    @Test
    public void insert() {
        S1UserPojo s1UserPojo = new S1UserPojo();
        s1UserPojo.setUsername("hell username");
        s1UserDao.insert(s1UserPojo);
        Assertions.assertNotNull(s1UserPojo.getId());
        insertUserId = s1UserPojo.getId();
    }

    @Test
    public void findById() {
        S1UserPojo findById = s1UserDao.findById(1);
        Assertions.assertNotNull(findById);

        if (insertUserId != null) {
            S1UserPojo userPojo = s1UserDao.findById(insertUserId);
            Assertions.assertNull(userPojo);
        }
    }
}
```

JOOQ转SQL

插件 CodeTools

## 参考资料

[从零开始 - jOOQ 系列教程](https://jooq.diamondfsd.com/learn/section-1-how-to-start.html)

[《你不知道的 JAVA》博客系列 💘 掌握数据库 Simple CRUD 的方法之 JOOQ 实战上述的使用方式在数据 - 掘金](https://juejin.cn/post/7437023118151450639)

[国内的 Java 体系真的很落后吗？ - V2EX](https://www.v2ex.com/t/1103584#reply99)

[Java ORM 哪家强？10个ORM框架测试对比与选型建议_java orm框架-CSDN博客](https://blog.csdn.net/fishsoul/article/details/138664390)