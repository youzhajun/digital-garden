---
title: Spring 相关问题
draft: false
tags:
  - java
  - 中间件
date: 2023-02-10
---
 # Spring 中 Bean 线程安全吗？

**非线程安全**。默认情况下 `spring bean` 为单例，所有线程共享一个 `bean`。

实际开发中，单例的 `bean` 一般以无状态的方式使用，线程之间的操作不会对 bean 的成员除查询之外的操作，因此线程安全（只是调用 `controller`、`service`、`dao`中的方法）。

# 如何保证 Spring 中的 Bean 线程安全？

1. 将默认的单例 `bean` 模式改为多例 `prototype`
2. 避免使用成员变量
3. 如果必须使用成员变量，可将成员变量保存在 `ThreadLocal` 中
	- ThreadLocal 线程隔离
	
	```java
		@RestController  
		public class DemoController {  


		    // 设置初始值0 
 		    private ThreadLocal<Integer> number = ThreadLocal.withInitial(() -> 0);  
		  
		    @GetMapping("/demo1")  
		    public String demo1() {  
		        number.set(number.get() + 1);  
		        // 输出1
		        return "demo1:" + number.get();  
		    }  
		  
		    @GetMapping("/demo2")  
		    public String demo2() {  
		        number.set(number.get() + 1);
		        // 输出1  
		        return "demo2:" + number.get();  
		    }  
		}
	```

# IO 异常是否能触发 Spring 事务？

**不能**。 `spring` 事务机制默认情况只在 `RuntimeException` 的情况下会触发。 而 IO异常与 `RuntimeException` 异常是平级的都属于 `Exception` 子类。

