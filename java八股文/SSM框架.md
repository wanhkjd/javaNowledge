# SSM 框架面试八股整理

> 覆盖范围：Spring、Spring MVC、Spring Boot、MyBatis。

## 一、Spring

### 1. Spring 框架中的单例 Bean 是线程安全的吗？

**回答：**

不是绝对线程安全。

当多个用户同时请求同一个服务时，容器会为每个请求分配线程，这些线程可能并发执行业务逻辑。如果业务逻辑中修改了单例 Bean 的成员变量，就需要考虑线程安全问题。Spring 框架本身不会对单例 Bean 做线程安全封装，并发安全需要开发者自行保证。

不过，在实际项目中，常见的 `Service`、`DAO` 等 Spring Bean 通常是**无状态**或**不可变状态**的，因此一般不会产生线程安全问题。若 Bean 中保存了可变状态，例如某些临时数据、用户上下文或 ViewModel 对象，则需要自行处理线程安全。

**常见解决方式：**

- 尽量让 Bean 保持无状态；
- 避免在单例 Bean 中保存请求级别的可变数据；
- 必要时使用同步机制；
- 将 Bean 的作用域从 `singleton` 改为 `prototype`。

---

### 2. 什么是 AOP？

**回答：**

AOP，即**面向切面编程**，用于将与核心业务无关、但会影响多个对象的公共行为和逻辑抽取出来，实现公共模块复用，降低代码耦合。

**常见应用场景：**

- 日志记录；
- 权限校验；
- 事务管理；
- 接口限流；
- 统一异常处理；
- 方法执行耗时统计。

---

### 3. 项目中有没有使用过 AOP？

**回答：**

在后台管理系统中使用过 AOP 记录系统操作日志。

主要思路是：

1. 使用切点表达式定位需要记录日志的方法；
2. 使用环绕通知拦截目标方法；
3. 从通知参数中获取请求信息，例如类名、方法名、注解、请求方式、请求参数等；
4. 执行目标方法；
5. 将操作日志保存到数据库。

---

### 4. Spring 中的事务是如何实现的？

**回答：**

Spring 事务的本质是基于 **AOP 动态代理** 实现的。

执行流程大致如下：

1. 在目标方法执行前开启事务；
2. 执行业务方法；
3. 如果方法正常执行完成，则提交事务；
4. 如果方法执行过程中抛出符合回滚规则的异常，则回滚事务。

---

### 5. Spring 中事务失效的场景有哪些？

**回答：**

常见事务失效场景包括：

1. **异常被捕获但没有继续抛出**  
   方法内部捕获并处理异常后，没有将异常继续抛出，事务代理感知不到异常，就不会触发回滚。

2. **抛出检查型异常但没有配置 `rollbackFor`**  
   默认情况下，Spring 只会对 `RuntimeException` 和 `Error` 回滚。若抛出 checked exception，需要配置：
   ```java
   @Transactional(rollbackFor = Exception.class)
   ```

3. **方法不是 `public` 修饰**  
   `@Transactional` 通常要求作用在 `public` 方法上，否则可能不会生效。

4. **同一个类中方法内部自调用**  
   事务依赖代理对象生效，如果在同一个类中直接调用带事务的方法，可能绕过代理，导致事务失效。

5. **类没有被 Spring 容器管理**  
   如果对象不是 Spring Bean，事务注解不会被 Spring 代理处理。

---

### 6. Spring Bean 的生命周期？

**回答：**

Spring Bean 的生命周期主要包括以下步骤：

1. 读取 `BeanDefinition`，获取 Bean 的定义信息；
2. 调用构造方法实例化 Bean；
3. 进行依赖注入，例如通过 setter 方法或 `@Autowired`；
4. 处理实现了 `Aware` 接口的 Bean；
5. 执行 `BeanPostProcessor` 的前置处理方法；
6. 执行初始化方法，例如 `InitializingBean#afterPropertiesSet()` 或自定义 `init-method`；
7. 执行 `BeanPostProcessor` 的后置处理方法，AOP 代理对象通常可能在此阶段生成；
8. Bean 可以被正常使用；
9. 容器关闭时执行销毁逻辑，例如 `DisposableBean#destroy()` 或自定义 `destroy-method`。

