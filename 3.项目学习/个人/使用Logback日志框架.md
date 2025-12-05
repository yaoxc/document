## 环境：Spring Boot 3.2 + Spring Cloud 2023.0.x（2025-06 验证）

在 **Spring Boot 3.2 / Spring Cloud 2023** 场景下，**使用 Logback 无需手动引入任何依赖**——  
`spring-boot-starter` 已经**自带**完整 Logback 实现，**开箱即用**。

---

### 1. 默认已包含（无需写）

```xml
<!-- 父 pom 已传递 -->
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.4.14</version>   <!-- 与 Boot 3.2 绑定 -->
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>2.0.13</version>
</dependency>
```

只要你的模块继承了 `spring-boot-starter`（或 `spring-boot-starter-web` / `spring-cloud-starter-gateway` 等），**logback-classic** 就会被自动传递进来。

---

### 2. 什么时候**才需要**手动引入

1. **排除其他日志实现后**，想**显式声明版本**（一般不需要）。
2. **非 Spring Boot 项目**（纯 Maven/Gradle 工程）才需要：
   ```xml
   <dependency>
       <groupId>ch.qos.logback</groupId>
       <artifactId>logback-classic</artifactId>
   </dependency>
   ```

---

### 3. 一句话记忆

**“Spring Boot 项目里写 `logback-spring.xml` 即可，依赖已经自带；除非你把 `spring-boot-starter` 排除了，否则永远不用手动加 logback 坐标。”**





