## SpringBoot 测试

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

你可以使用`@SpringBootTest`的`webEnvironment`属性进一步控制你的测试运行：
- `Mock`：加载一个`WebApplicationContext`并提供一个mock的servlet环境。内嵌的servlet容器不会启动。如果你的类路径中没有servlet api，这个模式会自动回退为创建一个普通的非web`ApplicationContext`。这个注解可以和为MockMvc测试的`@AutoConfigureMockMvc` 联合使用。
- `RANDOM_PORT`：加载一个`ServletWebServerApplicationContext`，并且提供一个真实的servlet环境。内嵌容器会启动鉴定一个随机端口。
- `DEFINED_PORT`：加载一个`ServletWebServerApplicationContext`，并提供一个真实的servlet环境。内嵌容器启动并监听在`application.properties`中指定的端口，如果未指定，则为8080端口。
- `NONE`：通过SpringApplication加载一个`ApplicationContext` ，但不提供任何servlet环境。

> 如果测试是`@Transactional`的，会在每个测试方法结束后自动回滚事务。然而，如果使用`RANDOM_PORT`  or  `DEFINED_PORT`隐式提供了真实servlet环境，http客户端和服务端在不同线程中运行，从而就会处在不同事务中，任何在服务端初始化的事务将不会回滚。

> 除了`@SpringBootTest`，也有很多其他注解，为我们提供应用更具体的维度的测试帮助，更多细节请继续看本章节。

> 不要忘记在你的test类上添加`@RunWith(SpringRunner.class)`注解，否则，前面提到的spring相关注解都不会生效。

#### 3.1 察觉Web应用类型

如果检测到Spring MVC可用，会产生一个MVC Application Context，如果你仅仅使用了WebFlux，将会产生一个WebFlux Application Context。
如果两者都存在，Spring MVC优先。如果你想要测试一个reactive web应用，你必须设置`spring.main.web-application-type` 属性:
```
@RunWith(SpringRunner.class)_
@SpringBootTest(properties = "spring.main.web-application-type=reactive")_
public class MyWebFluxTests { ... }
```

#### 3.2 察觉测试配置

如果你熟悉Spring Test Framework，你可能会使用`@ContextConfiguration(classes=…​)`去指定你想加载的`@Configuration`。另一个选择是你在test类中嵌入`@Configuration`类。

测试springboot应用时，不需要以上的编程方式，SpringBoot的`@*Test`系列注解会在你未显示指定的情况下自动搜索你的主要配置。