---

### 7. Spring 中的循环依赖是什么？

**回答：**

循环依赖是指两个或多个 Bean 之间相互依赖，形成闭环。例如：A 依赖 B，B 又依赖 A。

Spring 可以通过**三级缓存**解决大部分单例 Bean 的属性注入循环依赖问题。

**三级缓存：**

1. **一级缓存：`singletonObjects`**  
   保存已经完成初始化的单例 Bean。

2. **二级缓存：`earlySingletonObjects`**  
   保存尚未完成完整生命周期的早期 Bean。

3. **三级缓存：`singletonFactories`**  
   保存 `ObjectFactory`，用于提前暴露 Bean 引用，必要时生成代理对象。

---

### 8. Spring 循环依赖的解决流程？

**回答：**

以 A 依赖 B、B 依赖 A 为例：

1. 实例化 A，并将 A 的 `ObjectFactory` 放入三级缓存；
2. A 进行属性注入时发现需要 B，于是开始创建 B；
3. 实例化 B，并将 B 的 `ObjectFactory` 放入三级缓存；
4. B 进行属性注入时发现需要 A；
5. Spring 从三级缓存中获取 A 的 `ObjectFactory`，生成 A 的早期引用，并放入二级缓存；
6. B 注入 A 后完成初始化，放入一级缓存；
7. A 继续注入已经创建完成的 B；
8. A 完成初始化后放入一级缓存，并清理二级缓存中的早期引用。

---

### 9. 构造方法出现循环依赖怎么解决？

**回答：**

构造方法循环依赖 Spring 默认无法解决，因为构造方法是 Bean 创建时最先执行的阶段，此时对象还没有完成实例化，无法提前暴露引用。

**解决方式：**

- 使用 `@Lazy` 懒加载；
- 改为 setter 注入；
- 重构代码，拆分职责，避免循环依赖；
- 引入中间服务或事件机制解耦。

---

### 10. Spring 的常见注解有哪些？

![[java八股文/资源/Spring的常见注解.png]]

常见注解包括：

- `@Component`、`@Controller`、`@Service`、`@Repository`：声明 Bean；
- `@Autowired`：按类型自动注入；
- `@Qualifier`：配合 `@Autowired` 按名称注入；
- `@Scope`：指定 Bean 作用域；
- `@Configuration`：声明配置类；
- `@ComponentScan`：指定组件扫描路径；
- `@Bean`：将方法返回值注册为 Bean；
- `@Import`：导入配置类或组件；
- `@Aspect`、`@Before`、`@After`、`@Around`、`@Pointcut`：AOP 相关注解。

---

## 二、Spring MVC

### 11. Spring MVC 的执行流程？

**回答：**

以前后端分离场景为例，Spring MVC 的执行流程如下：

1. 用户请求发送到前端控制器 `DispatcherServlet`；
2. `DispatcherServlet` 调用 `HandlerMapping` 查找具体处理器；
3. `HandlerMapping` 返回处理器对象以及拦截器链；
4. `DispatcherServlet` 调用 `HandlerAdapter`；
5. `HandlerAdapter` 适配并执行具体的 Controller 方法；
6. Controller 方法返回结果，通常配合 `@ResponseBody`；
7. `HttpMessageConverter` 将返回对象转换为 JSON；
8. 将响应结果返回给前端。

---

### 12. Spring MVC 常见注解有哪些？

**回答：**

![[java八股文/资源/SpringMVC常见的注解.png]]

常见注解包括：

- `@RequestMapping`：映射请求路径；
- `@GetMapping`、`@PostMapping`、`@PutMapping`、`@DeleteMapping`：请求方法映射；
- `@RequestBody`：接收 HTTP 请求体中的 JSON 数据；
- `@RequestParam`：获取请求参数；
- `@PathVariable`：获取路径参数；
- `@ResponseBody`：将方法返回值转换为响应体数据；
- `@RequestHeader`：获取请求头数据；
- `@RestController`：等价于 `@Controller + @ResponseBody`。