![](https://obsidian-shanxin.oss-cn-beijing.aliyuncs.com/obsidian/20250702113906484.jpg)

**解决方法**：`Transcational(rollbackFor = Exception.class)`

# Spring 事务失效的情况

- 非 `public` 方法
- 同一 `service`，没有添加事务管理的入口方法调用带有事务的方法。
	- 当 `A()` 内部调用 `B()` 时，实际上是通过 `this.B()` 调用，**绕过了代理对象**，因此事务拦截器不会生效。
	- 特殊情况！！不同 `service`，没有添加事务管理的入口方法调用带有事务的方法，**事务会生效！**（新的 `service` 会生成新的代理对象）
- 没有正确处理异常。
	- `try catch` 住异常后 后仅记录日志
- 同一个 service 自调用问题。
- 在一个新线程中抛出异常，主线程的事务不会回滚。

# Spring 框架的声明周期是怎样的？

1. `Bean` 的定义和注册
	- 解析 `xml` 或注解（`controller`、`service` 等），得到 `BeanDefinittion`
2. 通过  `BeanDefinittion` 实例化对象。实例化的方式有：
	- 构造函数反射、工厂方法、`@Configuration` 类中的 `@Bean` 方法
3. 属性注入
	- 实例化后，容器会根据配置为 Bean 设置属性和依赖。
4. `BeanPostProcessor` 介入
	- 在初始化前后，`BeanPostProcessor`可以修改 Bean 实例：**postProcessBeforeInitialization** 、 **postProcessAfterInitialization**
5. 调用 `init` 初始化方法（如果有的话）
6. 调用初始化回调
7. `bean` 放置 `spring` 容器（`Map`对象）中，使用时从 `map` 中获取
8. `spring` 容器关闭时调用销毁方法

```java
import javax.annotation.PostConstruct;  
import javax.annotation.PreDestroy;  
import org.springframework.beans.BeansException;  
import org.springframework.beans.factory.BeanFactory;  
import org.springframework.beans.factory.BeanFactoryAware;  
import org.springframework.beans.factory.BeanNameAware;  
import org.springframework.beans.factory.DisposableBean;  
import org.springframework.beans.factory.InitializingBean;  
import org.springframework.beans.factory.config.BeanPostProcessor;  
import org.springframework.context.annotation.AnnotationConfigApplicationContext;  
import org.springframework.context.annotation.Bean;  
import org.springframework.context.annotation.Configuration;  
  
// Bean类实现多个生命周期接口  
class MyBean implements BeanNameAware, BeanFactoryAware, InitializingBean, DisposableBean {  
    private String name;  
  
    public MyBean() {  
        System.out.println("1. 构造函数调用");  
    }  
  
    public void setName(String name) {  
        this.name = name;  
        System.out.println("2. 属性注入: " + name);  
    }  
  
    @Override  
    public void setBeanName(String beanName) {  
        System.out.println("3. BeanNameAware: " + beanName);  
    }  
  
    @Override  
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {  
        System.out.println("4. BeanFactoryAware: " + beanFactory.getClass().getSimpleName());  
    }  
  
    @PostConstruct  
    public void postConstruct() {  
        System.out.println("5. @PostConstruct");  
    }  
  
    @Override  
    public void afterPropertiesSet() throws Exception {  
        System.out.println("6. InitializingBean: afterPropertiesSet");  
    }  
  
    public void customInit() {  
        System.out.println("7. init-method: customInit");  
    }  
  
    @PreDestroy  
    public void preDestroy() {  
        System.out.println("8. @PreDestroy");  
    }  
  
    @Override  
    public void destroy() throws Exception {  
        System.out.println("9. DisposableBean: destroy");  
    }  
  
    public void customDestroy() {  
        System.out.println("10. destroy-method: customDestroy");  
    }  
}  
  
// BeanPostProcessor实现类  
class MyBeanPostProcessor implements BeanPostProcessor {  
    @Override  
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {  
        if (bean instanceof MyBean) {  
            System.out.println("BeforeInit: " + beanName);  
        }  
        return bean;  
    }  
  
    @Override  
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {  
        if (bean instanceof MyBean) {  
            System.out.println("AfterInit: " + beanName);  
        }  
        return bean;  
    }  
}  
  
// 配置类  
@Configuration  
class AppConfig {  
    @Bean(initMethod = "customInit", destroyMethod = "customDestroy")  
    public MyBean myBean() {  
        MyBean bean = new MyBean();  
        bean.setName("TestBean");  
        return bean;  
    }  
  
    @Bean  
    public MyBeanPostProcessor beanPostProcessor() {  
        return new MyBeanPostProcessor();  
    }  
}  
  
// 主程序  
public class BeanLifeCycleDemo {  
    public static void main(String[] args) {  
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);  
        MyBean bean = context.getBean(MyBean.class);  
        System.out.println("Bean已就绪: " + bean);  
        context.close();  
    }  
}
```

# Spring 中的三级缓存是什么？

三级缓存是解决 bean 创建过程中循环依赖问题的核心机制。其本身是使用三个`ConcurrentHashMap`作为缓存容器。

- 一级缓存 **（singletonObjects）**
	- （创建、属性注入、初始化方法执行完毕）的单例 bean。
- 二级缓存 **（singletonFactories）**
	- 存储早期曝光的单例工厂对象，用于生成半成品 bean
- 三级缓存 **（earlySingletonObjects）**
	- 存储提前曝光的半成品 bean（已实例化但未完成属性注入和初始化）。

## 三级缓存的具体的工作流程

假设发生了循环依赖的场景，A 依赖了 B ， B 依赖了 A 

1. A 创建过程中
	- A 实例化后，将其工厂对象放入三级缓存
	- A 开始填充属性，发现依赖了 B ，暂停自身创建，触发 B 的创建。
2. B 创建过程中
	- B 实例化后，将其工厂对象放入三级缓存
	- B 开始填充属性，发现依赖了 A ，从三级缓存获取 A 的工厂对象，通过工厂生成 A 的早期引用（半成品），放入二级缓存。
	- B 使用 A 的早期引用完成自身的初始化，存入一级缓存。
3. A 继续创建
	-  A 从一级缓存获取已完成的 B，完成自身属性注入和初始化，最终存入一级缓存。

> 相关源码：

```java
	// 获取单例bean的核心方法  
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {  
	    // 1. 检查一级缓存  
	    Object singletonObject = this.singletonObjects.get(beanName);  
	    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {  
	        // 2. 检查二级缓存  
	        singletonObject = this.earlySingletonObjects.get(beanName);  
	        if (singletonObject == null && allowEarlyReference) {  
	            // 3. 检查三级缓存，获取工厂对象  
	            synchronized (this.singletonObjects) {  
	                singletonObject = this.singletonObjects.get(beanName);  
	                if (singletonObject == null) {  
	                    singletonObject = this.earlySingletonObjects.get(beanName);  
	                    if (singletonObject == null) {  
	                        ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);  
	                        if (singletonFactory != null) {  
	                            // 通过工厂生成早期引用，放入二级缓存  
	                            singletonObject = singletonFactory.getObject();  
	                            this.earlySingletonObjects.put(beanName, singletonObject);  
	                            this.singletonFactories.remove(beanName);  
	                        }  
	                    }  
	                }  
	            }  
	        }  
	    }  
	    return singletonObject;  
	}
```

# Spring AOP 是如何使用代理机制的？
## 实现原理

实现代理的方式有静态代理、`jdk`动态代理、`cglib` 动态代理三种方式。`spring aop` 中的实现机制在于会根据**目标类是否实现了接口**来决定使用哪种代理方式：
- 如果目标类实现了接口，使用 jdk 动态代理
- 如果目标类没有实现接口，使用 cglib 代理
- 如果使用了注解 `@EnableAspectJAutoProxy(proxyTargetClass = true)`，`Spring` 强制使用 `Cglib`。

## 如何使用？

1. 引入 spring aop 的依赖。（springboot 项目默认集成）
	
	```xml
		<!-- Spring AOP 依赖 -->
		<dependency>
		    <groupId>org.springframework.boot</groupId>
		    <artifactId>spring-boot-starter-aop</artifactId>
		</dependency>
	```

2. 编写业务类

	 ```java
		@Service
		public class UserService {
		    public void createUser(String name) {
		        System.out.println("执行：创建用户 " + name);
		    }
		}
	```

3. 编写切面

	```java
		@Aspect  // 表明是个切面
		@Component // 注入到容器
		public class LogAspect {
		
		    // 切点表达式：拦截 UserService 的所有方法
		    @Pointcut("execution(* com.example.service.UserService.*(..))")
		    public void userServiceMethods() {}
		
		    @Before("userServiceMethods()")
		    public void beforeLog(JoinPoint joinPoint) {
		        System.out.println("【前置通知】方法开始执行：" + joinPoint.getSignature().getName());
		    }
		
		    @After("userServiceMethods()")
		    public void afterLog(JoinPoint joinPoint) {
		        System.out.println("【后置通知】方法执行完毕：" + joinPoint.getSignature().getName());
		    }
		
		    @Around("userServiceMethods()")
		    public Object aroundLog(ProceedingJoinPoint pjp) throws Throwable {
		        System.out.println("【环绕通知-前】" + pjp.getSignature().getName());
		        Object result = pjp.proceed();  // 执行原方法
		        System.out.println("【环绕通知-后】" + pjp.getSignature().getName());
		        return result;
		    }
		}

	```

4. 调用

	```java
		@SpringBootApplication
		public class AopDemoApp {
		    public static void main(String[] args) {
		        ApplicationContext context = SpringApplication.run(AopDemoApp.class, args);
		        UserService userService = context.getBean(UserService.class);
		        userService.createUser("张三");
		    }
		}
	```
	
	```markdown
	【前置通知】方法开始执行：createUser
	【环绕通知-前】createUser
	执行：创建用户 张三
	【环绕通知-后】createUser
	【后置通知】方法执行完毕：createUser
	```

