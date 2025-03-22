---
date: 2024-08-11 11:55
---

# 对 MySQL 中字段进行加解密

[TOC]

MySQL 原生支持对字段进行加解密操作的，使用它提供的函数 `aes_encrypt` 和 `aes_decrypt` 即可实现，但是直接使用加解密函数有两个弊端，会导致在项目中使用不太便捷：
1. 增删改查只能使用原生 SQL，使用起来较为繁琐；
2. 加密 key_str 只能是固定的，在编码阶段就需要定义好。
针对这两个弊端本文给出了一种新的解决方式，使用起来较为方便。
1. 支持使用 JPA 进行增删改查，不需要编写原生 SQL；
2. 配置文件中可以配置加密 key_str，让客户有更多的选择。

## 1. 定义表结构
表字段需要定义为 blob 类型，对应的实体类中定义为 String 类型，这边需要特别注意一下枚举的定义，枚举类也要定义为 blob 类型。
```sql
DROP TABLE IF EXISTS `demo`;
CREATE TABLE `demo` (
  `id` varchar(50) NOT NULL,
  `name` varchar(50) DEFAULT NULL,
  `password` blob,
  `status_a` blob,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;
```

## 2. 定义实体类
需要在加解密的字段上增加 `@ColumnTransformer` 注解，枚举类需要将枚举值设置为 String 类型 `@Enumerated(EnumType.STRING)`。
```java
@Data
@Entity
@Table(name = "demo")
public class Demo {

    @Id
    private String id;

    private String name;

    @ColumnTransformer(
            read = "cast(aes_decrypt(password, sha2('my_key', 256)) as char)",
            write = "aes_encrypt(?, sha2('my_key', 256))"
    )
    private String password;

    @ColumnTransformer(
            read = "cast(aes_decrypt(STATUS_A, sha2('my_key', 256)) as char)",
            write = "aes_encrypt(?, sha2('my_key', 256))"
    )
    @Enumerated(EnumType.STRING)
    @Column(name = "status_a")
    private StatusAEnum statusA;

    public enum StatusAEnum {
        LOCKED,
        UNLOCKED
    }
}
```

## 3. 使用 JPA 对数据进行增删改查
完成第一步和第二步操作之后就可以使用 JPA 中的方法对数据进行增删改查了。
```java
demoRepository.save(demo);

// ------------------------

Optional<Demo> optionalDemo = demoRepository.findById(id);
Demo demo = optionalDemo.get();
demo.setStatusA(statusA);
demoRepository.save(demo);
```

## 4. 使用 Java 反射技术动态指定加解密 key_str
实体类中的字段上有
```java
@ColumnTransformer(
            read = "cast(aes_decrypt(password, sha2('my_key', 256)) as char)",
            write = "aes_encrypt(?, sha2('my_key', 256))"
)
```
注解，其中 my_key 是无法设置为变量，故无法从配置文件中动态的获取配置。需要使用反射技术在工程启动的时候修改 my_key 的值。
需要注意⚠️的是：在 Spring 容器启动之前做反射处理。

```java
@SpringBootApplication
@Slf4j
public class DemoApplication {

    public static void main(String[] args) {

        // 需要放在spring容器启动之前做反射处理
        databaseCrypt();

        SpringApplication.run(DemoApplication.class, args);
    }

    private static void databaseCrypt() {
        Demo demo = new Demo();
        databaseCryptConfig(demo, "password", "password");
        databaseCryptConfig(demo, "statusA", "status_a");
    }

    /**
     * 反射修改实体类中@ColumnTransformer注解内容，从外部配置文件中获取加密密钥
     *
     * @param obj             实体类
     * @param cryptField      实体类中需要加密字段
     * @param datasourceField 数据库中的字段
     */
    private static void databaseCryptConfig(Object obj, String cryptField, String datasourceField) {
        try {
            Field field = obj.getClass().getDeclaredField(cryptField);
            field.setAccessible(true);
            ColumnTransformer annotation = field.getAnnotation(ColumnTransformer.class);
            InvocationHandler invocationHandler = Proxy.getInvocationHandler(annotation);
            Field memberValues = invocationHandler.getClass().getDeclaredField("memberValues");
            memberValues.setAccessible(true);
            Map<String, Object> values = (Map<String, Object>) memberValues.get(invocationHandler);
            log.info("------------修改之前：" + annotation.read());
            log.info("------------修改之前：" + annotation.write());

            String readValue = getReadValue(datasourceField);
            values.put("read", readValue);
            String writeValue = getWriteValue();
            values.put("write", writeValue);

            field.setAccessible(true);
            ColumnTransformer annotation2 = field.getAnnotation(ColumnTransformer.class);
            log.info("------------修改之后：" + annotation.read());
            log.info("------------修改之后：" + annotation.write());

        } catch (NoSuchFieldException | IllegalAccessException e) {
            throw new RuntimeException(e);
        }
    }

    /**
     * 拼接的mysql解密函数
     *
     * @param datasourceField
     * @return
     */
    private static String getReadValue(String datasourceField) {
        return "cast(aes_decrypt(" + datasourceField + ", sha2('" + databaseCryptConfigReader() + "', 256)) as char)";
    }

    /**
     * 拼接的mysql加密函数
     *
     * @return
     */
    private static String getWriteValue() {
        return "aes_encrypt(?, sha2('" + databaseCryptConfigReader() + "', 256))";
    }

    /**
     * 从配置文件中获取加密key
     *
     * @return
     */
    private static String databaseCryptConfigReader() {
        Properties properties = new Properties();
        String cryptKey;
        try (InputStream inputStream = ResourceReader.class.getClassLoader()
                .getResourceAsStream("application.properties")) {
            properties.load(inputStream);
            cryptKey = properties.getProperty("database.encrypt.key");
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        return cryptKey;
    }
}
```