---

## 三、Spring Boot

### 13. Spring Boot 自动配置原理？

**回答：**

Spring Boot 自动配置主要基于 `@SpringBootApplication` 注解。

`@SpringBootApplication` 由以下注解组合而成：

- `@SpringBootConfiguration`；
- `@EnableAutoConfiguration`；
- `@ComponentScan`。

其中，`@EnableAutoConfiguration` 是核心。它通过 `@Import` 导入自动配置选择器，读取自动配置类信息，并根据条件注解决定是否将对应配置类中的 Bean 注册到 Spring 容器中。

> 说明：Spring Boot 2.x 主要通过 `META-INF/spring.factories` 加载自动配置类；Spring Boot 3.x 主要使用 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`。

---

### 14. Spring Boot 常见注解有哪些？

**回答：**

常见注解包括：

- `@SpringBootApplication`：Spring Boot 启动类核心注解；
- `@SpringBootConfiguration`：声明当前类是 Spring Boot 配置类；
- `@EnableAutoConfiguration`：开启自动配置；
- `@ConfigurationProperties`：绑定配置文件属性；
- `@ConditionalOnClass`、`@ConditionalOnMissingBean`、`@ConditionalOnProperty`：条件装配相关注解；
- `@RestController`、`@GetMapping`、`@PostMapping`：Web 开发常用注解。

---

## 四、MyBatis

### 15. MyBatis 执行流程？

**回答：**

![[java八股文/资源/MyBatis的执行流程.png]]

MyBatis 的执行流程如下：

1. 读取 MyBatis 配置文件，例如 `mybatis-config.xml`；
2. 构建会话工厂 `SqlSessionFactory`；
3. 通过 `SqlSessionFactory` 创建 `SqlSession`；
4. 通过 `SqlSession` 调用 Mapper 接口方法；
5. 底层由 `Executor` 执行器处理 SQL；
6. 根据 `MappedStatement` 获取 SQL、参数映射和结果映射信息；
7. 完成输入参数映射；
8. 执行 SQL；
9. 完成结果集映射并返回 Java 对象。

---

### 16. MyBatis 是否支持延迟加载？

**回答：**

支持。

延迟加载指的是：查询主数据时，暂时不查询关联数据，等代码真正访问关联数据时，再去数据库查询。

可以通过配置项控制是否开启延迟加载：

```xml
<setting name="lazyLoadingEnabled" value="true"/>
```

---

### 17. MyBatis 延迟加载的底层原理是什么？

**回答：**

MyBatis 延迟加载底层主要基于**动态代理**实现，常见方式包括 CGLIB 或 Javassist。

大致流程如下：

1. 为目标对象创建代理对象；
2. 当访问延迟加载属性时，代理对象拦截方法调用；
3. 判断该属性是否已经加载；
4. 如果尚未加载，则执行对应 SQL 查询；
5. 查询完成后，将结果设置到属性中并返回。

---

### 18. MyBatis 的一级缓存和二级缓存用过吗？

**回答：**

MyBatis 缓存分为一级缓存和二级缓存。

| 缓存类型 | 作用域 | 是否默认开启 | 说明 |
| --- | --- | --- | --- |
| 一级缓存 | `SqlSession` | 默认开启 | 基于 `PerpetualCache`，本质是 HashMap 本地缓存 |
| 二级缓存 | `Mapper/Namespace` | 需要配置开启 | 多个 `SqlSession` 可以共享同一 Namespace 下的缓存 |

---

### 19. MyBatis 的二级缓存什么时候会清理？

**回答：**

当对应作用域中执行了新增、修改、删除操作后，默认会清空该作用域下的缓存数据。

也就是说：

- 一级缓存：同一个 `SqlSession` 中执行增删改操作后会清理；
- 二级缓存：同一个 `Namespace` 下执行增删改操作后会清理相关缓存。
