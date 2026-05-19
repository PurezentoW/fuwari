---
title: MapStruct
published: 2023-12-28
description: MapStruct的使用
image: 'https://cdn.wcxian.cc/img/20260326105308839.png'
tags: [JAVA]
category: '技术分享'
draft: false 
lang: zh-CN
---
# MapStruct的使用

> 在一个Java工程中会涉及到多种对象，po、vo、dto、entity、do、domain这些定义的对象运用在不同的场景模块中，这种对象与对象之间的互相转换，就需要有一个专门用来解决转换问题的工具。以往的方式要么是自己写转换器，要么是用Apache或Spring的**BeanUtils**来实现转换。无论哪种方式都存在明显的缺点，比如手写转换器既浪费时间， 而且在添加新的字段的时候也要进行方法的修改；而无论是 BeanUtils, BeanCopier 等都是使用反射来实现，效率低下并且仅支持属性名一致时的转换。

## 各大对象映射框架性能对比

![img.png](https://cdn.jsdelivr.net/gh/PurezentoW/PicGo/img/202312140015158.png)

### 反射

```java
/**
* Beanutils copyProperties 的反射
*/
private static void copyProperties(Object source, Object target, Class<?> editable, String... ignoreProperties)
            throws BeansException {
        // 检查source和target对象是否为null，否则抛运行时异常
        Assert.notNull(source, "Source must not be null");
        Assert.notNull(target, "Target must not be null");
        // 获取target对象的类信息
        Class<?> actualEditable = target.getClass();
        // 若editable不为null，检查target对象是否是editable类的实例，若不是则抛出运行时异常
        // 这里的editable类是为了做属性拷贝时限制用的
        // 若actualEditable和editable相同，则拷贝actualEditable的所有属性
        // 若actualEditable是editable的子类，则只拷贝editable类中的属性
        if (editable != null) {
            if (!editable.isInstance(target)) {
                throw new IllegalArgumentException("Target class [" + target.getClass().getName() +
                        "] not assignable to Editable class [" + editable.getName() + "]");
            }
            actualEditable = editable;
        }
        // 获取目标类的所有PropertyDescriptor，getPropertyDescriptors这个方法请看下方
        PropertyDescriptor[] targetPds = getPropertyDescriptors(actualEditable);
        List<String> ignoreList = (ignoreProperties != null ? Arrays.asList(ignoreProperties) : null);

        for (PropertyDescriptor targetPd : targetPds) {
            // 获取该属性对应的set方法
            Method writeMethod = targetPd.getWriteMethod();
            // 属性的set方法存在 且 该属性不包含在忽略属性列表中
            if (writeMethod != null && (ignoreList == null || !ignoreList.contains(targetPd.getName()))) {
                // 获取source类相同名字的PropertyDescriptor, getPropertyDescriptor的具体实现看下方
                PropertyDescriptor sourcePd = getPropertyDescriptor(source.getClass(), targetPd.getName());
                if (sourcePd != null) {
                    // 获取对应的get方法
                    Method readMethod = sourcePd.getReadMethod();
                    // set方法存在 且 target的set方法的入参是source的get方法返回值的父类或父接口或者类型相同
                    // 具体ClassUtils.isAssignable()的实现方式请看下面详解
                    if (readMethod != null &&
                            ClassUtils.isAssignable(writeMethod.getParameterTypes()[0], readMethod.getReturnType())) {
                        try {
                            //get方法是否是public的
                            if (!Modifier.isPublic(readMethod.getDeclaringClass().getModifiers())) {
                                //暴力反射，取消权限控制检查
                                readMethod.setAccessible(true);
                            }
                            //获取get方法的返回值
                            Object value = readMethod.invoke(source);
                            // 原理同上
                            if (!Modifier.isPublic(writeMethod.getDeclaringClass().getModifiers())) {
                                writeMethod.setAccessible(true);
                            }
                            // 将get方法的返回值 赋值给set方法作为入参
                            writeMethod.invoke(target, value);
                        }
                        catch (Throwable ex) {
                            throw new FatalBeanException(
                                    "Could not copy property '" + targetPd.getName() + "' from source to target", ex);
                        }
                    }
                }
            }
        }
    }
```





## 性能对比

```java
 public static void main(String[] args) {
        StopWatch stopWatch = new StopWatch("对象拷贝性能测试");

        stopWatch.start("Mapstruct");
        for (int i = 0; i < 10000000; i++) {
            User user = new User();
            user.setAge(99L);
            user.setName("Xx。");
            user.setId(11L);
            UserVo vo = UserConverter.INSTANCE.user2UserVo(user);
        }
        stopWatch.stop();
        System.out.println("Mapstruct执行时间：" + stopWatch.getLastTaskTimeMillis() + " ms");

        stopWatch.start("Beanutils");
        for (int i = 0; i < 10000000; i++) {
            User user = new User();
            user.setAge(99L);
            user.setName("Xx。");
            user.setId(11L);
            UserVo vo = new UserVo();
            BeanUtils.copyBeanProp(vo,user);
        }
        stopWatch.stop();
        System.out.println("Beanutils执行时间：" + stopWatch.getLastTaskTimeMillis() + " ms");

        stopWatch.start("Hutools");
        for (int i = 0; i < 10000000; i++) {
            User user = new User();
            user.setAge(99L);
            user.setName("Xx。");
            user.setId(11L);
            UserVo vo = new UserVo();
            cn.hutool.core.bean.BeanUtil.copyProperties(user,vo);
        }
        stopWatch.stop();
        System.out.println("Hutools执行时间：" + stopWatch.getLastTaskTimeMillis() + " ms");
        System.out.println(stopWatch.prettyPrint());
    }


Mapstruct执行时间：38 ms
Beanutils执行时间：28487 ms
Hutools执行时间：23559 ms
StopWatch '对象拷贝性能测试': running time = 52085293600 ns
---------------------------------------------
ns         %     Task name
---------------------------------------------
038283000  000%  Mapstruct
28487481200  055%  Beanutils
23559529400  045%  Hutools
```

| **对比对象**          | **10个对象复制1次** | **1万个对象复制1次** | **100万个对象复制1次** | **100万个对象复制5次** |
| --------------------- | ------------------- | -------------------- | ---------------------- | ---------------------- |
| **MapStruct**         | **0ms**             | **3ms**              | **96ms**               | **281ms**              |
| Hutools的BeanUtil     | 23ms                | 102ms                | 1734ms                 | 8316ms                 |
| Spring的BeanUtils     | 2ms                 | 47ms                 | 726ms                  | 3676ms                 |
| Apache的BeanUtils     | 20ms                | 156ms                | 10658ms                | 52355ms                |
| Apache的PropertyUtils | 5ms                 | 68ms                 | 6767ms                 | 30694ms                |

## 原理

> MapStruct 是一个生成类型安全， 高性能且无依赖的 JavaBean 映射代码的注解处理器。

> 您要做的就是定义一个映射器接口，该接口声明任何必需的映射方法。在编译期间，MapStruct将生成此接口的实现。此实现使用简单的Java方法调用在源对象和目标对象之间进行映射，即没有反射或类似内容。
>
> 1. 通过使用普通方法调用（settter/getter）而不是反射来快速执行
> 2. 编译时类型安全性：只能映射相互映射的对象和属性，不能将order实体意外映射到customer DTO等。
> 3. 如果有如下问题，编译时会抛出异常
     >    3.1 映射不完整（并非所有目标属性都被映射）
     >    3.2 映射不正确（找不到正确的映射方法或类型转换）
> 4. 可以通过freemarker定制化开发



## 使用方法

### 1.Maven引入

```xml
  		<!--mapStruct依赖 高性能对象映射-->
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct</artifactId>
            <version>1.5.5.Final</version>
        </dependency>
        <!--mapstruct编译-->
        <dependency>
            <groupId>org.mapstruct</groupId>
            <artifactId>mapstruct-processor</artifactId>
            <version>1.5.5.Final</version>
        </dependency>


    <build>
        <plugins>
			<!-- 增加mapstruct编译顺序，处理lombok和mapstruct冲突 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.mapstruct</groupId>
                            <artifactId>mapstruct-processor</artifactId>
                            <version>1.5.5.Final</version>
                        </path>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                            <version>1.18.16</version>
                        </path>
                        <!-- This is needed when using Lombok 1.18.16 and above -->
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok-mapstruct-binding</artifactId>
                            <version>0.2.0</version>
                        </path>
                    </annotationProcessorPaths>
                </configuration>
            </plugin>
        </plugins>
    </build>
```

### 2. 基本映射

```java
@TableName("sb_user")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
@ToString
public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    @TableId(type = IdType.AUTO)
    private Long id;
    private String name;
    private Long age;
    private String email;
    private Date createTime;
}

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class UserVo2 {
    private String nickName;
    private Long age;
    private String email;
    private Date birthday;
    private String createTime;
}

@Mapper
public interface UserConverter {
    UserConverter INSTANCE = Mappers.getMapper(UserConverter.class);
    //1.基本映射
    UserVo user2UserVo(User user);
    
    @Mapping(target = "nickName", source = "name",defaultValue = "默认")//2.设默认值
    @Mapping(target = "birthday" ,expression = "java(new java.util.Date())")//3.使用表达式
    @Mapping(target = "createTime",source = "createTime",dateFormat = "yyyy-MM-dd")//4.日期格式化
    UserVo2 user2UserVo2(User user);
    
    //5.多源映射
    @Mapping(target = "birthday", source = "user.createTime")
    @Mapping(target = "age",source = "age", numberFormat = "#0.00")//6.数字格式转换
    UserVo2 combinationConvrter(User user, UserVo userVo);
    
    //7.嵌套映射
     @Mapping( target = "name", source = "record.name" )//嵌套映射
     @Mapping( target = ".", source = "record" )//直接使用"."的方式将对象中的属性全部映射到当前目标对象
     @Mapping(target = "sex", source = "user.gender",qualifiedByName = "getSex"),
     Customer  customerDtoToCustomer(CustomerDto customerDto);
    //8.逆映射
    //9.继承与共享配置
    
    //枚举类字段转换
    @Named("getSex")
    static String genderConverter(Integer code){
        for (GenderEnum ge: GenderEnum.values()){
            if (ge.getCode().equals(code)){
                return ge.getName();
            }
        }
        return null;
    }
}

//生成的代码
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2023-12-23T19:24:15+0800",
    comments = "version: 1.5.5.Final, compiler: javac, environment: Java 1.8.0_181 (Oracle Corporation)"
)
public class UserConverterImpl implements UserConverter {
  @Override
  public UserVo2 combinationConverter(User user, UserVo userVo) {
        if ( user == null && userVo == null ) {
            return null;
        }

        UserVo2.UserVo2Builder userVo2 = UserVo2.builder();

        if ( user != null ) {
            userVo2.birthday( user.getCreateTime() );
            userVo2.age( user.getAge() );
            userVo2.sex( UserConverter.genderConverter( user.getGender() ) );
            userVo2.nickName( user.getName() );
            userVo2.email( user.getEmail() );
            userVo2.createTime( dateMapper.asString( user.getCreateTime() ) );
        }

        return userVo2.build();
    }
}
 

```

### 3.自定义映射

```java
public class DateMapper {
    public String asString(Date date) {
        return date != null ? new SimpleDateFormat( "yyyy-MM-dd" )
            .format( date ) : null;
    }
    public Date asDate(String date) {
        try {
            return date != null ? new SimpleDateFormat( "yyyy-MM-dd" )
                .parse( date ) : null;
        }
        catch ( ParseException e ) {
            throw new RuntimeException( e );
        }
    }
}

@Mapper(uses=DateMapper.class)
public interface UserConverter {
    UserVo user2UserVo(User user);
}
```

### 4.集合映射

`MapStruct`有`CollectionMappingStrategy`，与可能的值：`ACCESSOR_ONLY，SETTER_PREFERRED，ADDER_PREFERRED`和T`ARGET_IMMUTABLE`。

在下表中，破折号-表示属性名称。接下来，尾部s表示复数形式。该表解释了这些选项以及它们是如何施加到存在/不存在的`set-s，add-s`和/或`get-s`在目标对象上的方法：

| 选项               | 仅目标set-s可用 | 仅目标add-可用 | 既可以set-s/add- | 没有set-s/add- | 现有目标`（@TargetType）` |
| ------------------ | --------------- | -------------- | ---------------- | -------------- | ------------------------- |
| `ACCESSOR_ONLY`    | set-s           | get-s          | set-s            | get-s          | get-s                     |
| `SETTER_PREFERRED` | set-s           | add-           | set-s            | get-s          | get-s                     |
| `ADDER_PREFERRED`  | set-s           | add-           | add-             | get-s          | get-s                     |
| `TARGET_IMMUTABLE` | set-s           | exception      | set-s            | exception      | set-s                     |





### 5.集成到 spring

在`@Mapper#componentModel` 中指定依赖注入框架

```java
@Mapper(componentModel = "spring")
public interface ModelMapper {

    ModelMapper INSTANT = Mappers.getMapper(ModelMapper.class);

    ModelVO conver(Model model);

}
// 直接在类中使用Autowired注入就行了
@RestController
class MapperSpringController {

    @Autowired
    ModelMapper modelMapper;

    @GetMapping("/get")
    ModelVO getModle(){
       Model model = new Model();
       model.setId("123456");
       model.setName("张三");
       model.setCreate(new Date());
       return modelMapper.conver(model);
    }
}
```







笔记

[神器MapStruct，性能爆棚的实体转换 / 复制工具 - 掘金 (juejin.cn)](https://juejin.cn/post/7214735181172654139)

[Mapstruct 使用教程 - 青竹玉简 - 博客园 (cnblogs.com)](https://www.cnblogs.com/matd/p/17149071.html)

[mapstruct的使用 - 掘金 (juejin.cn)](https://juejin.cn/post/6961255999751585829)

[如何解决mapstruct和lombok冲突问题-CSDN博客](https://blog.csdn.net/TreeShu321/article/details/122354126)

[Lombok和MapStruct整合 - 掘金 (juejin.cn)](https://juejin.cn/post/7099874296373182478)