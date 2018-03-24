## 测试

spring boot 提供很多工具类和注解帮助你测试你的应用。测试的支持模块有：包含核心类的spring-boot-test和支持测试自动配置的spring-boot-test-autoconfig。
大多数开发者使用spring-boot-starter-test开始器，他同时保护了spring boot test的模块和Junit，AssertJ，Hamcrest和一系列其他酷。

### 1. 测试范围依赖
spring-boot-starter-test包含下面的库：
- Junit：Java应用测试的实际标准
- Spring Test和spring boot test：spring boot应用相关的工具类和集成测试支持
- AssertJ：一种流式断言库
- Hamcrest：一种对象匹配库（也被称作限制和预言）
- Mockito：一种Java mock框架
- JsonNassert：一种Json的断言库
- JsonPath：Json的XPath

通常发现这些常用库在写测试时非常有用，如果这些库不满足你的需求，你可以添加你想要的测试库。

### 2. 测试spring应用程序
依赖注入的优势让你的代码很容易去做单元测试。你可以忽略spring，用new实例化对象，你也可以用mock的对象代替原来的依赖。
除了单元测试，你通常还需要做集成测试（使用Spring ApplicationContext）。避免部署程序或连接其他基础设施执行集成测试是重要的特性。
Spring Framework包含了专门支持集成测试的模块，你可以直接声明依赖org.springframework:spring-test 或使用spring-boot-starter-test “Starter” 隐式依赖该模块。
如果你还未使用过spring-test模块，请先阅读spring framework参考文档的[相关章节](https://docs.spring.io/spring/docs/5.0.4.RELEASE/spring-framework-reference/testing.html#testing)

### 3. 测试spring boot应用程序
springboot应用程序也是spring `ApplicationContext`，因此和测试普通的spring context没有特殊的地方。

> 如果你使用`SpringApplication`类创建application context，外部配置，日志和其他springboot的特性在这个context中被默认加载。

Spring boot提供`@SpringBootTest`注解，当你使用springboot特性时，使用它来代替标准的springtest提供的`@ContextConfiguration`注解。这个注解通过`SpringApplication`类创建你测试中的ApplicationContext。

You can use the  `webEnvironment`  attribute of  `@SpringBootTest`  to further refine how your tests run:
你可以使用`@SpringBootTest`的`webEnvironment`属性进一步控制你的测试运行：
- `Mock`：加载一个`WebApplicationContext`并提供一个mock的servlet环境。内嵌的servlet容器不会启动。如果你的类路径中没有servlet api，这个模式会自动回退为创建一个普通的非web`ApplicationContext`。这个注解可以和为MockMvc测试的`@AutoConfigureMockMvc` 联合使用。
- `RANDOM_PORT`：加载一个`ServletWebServerApplicationContext`，并且提供一个真实的servlet环境。内嵌容器会启动鉴定一个随机端口。
- `DEFINED_PORT`：加载一个`ServletWebServerApplicationContext`，并提供一个真实的servlet环境。内嵌容器启动并监听在`application.properties`中指定的端口，如果未指定，则为8080端口。
- `NONE`：通过SpringApplication加载一个`ApplicationContext` ，但不提供任何servlet环境。

> 如果测试是`@Transactional`的，会在每个测试方法结束后自动回滚事务。然而，如果使用`RANDOM_PORT`  or  `DEFINED_PORT`隐式提供了真实servlet环境，http客户端和服务端在不同线程中运行，从而就会处在不同事务中，任何在服务端初始化的事务将不会回滚。

> 除了`@SpringBootTest`，也有很多其他注解，为我们提供应用更具体的维度的测试帮助，更多细节请继续看本章节。

> 不要忘记在你的test类上添加`@RunWith(SpringRunner.class)`注解，否则，前面提到的spring相关注解都不会生效。

#### 3.1 检测Web应用类型

If Spring MVC is available, a regular MVC-based application context is configured. If you have only Spring WebFlux, we’ll detect that and configure a WebFlux-based application context instead.
如果检测到Spring MVC可用，会产生一个MVC Application Context，如果你仅仅使用了WebFlux，将会产生一个WebFlux Application Context。
如果两者都存在，Spring MVC优先。如果你想要测试一个reactive web应用，你必须设置`spring.main.web-application-type` 属性:
```
@RunWith(SpringRunner.class)_
@SpringBootTest(properties = "spring.main.web-application-type=reactive")_
public class MyWebFluxTests { ... }
```
