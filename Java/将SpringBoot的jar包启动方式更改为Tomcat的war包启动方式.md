---
date: 2024-08-10 17:40
---

# 将 SpringBoot 的 jar 包启动方式更改为 Tomcat 的 war 包启动方式

个人认为 SpringBoot 开发项目是比 Spring 项目简单方便的，但是有些场景下需要将新开发的项目运行在之前的老环境上，或者是需要兼容其他项目的运行环境，就需要将 jar 包改造成 war 包放到 Tomcat 中去运行，下面记录一下改造过程。

### 1. pom 文件的修改
将 pom.xml 中的 `<packaging>jar</packaging>` 更改为 `<packaging>war</packaging>`。
去除 SpringBoot 中内嵌的 Tomcat，增加 `javax.servlet-api` 依赖
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
	<exclusions>
		<exclusion>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-tomcat</artifactId>
		</exclusion>
	</exclusions>
</dependency>
<dependency>
	<groupId>javax.servlet</groupId>
	<artifactId>javax.servlet-api</artifactId>
	<version>4.0.0</version>
	<scope>provided</scope>
</dependency>
```

### 2. SpringBoot 启动类的改造
将启动类继承 `SpringBootservletInitializer` 类，然后重写 `configure` 方法
```java
@SpringBootApplication
@Slf4j
public class DemoApplication extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
        return builder.sources(DemoApplication.class);
    }

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

### 3. 配置文件修改
需要将其他地方的配置文件都合入到 `resources/application.properties` 配置文件中。
在 Tomcat 部署的时候我们一般是需要把项目的配置文件抽取出来放到 `tocmat/conf/{项目名称}` 下的。此时需要编写一个类，动态的加载配置文件。
```java
public class DemoEnvironmentPostProcessor implements EnvironmentPostProcessor {

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
        // tomcat路径
        String property = System.getProperty("catalina.home");
        String path = property + File.separator + "conf" + File.separator + "demo" + File.separator + "application.properties";
        File file = new File(path);
        if (file.exists()) {
            MutablePropertySources propertySources = environment.getPropertySources();
            Properties properties = loadProperties(file);
            // 相同配置项以tomcat/conf中的为主
            propertySources.addFirst(new PropertiesPropertySource("Config", properties));
        }
    }

    private Properties loadProperties(File file) {
        FileSystemResource resource = new FileSystemResource(file);
        try {
            return PropertiesLoaderUtils.loadProperties(resource);
        } catch (IOException e) {
            throw new IllegalStateException("filed to load local settings from " + file.getAbsolutePath(), e);
        }
    }
}
```

在项目的 `resources` 目录下面创建 `META-INF/spring.factories` 文件，配置指向刚刚创建的 `DemoEnvironmentPostProcessor` 类。
```properties
org.springframework.boot.env.EnvironmentPostProcessor=pw.ydh.demo.DemoEnvironmentPostProcessor
```

好了，现在就可以打出 war 包并放到 Tomcat 中运行了。