搜索的算法是从当前package开始，逐步搜索直到找到一个被 `@SpringBootApplication`  or  `@SpringBootConfiguration`注解标注的类。只要你的使用 [合理的代码组织结构](https://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-structuring-your-code.html "14. Structuring Your Code")，通常配置都会被找到。

> 如果你使用 [测试注解去测试应用更具体的维度](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-testing-autoconfigured-tests "43.3.6 Auto-configured Tests"), 你应该避免在[main方法的应用类](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-testing-user-configuration "43.3.19 User Configuration and Slicing")上添加具体到某个区域的配置 。

如果你想要定制化主要的配置，你可以用一个内嵌的`@TestConfiguration`类。不同于内嵌的`@Configuration`类会被用来替代你应用的主要配置，一个`@TestConfiguration`类被用于补充应用的主要配置。

> Spring的测试框架在多个@Test测试方法中缓存ApplicationContexts。因此，只要你的测试共享一个配置（不管怎么被发现的），加载context的过程只会加载一次，只耗费一次加载时间。

#### 3.3 排除测试配置

如果你的应用要使用component scanning（比如你用了`@SpringBootApplication`  or  `@ComponentScan`），如果你把为某个具体测试创建的configuration类放在顶层包，这个configuration会被各种test使用。

如[之前所述](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-detecting-config "43.3.2 Detecting Test Configuration")`@TestConfiguration`可以被用于一个测试类的内嵌类，去定制化主要配置，当它在顶层类中，`@TestConfiguration`表明在`src/test/java`中的类不应该被Scan，你可以随后在需要的时候显示导入该类，如下例所示：
```
@RunWith(SpringRunner.class)_
@SpringBootTest
@Import(MyTestsConfiguration.class)
public class MyTests {

	@Test
	public void exampleTest() {
		...
	}

}
```

> 如果你直接使用@ComponentScan（也就是不通过@SpringBootApplication），你必须注册`TypeExcludeFilter`，如下：
>`@ComponentScan(excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
		`

#### 3.4 运行服务测试

如果你需要启动一个完整的服务器，我们推荐你使用随机端口。如果你使用`@SpringBootTest(webEnvironment=WebEnvironment.RANDOM_PORT)`，你每个test运行时都会分配一个随机端口。

`@LocalServerPort`注解可以被用于[注入实际使用的端口](https://docs.spring.io/spring-boot/docs/current/reference/html/howto-embedded-web-servers.html#howto-discover-the-http-port-at-runtime "75.6 Discover the HTTP Port at Runtime") 到你的测试中。为方便，需要rest调用的测试可以`@Autowire`一个[`WebTestClient`](https://docs.spring.io/spring/docs/5.0.4.RELEASE/spring-framework-reference/testing.html#webtestclient-tests)，这个bean可以使用相对路径访问当前运行的服务，并且有专用严重response的api，如下面例子所示：
```
import org.junit.Test;
import org.junit.runner.RunWith;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.reactive.server.WebTestClient;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
public class RandomPortWebTestClientExampleTests {

	@Autowired
	private WebTestClient webClient;

	@Test
	public void exampleTest() {
		this.webClient.get().uri("/").exchange().expectStatus().isOk()
				.expectBody(String.class).isEqualTo("Hello World");
	}

}
```
SpringBoot也提供一个`TestRestTemplate`工具：
```
import org.junit.Test;
import org.junit.runner.RunWith;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.test.context.junit4.SpringRunner;

import static org.assertj.core.api.Assertions.assertThat;

_@RunWith(SpringRunner.class)_
_@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)_
public class RandomPortTestRestTemplateExampleTests {

	_@Autowired_
	private TestRestTemplate restTemplate;

	_@Test_
	public void exampleTest() {
		String body = this.restTemplate.getForObject("/", String.class);
		assertThat(body).isEqualTo("Hello World");
	}

}
```

#### 3.5 Mocking和Spying Bean

运行测试时，有时候需要去mock一些组件到你的application context中，比如，你可能有一些依赖的远程服务在部署的时候不可用，Mocking也可用于模拟真实环境中很难碰到的失败。

Spring Boot有一个`@MockBean`注解，可用于为`ApplicationContext`定义一个Mockito mock bean，可以用这个注解去添加新的Bean或替换已存在的bean。这个注解可以直接在test类上使用，或在test类的属性上使用，或在`@Configuration`类或起属性上使用。当你用在一个属性上时，mock的实例会被注入，mock bean在每次测试方法运行时会被重置。

> 如果你的测试使用spring boot测试注解（比如`@SpringBootTest`），以上特性时自动启用的。为了在其他情形下使用这个特性，需要显示添加一个listener，如下例所示：
```
@TestExecutionListeners(MockitoTestExecutionListener.class)
```

下面的例子是使用mock替换一个已经存在的`RemoteService`  bean
```
import org.junit.*;
import org.junit.runner.*;
import org.springframework.beans.factory.annotation.*;
import org.springframework.boot.test.context.*;
import org.springframework.boot.test.mock.mockito.*;
import org.springframework.test.context.junit4.*;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.BDDMockito.*;

@RunWith(SpringRunner.class)
@SpringBootTest
public class MyTests {

	@MockBean
	private RemoteService remoteService;

	@Autowired
	private Reverser reverser;

	@Test
	public void exampleTest() {
		// RemoteService has been injected into the reverser bean
		given(this.remoteService.someCall()).willReturn("mock");
		String reverse = reverser.reverseSomeCall();
		assertThat(reverse).isEqualTo("kcom");
	}
}
```
除此之外，你可以使用`@SpyBean`去包裹一个已存在的bean，详请参考[Javadoc](https://docs.spring.io/spring-boot/docs/2.0.0.RELEASE/api/org/springframework/boot/test/mock/mockito/SpyBean.html)

> Spring测试框架在测试中缓存application context，重用context，configuration，`@MockBean`和 `@SpyBean`影响缓存key，很可能增加context的数量。

#### 3.5 自动配置测试

Spring Boot的自动配置系统非常方便，但有时候对测试来说是部分是多余的。加载一部分需要的配置去测试你应用的某个维度是很有帮助的。比如，你可能想要测试Spring MVC controllers mapping的url是否正确，但你不想访问数据库；或你可能想测试Jpa的entity，但你对web层不关注。

`spring-boot-test-autoconfigure`模块包含了很多注解，可实现以上需求的测试的自动配置，他们的工作方式都是类似的，一个`@…​Test`注解加载`ApplicationContext`，一个或多个`@AutoConfigure…​`注解定制化自动配置。

> 每个维度加载一个非常贴切的自动配置类组。如果你想排除其中一个，大多数`@…​Test`注解提供`excludeAutoConfiguration` 属性，另一个方式是利用`@ImportAutoConfiguration`注解的exclude属性。

> 结合标准的`@SpringBootTest` 注解使用`@AutoConfigure…​`注解也是可以的，如果你对切分你应用的维度不感兴趣但又洗完一些自动配置的测试bean，你可以使用这样的组合。

#### 3.7 自动配置Json测试

为了测试对象的json序列化和反序列化是否如预期，你可以使用`@JsonTest`注解。这个注解自动配置了可用的json mapper，可以是如下任何一种：
-   Jackson  `ObjectMapper`, 任何  `@JsonComponent`  beans 以及任何Jackson模块
-   `Gson`
-   `Jsonb`

如果你需要配置自动配置的元素，可以使用`@AutoConfigureJsonTesters` 注解。

Spring Boot包含了基于AssertJ的帮助类，结合JsonAssert和JsonPath库去检测Json是否如预期。`JacksonTester`,  `GsonTester`,  `JsonbTester`和  `BasicJsonTester`类分别针对Jackson, Gson, Jsonb和Strings。当使用`@JsonTest`注解后，以上任何一个类的实例都可以通过`@Autowired`注入测试类属性中。下面是一个针对Jackson的例子：
```
import org.junit.*;
import org.junit.runner.*;
import org.springframework.beans.factory.annotation.*;
import org.springframework.boot.test.autoconfigure.json.*;
import org.springframework.boot.test.context.*;
import org.springframework.boot.test.json.*;
import org.springframework.test.context.junit4.*;

import static org.assertj.core.api.Assertions.*;

_@RunWith(SpringRunner.class)_
_@JsonTest_
public class MyJsonTests {

	_@Autowired_
	private JacksonTester<VehicleDetails> json;

	_@Test_
	public void testSerialize() throws Exception {
		VehicleDetails details = new VehicleDetails("Honda", "Civic");
		// Assert against a `.json` file in the same package as the test
		assertThat(this.json.write(details)).isEqualToJson("expected.json");
		// Or use JSON path based assertions
		assertThat(this.json.write(details)).hasJsonPathStringValue("@.make");
		assertThat(this.json.write(details)).extractingJsonPathStringValue("@.make")
				.isEqualTo("Honda");
	}

	_@Test_
	public void testDeserialize() throws Exception {
		String content = "{\\"make\\":\\"Ford\\",\\"model\\":\\"Focus\\"}";
		assertThat(this.json.parse(content))
				.isEqualTo(new VehicleDetails("Ford", "Focus"));
		assertThat(this.json.parseObject(content).getMake()).isEqualTo("Ford");
	}

}
```

> 如果不使用@JsonTest注解，Json帮助类也可以直接在标准unit test中使用，但需要在`@Before`方法中先调用帮助类的`initFields`方法。

被`@JsonTest`启用的所有自动配置列表请参考[附录](https://docs.spring.io/spring-boot/docs/current/reference/html/test-auto-configuration.html "Appendix D. Test auto-configuration annotations").

#### 3.8 自动配置Spring MVC测试

测试Spring MVC controller是否如预期方式工作，可使用`@WebMvcTest`注解，该注解自动配置SpringMVC的基础，将scan的注解限制为`@Controller`,  `@ControllerAdvice`,  `@JsonComponent`,  `Converter`,  `GenericConverter`,  `Filter`,  `WebMvcConfigurer`和 `HandlerMethodArgumentResolver`.普通的`@Component` bean不会被scan到。

如果你需要注册额外的component，比如Jackson `Module`，可以在测试中使用`@Import` 导入额外的@configuration类。

通常的`@WebMvcTest`测试限制到某一个controller，结合`@MockBean`提供mock的协作实现。

`@WebMvcTest`  also auto-configures  `MockMvc`. Mock MVC offers a powerful way to quickly test MVC controllers without needing to start a full HTTP server.
`@WebMvcTest`也自动配置`MockMvc`，它提供快速测试MVC controller的有力方式，并且不需要启动一个完整的http服务。

> 你也可以在不使用`@WebMvcTest`的情况（比如使用`@SpringBootTest`）下使用自动配置的`MockMvc` ，只需要对测试类添加`@AutoConfigureMockMvc`注解，以下例子使用了`MockMvc`：
```
import org.junit.*;
import org.junit.runner.*;
import org.springframework.beans.factory.annotation.*;
import org.springframework.boot.test.autoconfigure.web.servlet.*;
import org.springframework.boot.test.mock.mockito.*;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.BDDMockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

_@RunWith(SpringRunner.class)_
_@WebMvcTest(UserVehicleController.class)_
public class MyControllerTests {

	_@Autowired_
	private MockMvc mvc;

	_@MockBean_
	private UserVehicleService userVehicleService;

	_@Test_
	public void testExample() throws Exception {
		given(this.userVehicleService.getVehicleDetails("sboot"))
				.willReturn(new VehicleDetails("Honda", "Civic"));
		this.mvc.perform(get("/sboot/vehicle").accept(MediaType.TEXT_PLAIN))
				.andExpect(status().isOk()).andExpect(content().string("Honda Civic"));
	}

}
```

`@AutoConfigureMockMvc`  annotation.
如果你需要配置MVC自动配置的元素（比如要应用servlet过滤器），可以使用`@AutoConfigureMockMvc`注解中的相关属性。

如果你使用HtmlUnit或Selenium，MVC自动配置也提供了HTMLUnit的`WebClient` bean和Selenium的`WebDriver` bean，可供注入使用。以下例子使用了HtmlUnit：
```
import com.gargoylesoftware.htmlunit.*;
import org.junit.*;
import org.junit.runner.*;
import org.springframework.beans.factory.annotation.*;
import org.springframework.boot.test.autoconfigure.web.servlet.*;
import org.springframework.boot.test.mock.mockito.*;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.BDDMockito.*;

_@RunWith(SpringRunner.class)_
_@WebMvcTest(UserVehicleController.class)_
public class MyHtmlUnitTests {

	_@Autowired_
	private WebClient webClient;

	_@MockBean_
	private UserVehicleService userVehicleService;

	_@Test_
	public void testExample() throws Exception {
		given(this.userVehicleService.getVehicleDetails("sboot"))
				.willReturn(new VehicleDetails("Honda", "Civic"));
		HtmlPage page = this.webClient.getPage("/sboot/vehicle.html");
		assertThat(page.getBody().getTextContent()).isEqualTo("Honda Civic");
	}

}
```

> 默认，Spring Boot将`WebDriver` bean放入一个特殊的scope，保证这个driver在测试结束后退出，新的测试开始后又注入一个新的实例。如果你不想这样，你可以在`WebDriver` bean上添加`@Scope("singleton")`注解。

`@WebMvcTest`注解启用的自动配置项列表请参考[附录](https://docs.spring.io/spring-boot/docs/current/reference/html/test-auto-configuration.html "Appendix D. Test auto-configuration annotations").

> 有时候写Spring MVC tests是不够的，Spring Boot帮助我们在实际server中运行[完整的端对端测试](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-testing-with-running-server "43.3.4 Testing with a running server").

#### 3.9 自动配置Spring WebFlux测试
_spring 5 reactive web内容，暂不翻译，请参考[最新英文文档](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-testing-autoconfigured-webflux-tests)_

#### 3.10 自动配置 JPA测试

你可以使用`@DataJpaTest`注解测试JPA的应用，这个注解默认会配置一个内存数据库，扫描`@Entity`类，配置Spring Data JPA repositories。普通的`@Component` bean将不会被加载到`ApplicationContext`.

默认，JPA测试是事务性的，数据操作会在每个测试结束后回滚，详请参考Spring Framework参考文档的 [相关章节](https://docs.spring.io/spring/docs/5.0.4.RELEASE/spring-framework-reference/testing.html#testcontext-tx-enabling-transactions) ，如果你不希望回滚，可以关闭某个测试的事务管理，如下例所示：
```
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

_@RunWith(SpringRunner.class)_
_@DataJpaTest_
_@Transactional(propagation = Propagation.NOT_SUPPORTED)_
public class ExampleNonTransactionalTests {

}
```

Data JPA测试也可以注入 [`TestEntityManager`](https://github.com/spring-projects/spring-boot/tree/v2.0.0.RELEASE/spring-boot-project/spring-boot-test-autoconfigure/src/main/java/org/springframework/boot/test/autoconfigure/orm/jpa/TestEntityManager.java)  bean，这个bean是标准JPA  `EntityManager` 的一种替代，专为测试设计。如果你希望在未使用`@DataJpaTest`的测试中使用这个bean，你可以用`@AutoConfigureTestEntityManager` 注解。`@DataJpaTest`也会产生一个可用的`JdbcTemplate` bean。下面的例子展示相关特性：
```
import org.junit.*;
import org.junit.runner.*;
import org.springframework.boot.test.autoconfigure.orm.jpa.*;

import static org.assertj.core.api.Assertions.*;

_@RunWith(SpringRunner.class)_
_@DataJpaTest_
public class ExampleRepositoryTests {

	_@Autowired_
	private TestEntityManager entityManager;

	_@Autowired_
	private UserRepository repository;

	_@Test_
	public void testExample() throws Exception {
		this.entityManager.persist(new User("sboot", "1234"));
		User user = this.repository.findByUsername("sboot");
		assertThat(user.getUsername()).isEqualTo("sboot");
		assertThat(user.getVin()).isEqualTo("1234");
	}

}
```

嵌入式内存数据库普遍能够在测试中工作良好，他们速度很快且不需要任何的安装。但如果你更喜欢和真实的database打交道，可以使用`@AutoConfigureTestDatabase`注解，如下面的例子：
```
_@RunWith(SpringRunner.class)_
_@DataJpaTest_
_@AutoConfigureTestDatabase(replace=Replace.NONE)_
public class ExampleRepositoryTests {

	// ...

}
```

`@DataJpaTest` 带来的所有自动配置项列表请参考[附录](https://docs.spring.io/spring-boot/docs/current/reference/html/test-auto-configuration.html "Appendix D. Test auto-configuration annotations").

#### 3.11 自动配置JDBC测试

`@JdbcTest`和`@DataJpaTest` 类似，但是只针对纯JDBC相关的测试。默认情况下，该注解会配置一个嵌入式内存数据库和一个`JdbcTemplate` bean。普通的`@Component` bean不会被加载到`ApplicationContext`.

默认JDBC测试是事务性的，在每个测试的结束后回滚，详请参考Spring Framework参考文档的 [相关章节](https://docs.spring.io/spring/docs/5.0.4.RELEASE/spring-framework-reference/testing.html#testcontext-tx-enabling-transactions) ，如果你不希望回滚，可以关闭某个测试的事务管理，如下例所示：
```
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.autoconfigure.jdbc.JdbcTest;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;

_@RunWith(SpringRunner.class)_
_@JdbcTest_
_@Transactional(propagation = Propagation.NOT_SUPPORTED)_
public class ExampleNonTransactionalTests {

}
```

如果你更希望与真实的数据库打交道，你可以使用`@AutoConfigureTestDatabase`注解，如`DataJpaTest`章节中所述的相同方式。(参考"[Section 43.3.10, “Auto-configured Data JPA Tests”](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-testing-autoconfigured-jpa-test "43.3.10 Auto-configured Data JPA Tests")".)

`@JdbcTest` 带来的所有自动配置项列表请参考[附录](https://docs.spring.io/spring-boot/docs/current/reference/html/test-auto-configuration.html "Appendix D. Test auto-configuration annotations").

#### 3.12 自动配置JOOQ测试

`@JooqTest`的使用方式和`@JdbcTest`类似，只是仅针对jOOQ相关测试。由于jOOQ重度依赖与数据库schema映射的Java schema，因此使用已有的`DataSource`。如果你想用内存数据库数据源，可以使用`@AutoconfigureTestDatabase` 注解覆盖这些配置。(更多基于SpringBoot的jOOQ 内容请参考本章"[Section 29.5, “Using jOOQ”](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-sql.html#boot-features-jooq "29.5 Using jOOQ")")

`@JooqTest`  configures a  `DSLContext`. Regular  `@Component`  beans are not loaded into the  `ApplicationContext`. The following example shows the  `@JooqTest`  annotation in use:
`@JooqTest`配置了一个`DSLContext`，普通`@Component` bean将不会被加载到`ApplicationContext`，下面是`@JooqTest`注解的使用示例：
```
import org.jooq.DSLContext;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.autoconfigure.jooq.JooqTest;
import org.springframework.test.context.junit4.SpringRunner;

_@RunWith(SpringRunner.class)_
_@JooqTest_
public class ExampleJooqTests {

	_@Autowired_
	private DSLContext dslContext;
}
```

JOOQ测试是事务性的，默认情况下测试结束后会回滚，如果你不希望回滚，可以关闭事务管理，针对某个test或所有test类  [见JDBC示例](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-testing-autoconfigured-jdbc-test "43.3.11 Auto-configured JDBC Tests").

`@JooqTest`带来的所有自动配置项请参考 [附录](https://docs.spring.io/spring-boot/docs/current/reference/html/test-auto-configuration.html "Appendix D. Test auto-configuration annotations").

#### 3.13 自动配置Data MongoDB Tests

 `@DataMongoTest`用于测试基于MongoDB存储数据的应用，默认会配置一个嵌入式内存MongoDB数据库（如果数据库依赖存在），配置一个`MongoTemplate`，扫描`@Document`注解的类，配置Spring Data MongoDB repositories，普通`@Component` bean不会被扫描并加载到`ApplicationContext`。(更多SpringBoot MongoDB介绍请参考"[Section 30.2, “MongoDB”](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-nosql.html#boot-features-mongodb "30.2 MongoDB")")

下面的示例展示了`@DataMongoTest`注解的用法：
```
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.data.mongo.DataMongoTest;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.test.context.junit4.SpringRunner;

_@RunWith(SpringRunner.class)_
_@DataMongoTest_
public class ExampleDataMongoTests {

	_@Autowired_
	private MongoTemplate mongoTemplate;

	//
}
```

嵌入式内存MongoDB数据库通常在测试中工作良好，迅速且不需要任何安装程序，但如果你更喜欢和真实的MongDB server打交道，可以排除掉embedded MongoDB的自动配置，示例如下：
 ```
import org.junit.runner.RunWith;
 import org.springframework.boot.autoconfigure.mongo.embedded.EmbeddedMongoAutoConfiguration;
import org.springframework.boot.test.autoconfigure.data.mongo.DataMongoTest;
import org.springframework.test.context.junit4.SpringRunner;

_@RunWith(SpringRunner.class)_
_@DataMongoTest(excludeAutoConfiguration = EmbeddedMongoAutoConfiguration.class)_
public class ExampleDataMongoNonEmbeddedTests {

}
```

`@DataMongoTest`启用的所有自动配置项列表请参考[附录](https://docs.spring.io/spring-boot/docs/current/reference/html/test-auto-configuration.html "Appendix D. Test auto-configuration annotations").

#### 3.14 自动配置Data  Neo4j测试

Neo4j技术暂未在Greenwich使用，暂不翻译，请参考[英文文档](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-testing-autoconfigured-neo4j-test)

#### 3.15 自动配置Data Redis Tests

`@DataRedisTest`用于测试基于Redis存储数据的应用，该注解默认扫描`@RedisHash`注解的类，配置Spring Data Redis repositories，普通`@Component` bean不会被扫描加载到`ApplicationContext`。(更多Spring Boot Redis的介绍请参考"[Section 30.1, “Redis”](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-nosql.html#boot-features-redis "30.1 Redis")".)

下面的例子展示了`@DataRedisTest`注解的用法：
```
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.data.redis.DataRedisTest;
import org.springframework.test.context.junit4.SpringRunner;

_@RunWith(SpringRunner.class)_
_@DataRedisTest_
public class ExampleDataRedisTests {

	_@Autowired_
	private YourRepository repository;

	//
}
```

`@DataRedisTest`启用的所有自动配置项列表请参考 [附录](https://docs.spring.io/spring-boot/docs/current/reference/html/test-auto-configuration.html "Appendix D. Test auto-configuration annotations").

#### 3.16 自动配置Data LDAP Tests

本章不翻译，请参考[英文文档](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-testing-autoconfigured-ldap-test)

#### 3.17 自动配置Rest Client

可以使用`@RestClientTest`注解测试rest客户端，该注解默认会自动配置Jackson，gson和Jsonb的支持，同时配置`RestTemplateBuilder`，支持`MockRestServiceServer`。你想测试的具体Bean需要使用`@RestClientTest`的value或`components`属性指定，如下例所示：
```
_@RunWith(SpringRunner.class)_
_@RestClientTest(RemoteVehicleDetailsService.class)_
public class ExampleRestClientTest {

	_@Autowired_
	private RemoteVehicleDetailsService service;

	_@Autowired_
	private MockRestServiceServer server;

	_@Test_
	public void getVehicleDetailsWhenResultIsSuccessShouldReturnDetails()
			throws Exception {
		this.server.expect(requestTo("/greet/details"))
				.andRespond(withSuccess("hello", MediaType.TEXT_PLAIN));
		String greeting = this.service.callRestService();
		assertThat(greeting).isEqualTo("hello");
	}

}
```

`@RestClientTest`带来的所有自动配置项列表请参考 [附录](https://docs.spring.io/spring-boot/docs/current/reference/html/test-auto-configuration.html "Appendix D. Test auto-configuration annotations").

#### 3.18 自动配置Spring Rest Docs Tests

当你的测试使用[Spring REST Docs](https://projects.spring.io/spring-restdocs/) 和Mock MVC或Rest Assured，可以使用`@AutoConfigureRestDocs`注解，在Spring REST Docs中你不需要JUnit Rule。

`@AutoConfigureRestDocs`注解可用来覆盖默认的输出目录(Maven `target/generated-snippets`或Gradle `build/generated-snippets`)，该注解还可以被用于修改所有文档中会出现的host、scheme和port等信息。

**结合Mock MVC的自动配置Spring Rest Docs测试**

为了使用Spring REST Docs，`@AutoConfigureRestDocs`定制了`MockMvc` bean。你可以注入这个bean并在测试中使用它，如下例所示：
```
import org.junit.Test;
import org.junit.runner.RunWith;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.restdocs.mockmvc.MockMvcRestDocumentation.document;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

_@RunWith(SpringRunner.class)_
_@WebMvcTest(UserController.class)_
_@AutoConfigureRestDocs_
public class UserDocumentationTests {

	_@Autowired_
	private MockMvc mvc;

	_@Test_
	public void listUsers() throws Exception {
		this.mvc.perform(get("/users").accept(MediaType.TEXT_PLAIN))
				.andExpect(status().isOk())
				.andDo(document("list-users"));
	}

}
```

如果需要对Spring REST Docs的配置进行更精细的控制，除了`@AutoConfigureRestDocs`提供的属性控制以外，可以使RestDocsMockMvcConfigurationCustomizer` config bean，如下例所示：
```
_@TestConfiguration_
static class CustomizationConfiguration
		implements RestDocsMockMvcConfigurationCustomizer {

	_@Override_
	public void customize(MockMvcRestDocumentationConfigurer configurer) {
		configurer.snippets().withTemplateFormat(TemplateFormats.markdown());
	}

}
```

如果想使用Spring REST Docs支持参数化输出，你可以创建一个`RestDocumentationResultHandler` bean。这种自动配置被称作`alwaysDo`，从而使得每个`MockMvc`调用自动生成默认片段，下面的例子展示了`RestDocumentationResultHandler` bean的定义方式：
```
_@TestConfiguration_
static class ResultHandlerConfiguration {

	_@Bean_
	public RestDocumentationResultHandler restDocumentation() {
		return MockMvcRestDocumentation.document("{method-name}");
	}

}
```

#### [Auto-configured Spring REST Docs Tests with REST Assured](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-testing-autoconfigured-rest-docs-rest-assured)

`@AutoConfigureRestDocs`  makes a  `RequestSpecification`  bean, preconfigured to use Spring REST Docs, available to your tests. You can inject it by using  `@Autowired`  and use it in your tests as you normally would when using REST Assured and Spring REST Docs, as shown in the following example:
```
import io.restassured.specification.RequestSpecification;
import org.junit.Test;
import org.junit.runner.RunWith;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.restdocs.AutoConfigureRestDocs;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.boot.web.server.LocalServerPort;
import org.springframework.test.context.junit4.SpringRunner;

import static io.restassured.RestAssured.given;
import static org.hamcrest.CoreMatchers.is;
import static org.springframework.restdocs.restassured3.RestAssuredRestDocumentation.document;

_@RunWith(SpringRunner.class)_
_@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)_
_@AutoConfigureRestDocs_
public class UserDocumentationTests {

	_@LocalServerPort_
	private int port;

	_@Autowired_
	private RequestSpecification documentationSpec;

	_@Test_
	public void listUsers() {
		given(this.documentationSpec).filter(document("list-users")).when()
				.port(this.port).get("/").then().assertThat().statusCode(is(200));
	}

}
```
If you require more control over Spring REST Docs configuration than offered by the attributes of  `@AutoConfigureRestDocs`, a  `RestDocsRestAssuredConfigurationCustomizer`  bean can be used, as shown in the following example:
```
_@TestConfiguration_
public static class CustomizationConfiguration
		implements RestDocsRestAssuredConfigurationCustomizer {

	_@Override_
	public void customize(RestAssuredRestDocumentationConfigurer configurer) {
		configurer.snippets().withTemplateFormat(TemplateFormats.markdown());
	}

}
```

#### 3.19 [用户配置和切片](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-testing-user-configuration)

如果你使用合理的方式[组织代码](https://docs.spring.io/spring-boot/docs/current/reference/html/using-boot-structuring-your-code.html "14. Structuring Your Code")，`@SpringBootApplication` 类[默认被用作测试的配置](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-detecting-config "43.3.2 Detecting Test Configuration")

不要将应用的main类和某个特定区域功能的配置放在一起，这很重要。

假设你使用Spring Batch且依赖它的自动配置，你以如下方式定义 `@SpringBootApplication` 
```
_@SpringBootApplication_
_@EnableBatchProcessing_
public class SampleApplication { ... }
```

由于这个类是测试的源配置，任何切片测试都会尝试启动Spring Batch，但这不是你想要的，一个推荐的方式是将区域特定的配置移到一个单独的`@Configuration`类中，例如：
```
_@Configuration_
_@EnableBatchProcessing_
public class BatchConfiguration { ... }
```

> 根据应用的复杂度，你可能需要一个单独的定制化`@Configuration` 类或在每个特定领域区域定义一个`@Configuration` 类，后一种方式让你可以在需要的时候在某个测试中开启该特殊配置，必要的时候使用`@Import` 注解导入。

另一个配置源困惑是类路径扫描，假设你采用合理的方式组织代码，你需要扫描一个额外的包，你的应用或许类似如下代码：
```
_@SpringBootApplication_
_@ComponentScan({ "com.example.app", "org.acme.another" })_
public class SampleApplication { ... }
```

以上配置会覆盖默认的component scan方式，由扫描你应用的包和子包变为扫描指定的两个包及起子包，比如，一个@DataJpaTest`突然扫描应用的components和用户的配置，将这个定制注解移动到一个单独Configuration类是解决该问题的好方法。

> 如果你不选择以上的方法，你可以在测试类相关的某一层级下创建`@SpringBootConfiguration`，使用该配置代替主应用配置，另一种方式是指定一个测试的源配置，该配置关闭寻找默认源的行为。

#### 3.20 [使用Spock测试Spring Boot应用](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-with-spock)

If you wish to use Spock to test a Spring Boot application, you should add a dependency on Spock’s  `spock-spring`  module to your application’s build.  `spock-spring`  integrates Spring’s test framework into Spock. It is recommended that you use Spock 1.1 or later to benefit from a number of improvements to Spock’s Spring Framework and Spring Boot integration. See  [the documentation for Spock’s Spring module](http://spockframework.org/spock/docs/1.1/modules.html)  for further details.


### 4 [测试工具类集](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-test-utilities)

一些测试工具类在测试你的应用的时候通常是有用的，这些类是`spring-boot`package的一部分。


#### 4.1 [ConfigFileApplicationContextInitializer](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-configfileapplicationcontextinitializer-test-utility)

`ConfigFileApplicationContextInitializer`是一个`ApplicationContextInitializer`，使测试加载Spring Boot `application.properties`文件，如果你不想要`@SpringBootTest`特供的所有特性，可以用该注解，例如：
```
@ContextConfiguration(classes = Config.class,
	initializers = ConfigFileApplicationContextInitializer.class)
```

> 单独使用`ConfigFileApplicationContextInitializer`不提供`@Value("${…​}")`的注入支持，它只保证`application.properties`文件加载到Spring’s  `Environment`，如需`@Value`支持，你需要额外配置一个`PropertySourcesPlaceholderConfigurer`，或使用`@SpringBootTest`，会自动配置一个`PropertySourcesPlaceholderConfigurer`。

#### 4.2 [EnvironmentTestUtils](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-environment-test-utilities)

`EnvironmentTestUtils`可以让你快速添加属性到`ConfigurableEnvironment`或`ConfigurableApplicationContext`中，可以通过`key=value`字符串调用，例如：
```
EnvironmentTestUtils.addEnvironment(env, "org=Spring", "name=Boot");
```

#### 4.3[OutputCapture](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-output-capture-test-utility)

`OutputCapture`是一个JUnit  `Rule`，可以用来捕获 `System.out`或 `System.err`的输出，将其声明为`@Rule`，然后使用`toString()`做断言，如下：
```
import org.junit.Rule;
import org.junit.Test;
import org.springframework.boot.test.rule.OutputCapture;

import static org.hamcrest.Matchers.*;
import static org.junit.Assert.*;

public class MyTest {

	_@Rule_
	public OutputCapture capture = new OutputCapture();

	_@Test_
	public void testName() throws Exception {
		System.out.println("Hello World!");
		assertThat(capture.toString(), containsString("World"));
	}

}
```

#### 4.4 [TestRestTemplate](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-rest-templates-test-utility)

> Spring Framework 5.0提供了一个新的`WebTestClient`，用于 [WebFlux集成测试](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-testing-autoconfigured-webflux-tests "43.3.9 Auto-configured Spring WebFlux Tests")和[WebFlux和MVC端对端测试](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html#boot-features-testing-spring-boot-applications-testing-with-running-server "43.3.4 Testing with a running server"). 相比`TestRestTemplate`，`WebTestClient`为断言提供了流式api。

`TestRestTemplate`是Spring’s  `RestTemplate`的一个更方便的替代品，经常被用于集成测试。通过它可以获取一个普通的template或一个发送Basic认证的template（带着用户名和密码），任何一种情况下，这个template都是测试友好的，不会抛出服务端的异常或错误，推荐使用Apache HTTP Client (version 4.3.2 or better)，但不是强制的，如果你的类路径上有它，`TestRestTemplate`会自动配置一个合适的client，如果你使用 Apache’s HTTP client，有一些额外的测试友好的特性：

-   不允许重定向(因此你能对重定向结果断言).
-   Cookies会被忽略(因此template是无状态的).

`TestRestTemplate`可以在你的集成测试中直接初始化，如下所示：
```
public class MyTest {

	private TestRestTemplate template = new TestRestTemplate();

	_@Test_
	public void testRequest() throws Exception {
		HttpHeaders headers = this.template.getForEntity(
				"http://myhost.example.com/example", String.class).getHeaders();
		assertThat(headers.getLocation()).hasHost("other.example.com");
	}

}
```

如果你使用`@SpringBootTest`注解，且设置`WebEnvironment.RANDOM_PORT`  or  `WebEnvironment.DEFINED_PORT`，你可以注入一个配置完整的`TestRestTemplate`供使用，必要的时候，可以通过`RestTemplateBuilder`这个bean来进行额外的定制化。任何不指定host和port的URL都将连接内嵌server，示例如下：

```
_@RunWith(SpringRunner.class)_
_@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)_
public class SampleWebClientTests {

	_@Autowired_
	private TestRestTemplate template;

	_@Test_
	public void testRequest() {
		HttpHeaders headers = this.template.getForEntity("/example", String.class)
				.getHeaders();
		assertThat(headers.getLocation()).hasHost("other.example.com");
	}

	_@TestConfiguration_
	static class Config {

		_@Bean_
		public RestTemplateBuilder restTemplateBuilder() {
			return new RestTemplateBuilder().setConnectTimeout(1000).setReadTimeout(1000);
		}

	}

}
```
----------

[SpringBoot前一章节](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-jmx.html)

[SpringBoot上一层级章节](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features.html)

[SpringBoot下一章节](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-websockets.html)

42\. Monitoring and Management over JMX

[SpringBoot Reference Doc](https://docs.spring.io/spring-boot/docs/current/reference/html/index.html)

44\. WebSockets
