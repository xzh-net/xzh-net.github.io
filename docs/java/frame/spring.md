# Spring的底层原理与源码实现

## 1. Spring核心体系结构

### 1.1 Spring核心知识点

![](../../assets/_images/java/frame/spring/spring.png)

### 1.2 依赖注入的工作流程

![](../../assets/_images/java/frame/spring/ioc.png)


### 1.3 AOP的工作流程

![](../../assets/_images/java/frame/spring/aop.png)

### 1.4 Bean的生命周期原理详解与源码分析

![](../../assets/_images/java/frame/spring/bean.png)

### 1.5 事务隔离级别流程

![](../../assets/_images/java/frame/spring/transaction.png)

### 1.6 ApplicationContext和BeanFactory架构图

两者都是可以加载bean的，但是相比之下ApplicationContext 提供了更多的扩展功能。

如果一个类实现了ApplicationContextAware接口，spring 创建bean的时候会自动注入ApplicationContext，大家获取ApplicationContext的成本很低，所以ApplicationContext被大家更多使用

![](../../assets/_images/java/frame/spring/applicationcontext.png)

### 1.7 BeanFactory架构

![](../../assets/_images/java/frame/spring/beanfactory.png)



## 2. Spring结构实现

### 2.1 BeanDefinition的底层原理与源码详解

BeanDefinition在Spring中是用来描述Bean对象的，其不是一个bean实例，仅仅是包含bean实例的所有信息，比如属性值、构造器参数以及其他信息。Bean对象创建是根据BeanDefinitionc中描述的信息来创建的，BeanDefinition存在的作用是为了可以方便的进行修改属性值和其他元信息，比如通过BeanFactoryPostProcessor进行修改一些信息，然后在创建Bean对象的时候就可以结合原始信息和修改后的信息创建对象了

#### 2.1.1 属性信息

```java
@SuppressWarnings("serial")
public abstract class AbstractBeanDefinition extends BeanMetadataAttributeAccessor
		implements BeanDefinition, Cloneable {
	//作用范围
	public static final String SCOPE_DEFAULT = "";
	public static final int AUTOWIRE_NO = AutowireCapableBeanFactory.AUTOWIRE_NO;
	public static final int AUTOWIRE_BY_NAME = AutowireCapableBeanFactory.AUTOWIRE_BY_NAME;
	public static final int AUTOWIRE_BY_TYPE = AutowireCapableBeanFactory.AUTOWIRE_BY_TYPE;
	public static final int AUTOWIRE_CONSTRUCTOR = AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR;
	public static final int AUTOWIRE_AUTODETECT = AutowireCapableBeanFactory.AUTOWIRE_AUTODETECT;
	//表示是否存在依赖的
	public static final int DEPENDENCY_CHECK_NONE = 0;
	public static final int DEPENDENCY_CHECK_OBJECTS = 1;
	public static final int DEPENDENCY_CHECK_SIMPLE = 2;
	public static final int DEPENDENCY_CHECK_ALL = 3;
	public static final String INFER_METHOD = "(inferred)";
	//
	@Nullable
	private volatile Object beanClass;
	//作用范围
	@Nullable
	private String scope = SCOPE_DEFAULT;
	//是不是抽象类
	private boolean abstractFlag = false;
	@Nullable
	private Boolean lazyInit;
	//表示自动注入模式
	private int autowireMode = AUTOWIRE_NO;
	//表示是否存在依赖
	private int dependencyCheck = DEPENDENCY_CHECK_NONE;
    //用来表示一个bean的实例化依靠另一个bean先实例化，对应bean属性depend-on
	@Nullable
	private String[] dependsOn;
	//false时容器在查找自动装配对象时，将不考虑该bean,但该bean本身还是可以自动注入其他bean
	private boolean autowireCandidate = true;
	//自动装配出现多个bean候选者是，将作为首选者，对应bean属性primary
	private boolean primary = false;
	//用于记录Qualifier，对应子元素qualifier
	private final Map<String, AutowireCandidateQualifier> qualifiers = new LinkedHashMap<>();

	@Nullable
	private Supplier<?> instanceSupplier;
	//允许访问非公开的构造器和方法，程序设置
	private boolean nonPublicAccessAllowed = true;
	private boolean lenientConstructorResolution = true;
	//对应bean属性factory-bean
	@Nullable
	private String factoryBeanName;
	//对应bean属性factory-method
	@Nullable
	private String factoryMethodName;
    //记录构造函数注入属性，对应bean属性constructor-arg
	@Nullable
	private ConstructorArgumentValues constructorArgumentValues;
	@Nullable
	private MutablePropertyValues propertyValues;

	private MethodOverrides methodOverrides = new MethodOverrides();
	//对应bean属性init-method	
	@Nullable
	private String initMethodName;
    //对应bean属性destroy-method
	@Nullable
	private String destroyMethodName;

	private boolean enforceInitMethod = true;

	private boolean enforceDestroyMethod = true;
	//是用户定义的而不是应用程序本身定义时为false，创建AOP时为true，程序设置
	private boolean synthetic = false;
	 /**
     *  ROLE_APPLICATION = 0 :用户
     *  ROLE_SUPPORT = 1：某些复杂配置一部分
     *  ROLE_INFRASTRUCTURE = 2：完全内部使用，与用户无关
     *  定义这个bean的应用
     */ 
	private int role = BeanDefinition.ROLE_APPLICATION;
	//bean的描述信息
	@Nullable
	private String description;
	//bean定义的资源
	@Nullable
	private Resource resource;


	/**
	 * Create a new AbstractBeanDefinition with default settings.
	 */
	protected AbstractBeanDefinition() {
		this(null, null);
	}

	/**
	 * Create a new AbstractBeanDefinition with the given
	 * constructor argument values and property values.
	 */
	protected AbstractBeanDefinition(@Nullable ConstructorArgumentValues cargs, @Nullable MutablePropertyValues pvs) {
		this.constructorArgumentValues = cargs;
		this.propertyValues = pvs;
	}

	/**
	 * Create a new AbstractBeanDefinition as a deep copy of the given
	 * bean definition.
	 * @param original the original bean definition to copy from
	 */
	protected AbstractBeanDefinition(BeanDefinition original) {
		setParentName(original.getParentName());
		setBeanClassName(original.getBeanClassName());
		setScope(original.getScope());
		setAbstract(original.isAbstract());
		setFactoryBeanName(original.getFactoryBeanName());
		setFactoryMethodName(original.getFactoryMethodName());
		setRole(original.getRole());
		setSource(original.getSource());
		copyAttributesFrom(original);

		if (original instanceof AbstractBeanDefinition) {
			AbstractBeanDefinition originalAbd = (AbstractBeanDefinition) original;
			if (originalAbd.hasBeanClass()) {
				setBeanClass(originalAbd.getBeanClass());
			}
			if (originalAbd.hasConstructorArgumentValues()) {
				setConstructorArgumentValues(new ConstructorArgumentValues(original.getConstructorArgumentValues()));
			}
			if (originalAbd.hasPropertyValues()) {
				setPropertyValues(new MutablePropertyValues(original.getPropertyValues()));
			}
			if (originalAbd.hasMethodOverrides()) {
				setMethodOverrides(new MethodOverrides(originalAbd.getMethodOverrides()));
			}
			Boolean lazyInit = originalAbd.getLazyInit();
			if (lazyInit != null) {
				setLazyInit(lazyInit);
			}
			setAutowireMode(originalAbd.getAutowireMode());
			setDependencyCheck(originalAbd.getDependencyCheck());
			setDependsOn(originalAbd.getDependsOn());
			setAutowireCandidate(originalAbd.isAutowireCandidate());
			setPrimary(originalAbd.isPrimary());
			copyQualifiersFrom(originalAbd);
			setInstanceSupplier(originalAbd.getInstanceSupplier());
			setNonPublicAccessAllowed(originalAbd.isNonPublicAccessAllowed());
			setLenientConstructorResolution(originalAbd.isLenientConstructorResolution());
			setInitMethodName(originalAbd.getInitMethodName());
			setEnforceInitMethod(originalAbd.isEnforceInitMethod());
			setDestroyMethodName(originalAbd.getDestroyMethodName());
			setEnforceDestroyMethod(originalAbd.isEnforceDestroyMethod());
			setSynthetic(originalAbd.isSynthetic());
			setResource(originalAbd.getResource());
		}
		else {
			setConstructorArgumentValues(new ConstructorArgumentValues(original.getConstructorArgumentValues()));
			setPropertyValues(new MutablePropertyValues(original.getPropertyValues()));
			setLazyInit(original.isLazyInit());
			setResourceDescription(original.getResourceDescription());
		}
	}
}
```

### 2.2 BeanFactoryPostProcessor的底层原理与源码详解

使用bean工厂的后置处理器可以达到两个目的1.注册bean 2.已有bean元数据的修改

注意：此时的单例池还没有成品对象，还未进行实例化

beanFactory.registerSingleton 直接将对象放入单例池

beanFactory.createBean 创建一个受管理的bean

```java
@Component
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		// 注入方式1
		GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
		beanDefinition.setBeanClass(User.class);
		beanDefinition.getPropertyValues().addPropertyValue("id", 100);
		((DefaultListableBeanFactory) beanFactory).registerBeanDefinition("user100", beanDefinition);

//		beanFactory.registerSingleton  ???

		// 修改之前bean中的属性
		GenericBeanDefinition beanDefinition2 = (GenericBeanDefinition)beanFactory.getBeanDefinition("user100");
		
		System.err.println(beanDefinition2);
		
		String[] beanStr = beanFactory.getBeanDefinitionNames();
		for (String beanName : beanStr) {
			if ("user100".equals(beanName)) {
				BeanDefinition bd = beanFactory.getBeanDefinition(beanName);
				MutablePropertyValues m = bd.getPropertyValues();
				if (m.contains("id")) {
					m.addPropertyValue("id", "111111111");
				}
			}
		}

	}

}
```

### 2.3 BeanDefinitionRegistry的底层原理与源码详解

### 2.4 BeanDefinitionRegistryPostProcessor的底层原理与源码详解

### 2.5 BeanPostProcessor的底层原理与源码详解

BeanFactoryPostProcessor和BeanPostProcessor这两个接口都是初始化bean时对外暴露的入口之一

AutowiredAnnotationBeanPostProcessor


```java
public class CustomBeanPostProcessor implements BeanPostProcessor {

    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if(bean instanceof Student){
            // 如果当前的bean是Student,则打印日志
            System.out.println("postProcessBeforeInitialization bean : " + beanName);
        }
        return bean;
    }

    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if(bean instanceof Student){
            System.out.println("postProcessAfterInitialization bean : " + beanName);
        }
        return bean;
    }
}
```

### 2.6 BeanNameGenerator的底层原理与源码详解

### 2.7 aware接口

![](../../assets/_images/java/frame/spring/aware.png)

#### 2.7.1 BeanNameAware

`dubbo`

#### 2.7.2 BeanFactoryAware

通过实现BeanFactoryAware接口将bean注册到spring容器中

```java
@Component
public class LocationRegister implements BeanFactoryAware {
    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        DefaultListableBeanFactory listableBeanFactory = (DefaultListableBeanFactory)beanFactory;
        //方式1
        Location location = new Location();
        listableBeanFactory.registerSingleton(Location.class.getName(),location);

        //方式2
        BeanDefinition locationBeanDefinition = new RootBeanDefinition(Location.class);
        listableBeanFactory.registerBeanDefinition(Location.class.getName(),locationBeanDefinition);
    }
}
```


## 3. 总结

### 3.1 依赖注入方式到底有几种？

1. 手动注入
   - setter注入
   - 构造器注入

```java
public class UserService{

	private OrderService orderService;//必须提供set方法

	public void setOrderService(OrderService orderService){
		this.orderService = orderService;
	}
	public void test(){
		System.out.println(orderService);
	}
}
```

```xml
<bean id="userService" class="com.xuzhihao.service.UserService"></bean>

<bean id="orderService" class="com.xuzhihao.service.OrderService">
	<property name="userService" ref="userService"></property> //第一种setter注入
	<constructor-arg index="0" ref="userService"></constructor-arg>    //第二种方式注入属性值
</bean>

```
1. 自动注入
   - XML自动注入
     - setter注入
     - 构造器注入
   - @Autowired注解的自动注入
     - 属性
     - 构造方法
     - 普通方法

```java
public class UserService{

	// @Autowired
	private OrderService orderService;//必须提供set方法

	public void setOrderService(OrderService orderService){
		this.orderService = orderService;
	}
	public void test(){
		System.out.println(orderService);
	}
}
```

```xml
<bean id="userService" class="com.xuzhihao.service.UserService"></bean>

<bean id="orderService" class="com.xuzhihao.service.OrderService" autowird="byName">
</bean>

```

### 3.2 如何解决循环依赖，三级缓存解决循环依赖问题的关键是什么？为什么通过提前暴漏对象能解决

实例化和初始化分开操作，在中间过程中给其他对象赋值的时候，并不是一个完整的对象，而是把半成品对象赋值给了其他对象，使用三级缓存的本质在于aop代理问题

1. 一级缓存，用于保存beanName和创建bean实例之间的关系  singletonObjects
2. 二级缓存，提前曝光的单例对象的cache，存放原始的 bean 对象（尚未填充属性），用于解决循环依赖 earlySingletonObjects
3. 三级缓存，单例对象工厂的cache，存放 bean 工厂对象，用于解决循环依赖，函数式接口 仅有一个方法 可以传入lamda表达式 可以是匿名内部类 调用getObject来执行具体的逻辑 实际调用的是createBean singletonFactories
    
如果使用一级缓存能否不解决问题？

不能，在整个过程中，缓存中放的是半成品(二级early)和成品对象(一级)，如果只有一级缓存，半成品和成品都放在一级中，由于半成品无法使用 需要做额外的判断，因此把成品和半成品存放空间分割开来

为什么使用三级缓存能解决aop代理问题？

当一个对象需要被代理的时候，整个创建过程包含两个对象，原对象和代理生成对象bean默认的都是单例，在整个生命周期的处理环节中一个beanName不能对应多个对象 ,所有对代理对象的时候覆盖原对象

如何知道什么时候使用代理对象？

因为不知道什么时候调用，所以通过一个匿名内部类的方式在使用的方式覆盖原对象，保证全局唯一，这就是三级缓存的本质

cglib核心代理处理ObjenesisCglibAopProxy.java



## 4. 常用注解

### 4.1 @Import

实现ImportSelector接口是springboot自动装配的核心原理

导入ImportBeanDefinitionRegistrar的实现类方式是`Feign`整合的关键

导入bean的三种方式

#### 4.1.1 导入@Configuration注解的配置类或者是直接将bean导入，导入的bean以全限定类名注册到容器中
   
```java
@Data
public class User {
	private int id;
	private String name;
}

@ComponentScan(value = "com.xuzhihao")
@Import({ User.class })
public class SpringConfiguration {

}
//输出结果
AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(SpringConfiguration.class);
String[] names = ac.getBeanDefinitionNames();
for (String name : names) {
    System.out.println(name);
}

```

#### 4.1.2 导入实现ImportSelector接口或子接口DeferredImportSelector的类

重写ImportSelector的selectImports方法 两种使用方式 1. 返回值注册 2. 通过beanFactory注册

```java
@Import({ MyImportSelector.class })
public class SpringConfiguration {

}

//单独的class
@Data
public class Order {
	private int id;
	private String name;
}
//单独的class
@Data
public class Stock {
	private int id;
	private String name;
}



public class MyImportSelector implements ImportSelector, BeanFactoryAware {

	private BeanFactory beanFactory;

	@Override
	public String[] selectImports(AnnotationMetadata annotationMetadata) {
		GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
		beanDefinition.setBeanClass(Stock.class);
		beanDefinition.getPropertyValues().addPropertyValue("id", 123456);
		((DefaultListableBeanFactory) beanFactory).registerBeanDefinition("Stock", beanDefinition);//beanName注册方式
		return new String[] { "com.xuzhihao.domain.Order" };//全限定类型注册
	}

	@Override
	public void setBeanFactory(BeanFactory beanFactory) {
		this.beanFactory = beanFactory;
	}
}

//输出结果
AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(SpringConfiguration.class);
String[] names = ac.getBeanDefinitionNames();
for (String name : names) {
    System.out.println(name);
}
System.out.println(ac.getBean("Stock"));
```


#### 4.1.3 导入ImportBeanDefinitionRegistrar的实现类

```java
@ComponentScan(value = "com.xuzhihao")
@Import({ MyImportBeanDefinitionRegistrar.class })
public class SpringConfiguration {

}

public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

	@Override
	public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry registry) {
		RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(Stock.class);
		registry.registerBeanDefinition("Stock2", rootBeanDefinition);//注册器注册bean
	}

}

//输出结果
AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(SpringConfiguration.class);
String[] names = ac.getBeanDefinitionNames();
for (String name : names) {
    System.out.println(name);
}
System.out.println(ac.getBean("Stock2"));

```


### 4.2 @Conditional 条件注入

1. ConditionalOnClass              应用中包含某个类的时候配置生效
2. ConditionalOnMissingClass       应用中不包含某个类的时候配置生效
3. ConditionalOnBean               容器中存在指定class的示例对象时配置生效
4. ConditionalOnMissingBean        容器中不存在指定class的示例对象时配置生效
5. ConditionalOnProperty           指定参数值符合要求时配置生效
6. ConditionalOnResource           存在指定资源时配置生效
7. ConditionalOnWebApplication     当前处于web环境时配置生效（WebApplicationContext）
8. ConditionalOnNotWebApplication  当前处于非web环境时配置生效
9. ConditionalOnExpression         指定参数值符合要求时配置生效，与ConditionalOnProperty区别在于这里使用SpringEL表达式


```java
//按条件注入 true表示注入
//Condition中获取beanFactory、registry也可以批量注册
public class Application {

	public static void main(String[] args) {
		AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(SpringConfiguration.class);
		String[] names = ac.getBeanDefinitionNames();
		for (String name : names) {
			System.out.println(name);
		}
		ac.close();
	}
}
```

条件1

```java
package com.xuzhihao.conditional;

import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.beans.factory.support.DefaultListableBeanFactory;
import org.springframework.beans.factory.support.GenericBeanDefinition;
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.env.Environment;
import org.springframework.core.type.AnnotatedTypeMetadata;

import com.xuzhihao.service.Unix;

/**
 * 条件判断 满足条件的时候注册
 * 
 * @author Administrator
 *
 */
public class LinuxConditional implements Condition {
	@Override
	@SuppressWarnings("unused")
	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();// 获取bean工厂
		
		ClassLoader classLoader = context.getClassLoader();// 获取类加载器
		Environment environment = context.getEnvironment();// 获取环境变量
		BeanDefinitionRegistry registry = context.getRegistry();// 获取bean自定义注册器

		GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
		beanDefinition.setBeanClass(Unix.class);
		beanDefinition.getPropertyValues().addPropertyValue("name", "Unix3");
		((DefaultListableBeanFactory) beanFactory).registerBeanDefinition("Unix3", beanDefinition);

		String property = environment.getProperty("os.name");
		if (property.contains("linux")) {
			return true;
		}
		return false;
	}
}
```

条件2 

```java
package com.xuzhihao.conditional;

import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;
import org.springframework.beans.factory.support.BeanDefinitionRegistry;
import org.springframework.beans.factory.support.RootBeanDefinition;
import org.springframework.context.annotation.Condition;
import org.springframework.context.annotation.ConditionContext;
import org.springframework.core.env.Environment;
import org.springframework.core.type.AnnotatedTypeMetadata;

import com.xuzhihao.service.Unix;

/**
 * 条件判断 满足条件的时候注册
 * 
 * @author Administrator
 *
 */
public class WindowsConditional implements Condition {

	@Override
	@SuppressWarnings("unused")
	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();// 获取bean工厂
		ClassLoader classLoader = context.getClassLoader();// 获取类加载器
		Environment environment = context.getEnvironment();// 获取环境变量
		BeanDefinitionRegistry registry = context.getRegistry();// 获取bean自定义注册器

		RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(Unix.class);
		rootBeanDefinition.getPropertyValues().addPropertyValue("name", "Unix");
		registry.registerBeanDefinition("Unix2", rootBeanDefinition);
		
		String property = environment.getProperty("os.name");
		if (property.contains("Windows")) {
			return true;
		}
		return false;
	}

}

```

扫描

```java
@ComponentScan(value = "com.xuzhihao")
public class SpringConfiguration {

}
```

实体

```java
@Data
@Conditional(value = { LinuxConditional.class })
@Component
public class Linux {
	
}

//

@Data
public class Unix {
	private String name;

}

//

@Data
@Conditional(value = { WindowsConditional.class })
@Component
public class Windows {

}

```

### 4.3 @Bean 声明实例

```java
public class Application {

	public static void main(String[] args) {
		AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(SpringConfiguration.class);
		String[] names = ac.getBeanDefinitionNames();
		for (String name : names) {
			System.out.println(name);
		}
		ac.close();
	}

}
```

```java
@ComponentScan(value = "com.xuzhihao")
public class SpringConfiguration {

	@Bean(initMethod = "initMethod", destroyMethod = "destroyMethod")
	public User user() {
		return new User(1, "admin");
	}
}
```

```java
package com.xuzhihao.domain;

import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;

import org.springframework.beans.factory.DisposableBean;
import org.springframework.beans.factory.InitializingBean;

import lombok.Data;

/**
 * 用户
 * 
 * @author Administrator
 *
 */
@Data
public class User implements InitializingBean, DisposableBean {
	private int id;
	private String name;

	// 1.@PostConstruct
	// 2.InitializingBean
	// 3.initMethod
	public void initMethod() {
		System.out.println("User initMethod");

	}

	public void destroyMethod() {
		System.out.println("User destroyMethod");
	}

	@Override
	public void afterPropertiesSet() throws Exception {
		System.out.println("User@InitializingBean@afterPropertiesSet");
	}

	@Override
	public void destroy() throws Exception {
		System.out.println("User@DisposableBean@destroy");
	}

	@PostConstruct
	void postConstruct() {
		System.out.println("User@PostConstruct====@JSR250");
	}

	@PreDestroy
	void preDestroy() {
		System.out.println("User@PreDestroy====@JSR250");
	}

	public User(int id, String name) {
		super();
		this.id = id;
		this.name = name;
	}
}

```

声明bean的三种方式
1. xml注入
2. 基于注解
    - @Component：当对组件的层次难以定位的时候使用这个注解
    - @Controller：表示控制层的组件
    - @Service：表示业务逻辑层的组件
    - @Repository：表示数据访问层的组件
3. @Bean作用在方法上


### 4.4 @ComponentScan 组件扫描

```java
//扫描方式1：默认路径下的所有 
//扫描方式2：自定义扫描器(包含规则、排除规则)
@ComponentScan(value = "com.xuzhihao", excludeFilters = {
		@Filter(type = FilterType.CUSTOM, classes = MyComponentScan.class) })
public class SpringConfiguration {
}
```

```java

public enum FilterType {
    //按照注解方式
	ANNOTATION,
    
    //按照指定类型
	ASSIGNABLE_TYPE,
    
    //使用ASPECTJ表达式指定
	ASPECTJ,
    
    //使用正则表达式进行指定
	REGEX,
    
    //自己实现TypeFilter接口，使用自定义规则
	CUSTOM
```

```java
package com.evan.component.config;
 
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.ComponentScan.Filter;
import org.springframework.context.annotation.FilterType;
import org.springframework.stereotype.Controller;
import org.springframework.stereotype.Repository;
import org.springframework.stereotype.Service;
 
// @ComponentScan("包路径") 会自动扫描包路径下面的所有@Controller、@Service、@Repository、@Component 的类
// includeFilters 指定哪些类需要过滤。
// excludeFilters 指定哪些类不需要过滤。
// @Filter的type指定过滤的类型，classes指定的扫描的规则类。
@ComponentScan(value = "com.evan.component",
		includeFilters = {
				@Filter(type = FilterType.ANNOTATION, classes = {Service.class}),
				@Filter(type = FilterType.ANNOTATION, classes = {Controller.class}),
				@Filter(type = FilterType.CUSTOM,classes = {EvanTypeFilter.class})
		},
		excludeFilters = {
				@Filter(type = FilterType.ANNOTATION, classes = {Repository.class}),
		},useDefaultFilters = false)
public class MyConfig {
 
}
```

```java

package com.evan.component.config;
 
import org.springframework.core.io.Resource;
import org.springframework.core.type.AnnotationMetadata;
import org.springframework.core.type.ClassMetadata;
import org.springframework.core.type.classreading.MetadataReader;
import org.springframework.core.type.classreading.MetadataReaderFactory;
import org.springframework.core.type.filter.TypeFilter;
 
 
public class EvanTypeFilter implements TypeFilter {
	@Override
	public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) {
		//获取当前类注解的信息
		AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
		//获取当前正在扫描的类信息
		ClassMetadata classMetadata = metadataReader.getClassMetadata();
		//获取当前类资源(类的路径)
		Resource resource = metadataReader.getResource();
 
		String className = classMetadata.getClassName();
		System.out.println("当前扫描的类："+className);
		//当类的类名以EvanTest开始，则匹配成功,返回true
		//excludeFilters返回true，会被过滤掉
		//includeFilters返回true，会通过
		if (className.contains("EvanTest")) {
			return true;
		}
		return false;
	}
}
```


### 4.5 @PropertySource 注解加载指定的属性文件

```java
//读取配置文件内容
public class Application {

	public static void main(String[] args) {
		AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(SpringConfiguration.class);
		JdbcTemplate jdbcTemplate = ac.getBean("jdbcTemplate", JdbcTemplate.class);
		jdbcTemplate.update("insert into account(id,name)values(?,?)", "1231456", "lisi");
		System.out.println(ac.getBean("port"));

		ac.close();
	}

}
```

配置

```java
@ComponentScan(value = "com.xuzhihao")
@PropertySource(value = "classpath:application.properties")
@PropertySource(factory = YamlPropertySourceFactory.class, value = "classpath:application.yml")
public class SpringConfiguration {

}

//

public class YamlPropertySourceFactory implements PropertySourceFactory {

	@Override
	public PropertySource<?> createPropertySource(String name, EncodedResource resource) throws IOException {
		YamlPropertiesFactoryBean factory = new YamlPropertiesFactoryBean();
		factory.setResources(resource.getResource());
		factory.afterPropertiesSet();
		Properties properties = factory.getObject();
		return (name != null ? new PropertiesPropertySource(name, properties)
				: new PropertiesPropertySource(resource.getResource().getFilename(), properties));
	}
}

```

测试

```java
package com.xuzhihao.jdbc;

import javax.sql.DataSource;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.datasource.DriverManagerDataSource;
import org.springframework.stereotype.Component;

@Component
public class JdbcConfig {
	@Value("${jdbc.driver}")
	private String driver;
	@Value("${jdbc.url}")
	private String url;
	@Value("${jdbc.username}")
	private String username;
	@Value("${jdbc.password}")
	private String password;
	
	//YML重写
	@Value("${server.port}")
	public String port;

	@Bean
	public String port() {
		return port;
	}

	@Bean("jdbcTemplate")
	public JdbcTemplate createJdbcTemplate(DataSource dataSource) {
		return new JdbcTemplate(dataSource);
	}

	@Bean("dataSource")
	public DataSource createDataSource() {
		DriverManagerDataSource dataSource = new DriverManagerDataSource();
		dataSource.setDriverClassName(driver);
		dataSource.setUrl(url);
		dataSource.setUsername(username);
		dataSource.setPassword(password);
		return dataSource;
	}
}
```

application.properties

```
jdbc.driver=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql://debug-registry:3306/mall
jdbc.username=root
jdbc.password=root
```

application.yml

```yml
server:
   port: 8080
spring:
   profiles:
      active: dev
   application:
      name: spring05-propertysource
```

### 4.6 @EnableAspectJAutoProxy

#### 4.6.1 准备

```java
public static void main(String[] args) {
		try (AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext("config")) {//按包扫描
			AccountService accountService = (AccountService) ac.getBean("proxyAccountService");
			accountService.transfer("aaa", "bbb", 100d);
		}
	}
```

config包下文件

```java
@Configuration
@ComponentScan("com.xuzhihao")
@Import(JdbcConfig.class)
@PropertySource(value = "classpath:jdbc.yml", factory = YmlPropertySourceFactory.class)
@EnableAspectJAutoProxy//开启spring注解aop配置的支持
public class SpringConfiguration {
}


//
/**
 * 自定义yml文件解析的工厂类
 */
public class YmlPropertySourceFactory implements PropertySourceFactory {

	@Override
	public PropertySource<?> createPropertySource(String name, EncodedResource resource) throws IOException {
		// 1.创建yaml文件解析工厂
		YamlPropertiesFactoryBean factoryBean = new YamlPropertiesFactoryBean();
		// 2.设置要解析的内容
		factoryBean.setResources(resource.getResource());
		// 3.把资源解析成properties文件
		Properties properties = factoryBean.getObject();
		// 4.返回spring提供的PropertySource对象
		return (name != null ? new PropertiesPropertySource(name, properties)
				: new PropertiesPropertySource(resource.getResource().getFilename(), properties));
	}
}
//

package config;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.datasource.DataSourceUtils;
import org.springframework.jdbc.datasource.DriverManagerDataSource;
import org.springframework.transaction.support.TransactionSynchronizationManager;

import javax.sql.DataSource;
import java.sql.Connection;

/**
 * 和jdbc操作相关的配置： JdbcTemplate创建 DataSource创建
 */
public class JdbcConfig {

	@Value("${jdbc.driver}")
	private String driver;

	@Value("${jdbc.url}")
	private String url;

	@Value("${jdbc.username}")
	private String username;

	@Value("${jdbc.password}")
	private String password;

	@Bean("jdbcTemplate1")
	public JdbcTemplate createJdbcTemplateOne(@Autowired DataSource dataSource) {
		return new JdbcTemplate(dataSource);
	}

	@Bean("jdbcTemplate2")
	public JdbcTemplate createJdbcTemplateTwo(@Autowired DataSource dataSource) {
		return new JdbcTemplate(dataSource);
	}

	@Bean("dataSource")
	public DataSource createDataSource() {
		// 1.创建数据源对象
		DriverManagerDataSource dataSource = new DriverManagerDataSource();
		// 2.给属性赋值
		dataSource.setDriverClassName(driver);
		dataSource.setUrl(url);
		dataSource.setUsername(username);
		dataSource.setPassword(password);
		// 3.返回
		return dataSource;
	}

	@Bean("connection")
	public Connection getConnection(DataSource dataSource) {
		// 1.初始化事务同步管理器
		TransactionSynchronizationManager.initSynchronization();
		// 2.使用spring的数据源工具类获取当前线程的连接
		Connection connection = DataSourceUtils.getConnection(dataSource);
		// 3.返回
		return connection;
	}
}

```

jdbc.yml

```yml
jdbc:
  driver: com.mysql.cj.jdbc.Driver
  url: jdbc:mysql://debug-registry:3306/mall
  username: root
  password: root
```



#### 4.6.2 日志


```java
/**
 * 系统日志的实体类
 */
@Data
public class SystemLog implements Serializable {

	/**
	 * 
	 */
	private static final long serialVersionUID = -1865325912289539485L;
	private String id; // 日志的主键
	private String method; // 当前执行的操作方法名称
	private String action; // 当前执行的操作方法说明
	private Date time; // 执行时间
	private String remoteIP;// 来访者IP

	@Override
	public String toString() {
		return "SystemLog{" + "id='" + id + '\'' + ", method='" + method + '\'' + ", action='" + action + '\''
				+ ", time=" + time + ", remoteIP='" + remoteIP + '\'' + '}';
	}
}
```

```java
package com.xuzhihao.utils;

import java.lang.reflect.Method;
import java.util.Date;
import java.util.UUID;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.Signature;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.context.annotation.Description;
import org.springframework.stereotype.Component;

import com.xuzhihao.domain.SystemLog;

/**
 * 记录日志的工具类
 * 
 */
@Component
@Aspect // 表明当前类是一个切面类
public class LogUtil {

	@Around("execution(* com.xuzhihao.service.impl.*.*(..))")
	public Object aroundPrintLog(ProceedingJoinPoint pjp) {
		// 定义返回值对象
		Object rtValue = null;
		try {
			// 创建系统日志对象
			SystemLog log = new SystemLog();
			// 设置主键
			String id = UUID.randomUUID().toString().replace("-", "").toUpperCase();
			log.setId(id);
			// 设置来访者ip，由于我们是java工程，没有请求信息
			log.setRemoteIP("127.0.0.1");
			// 设置执行时间
			log.setTime(new Date());
			// 设置当前执行的方法名称
			// 1.使用ProceedingJoinPoint接口中的获取签名方法
			Signature signature = pjp.getSignature();
			// 2.判断当前签名是否方法签名
			if (signature instanceof MethodSignature) {
				// 3.把签名转成方法签名
				MethodSignature methodSignature = (MethodSignature) signature;
				// 4.获取当前执行的方法
				Method method = methodSignature.getMethod();
				// 5.得到方法名称
				String methodName = method.getName();
				// 6.给系统日志中方法名称属性赋值
				log.setMethod(methodName);

				// 设置当前执行的方法说明
				// 7.判断当前方法上是否有@Description注解
				boolean isAnnotated = method.isAnnotationPresent(Description.class);
				if (isAnnotated) {
					// 8.得到当前方法上的Description注解
					Description description = method.getAnnotation(Description.class);
					// 9.得到注解的value属性
					String value = description.value();
					// 10.给系统日志的方法说明属性赋值
					log.setAction(value);
				}
			}

			System.out.println("环绕通知执行了记录日志：" + log);

			// 获取切入点方法的参数
			Object[] args = pjp.getArgs();
			// 切入点方法执行
			rtValue = pjp.proceed(args);
			return rtValue;
		} catch (Throwable e) {
			throw new RuntimeException(e);
		}
	}
}
```

#### 4.6.3 事务控制

```java
package com.xuzhihao.utils;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.sql.Connection;

/**
 * 事务控制
 */
@Component
public class TransactionManager {

	@Autowired
	private Connection connection;

	/**
	 * 开启事务
	 */
	public void begin() {
		try {
			connection.setAutoCommit(false);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	/**
	 * 提交事务
	 */
	public void commit() {
		try {
			connection.commit();
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	/**
	 * 回滚事务
	 */
	public void rollback() {
		try {
			connection.rollback();
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	/**
	 * 释放资源
	 */
	public void close() {
		try {
			connection.close();
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
```

代理类

```java
package com.xuzhihao.factory;

import com.xuzhihao.service.AccountService;
import com.xuzhihao.utils.TransactionManager;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.stereotype.Component;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

/**
 * 用于生产service代理对象的工厂 此处我们只做入门，实现对AccountService的代理创建，同时加入事务
 * 
 * @author admin
 */
@Component
public class ProxyServiceFactory {

	@Autowired
	private AccountService accountService;

	@Autowired
	private TransactionManager transactionManager;

	@Bean("proxyAccountService")
	public AccountService getProxyAccountService() {
		// 1.创建代理对象
		AccountService proxyAccountService = (AccountService) Proxy.newProxyInstance(
				accountService.getClass().getClassLoader(), accountService.getClass().getInterfaces(),
				new InvocationHandler() {
					@Override
					public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
						// 1.定义返回值
						Object rtValue = null;
						try {
							// 开始事务
							transactionManager.begin();
							// 执行被代理对象的方法
							rtValue = method.invoke(accountService, args);
							// 提交事务
							transactionManager.commit();
						} catch (Exception e) {
							// 回滚事务
							transactionManager.rollback();
							e.printStackTrace();
						} finally {
							// 释放资源
							transactionManager.close();
						}

						// 返回
						return rtValue;
					}
				});
		// 2.返回
		return proxyAccountService;
	}
}
```

Dao实现

```java
package com.xuzhihao.dao;

import java.util.List;

import com.xuzhihao.domain.Account;

/**
 * 账户的持久层接口
 */
public interface AccountDao {

	/**
	 * 保存账户
	 * 
	 * @param account
	 */
	void save(Account account);

	/**
	 * 更新账户
	 * 
	 * @param account
	 */
	void update(Account account);

	/**
	 * 删除
	 * 
	 * @param id
	 */
	void delete(Integer id);

	/**
	 * 根据id查询
	 * 
	 * @param id
	 * @return
	 */
	Account findById(Integer id);

	/**
	 * 查询所有
	 * 
	 * @return
	 */
	List<Account> findAll();

	/**
	 * 根据名称查询账户
	 * 
	 * @param name
	 * @return
	 */
	Account findByName(String name);
}

```

```java
package com.xuzhihao.dao.impl;

import com.xuzhihao.dao.AccountDao;
import com.xuzhihao.domain.Account;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.jdbc.core.BeanPropertyRowMapper;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

import java.util.List;

/**
 * 账户的持久层实现
 */
@Repository
public class AccountDaoImpl implements AccountDao {

	@Autowired(required = true)
	@Qualifier("jdbcTemplate1")
	private JdbcTemplate jdbcTemplate;

	@Override
	public void save(Account account) {
		jdbcTemplate.update("insert into account(name,money)values(?,?)", account.getName(), account.getMoney());
	}

	@Override
	public void update(Account account) {
		jdbcTemplate.update("update account set name=?,money=? where id =?", account.getName(), account.getMoney(),
				account.getId());
	}

	@Override
	public void delete(Integer id) {
		jdbcTemplate.update("delete from account where id = ? ", id);
	}

	@Override
	public Account findById(Integer id) {
		List<Account> accounts = jdbcTemplate.query("select * from account where id = ?",
				new BeanPropertyRowMapper<Account>(Account.class), id);
		return accounts.isEmpty() ? null : accounts.get(0);
	}

	@Override
	public List<Account> findAll() {
		return jdbcTemplate.query("select * from account ", new BeanPropertyRowMapper<Account>(Account.class));
	}

	@Override
	public Account findByName(String name) {
		List<Account> accounts = jdbcTemplate.query("select * from account where name = ?",
				new BeanPropertyRowMapper<Account>(Account.class), name);
		if (accounts.isEmpty()) {
			return null;
		}
		if (accounts.size() > 1) {
			throw new IllegalArgumentException("账户名不唯一");
		}
		return accounts.get(0);
	}
}
```

实体

```java
package com.xuzhihao.domain;

import java.io.Serializable;

import lombok.Data;

/**
 * 账户的实体类
 */
@Data
public class Account implements Serializable {

	/**
	 * 
	 */
	private static final long serialVersionUID = 7009401783939331569L;
	private Integer id;
	private String name;
	private Double money;

}

```

业务类

```java
package com.xuzhihao.service;

import java.util.List;

import com.xuzhihao.domain.Account;

/**
 * 账户的业务层接口
 */
public interface AccountService {

	/**
	 * 保存账户
	 * 
	 * @param account
	 */
	void save(Account account);

	/**
	 * 更新账户
	 * 
	 * @param account
	 */
	void update(Account account);

	/**
	 * 删除
	 * 
	 * @param id
	 */
	void delete(Integer id);

	/**
	 * 根据id查询
	 * 
	 * @param id
	 * @return
	 */
	Account findById(Integer id);

	/**
	 * 查询所有
	 * 
	 * @return
	 */
	List<Account> findAll();

	/**
	 * 转账
	 * 
	 * @param sourceName 转出账户名称
	 * @param targetName 转入账户名称
	 * @param money      转账金额
	 */
	void transfer(String sourceName, String targetName, Double money);
}

```

```java
package com.xuzhihao.service.impl;

import com.xuzhihao.dao.AccountDao;
import com.xuzhihao.domain.Account;
import com.xuzhihao.service.AccountService;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;

/**
 * 账户的业务层实现
 */
@Service("accountService")
public class AccountServiceImpl implements AccountService {

	@Autowired
	private AccountDao accountDao;

	@Override
	public void save(Account account) {
		accountDao.save(account);
	}

	@Override
	public void update(Account account) {
		accountDao.update(account);
	}

	@Override
	public void delete(Integer id) {
		accountDao.delete(id);
	}

	@Override
	public void transfer(String sourceName, String targetName, Double money) {
		// 1.根据名称查询转出账户
		Account source = accountDao.findByName(sourceName);
		// 2.根据名称查询转入账户
		Account target = accountDao.findByName(targetName);
		// 3.转出账户减钱
		source.setMoney(source.getMoney() - money);
		// 4.转入账户加钱
		target.setMoney(target.getMoney() + money);
		// 5.更新转出账户
		accountDao.update(source);
//		int i = 1 / 0;// 模拟转账异常
		// 6.更新转入账户
		accountDao.update(target);
	}

	@Override
	public Account findById(Integer id) {
		return accountDao.findById(id);
	}

	@Override
	public List<Account> findAll() {
		return accountDao.findAll();
	}
}

```

### 4.7 JDBC

实体

```java
@Data
public class Account implements Serializable {

	/**
	 * 
	 */
	private static final long serialVersionUID = 7009401783939331569L;
	private Integer id;
	private String name;
	private Double money;

}
```

jdbc模板

```java
package com.xuzhihao.jdbc;

import java.sql.Connection;
import java.sql.ParameterMetaData;
import java.sql.PreparedStatement;
import java.sql.ResultSet;

import javax.sql.DataSource;

import com.xuzhihao.jdbc.handler.ResultSetHandler;

/**
 * 自定义JdbcTemplate
 * 
 * @author admin
 */
public class JdbcTemplate {

	private DataSource dataSource;

	public void setDataSource(DataSource dataSource) {
		this.dataSource = dataSource;
	}

	public JdbcTemplate() {

	}

	public JdbcTemplate(DataSource dataSource) {
		setDataSource(dataSource);
	}

	/**
	 * 用于执行增删改方法的
	 * 
	 * @param sql    执行的SQL语句
	 * @param params 执行SQL所需的参数
	 * @return 影响数据库记录的行数
	 */
	public int update(String sql, Object... params) {
		// 1.验证数据源是否有值
		if (dataSource == null) {
			throw new NullPointerException("DataSource can not be null!");
		}
		// 2.定义jdbc操作的相关对象
		Connection conn = null;
		PreparedStatement pstm = null;
		int res = 0;
		try {
			// 3.获取连接
			conn = dataSource.getConnection();
			// 4.获取预处理对象
			pstm = conn.prepareStatement(sql);
			// 5.获取参数的元信息
			ParameterMetaData pmd = pstm.getParameterMetaData();
			// 6.获取sql域中参数的个数
			int parameterCount = pmd.getParameterCount();
			// 7.判断sql语句中是否有参数
			if (parameterCount > 0) {
				// 判断是否提供了参数
				if (params == null) {
					throw new IllegalArgumentException("Parameter can not be null!");
				}
				// 判断是否个数匹配
				if (parameterCount != params.length) {
					throw new IllegalArgumentException("Incorrect parameter size: expected "
							+ String.valueOf(parameterCount) + ", actual " + String.valueOf(params.length));
				}
				// 参数校验通过，给占位符赋值
				for (int i = 0; i < parameterCount; i++) {
					pstm.setObject(i + 1, params[i]);
				}
			}
			// 8.执行sql语句
			res = pstm.executeUpdate();
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			release(null, pstm, conn);
		}
		return res;
	}

	/**
	 * 执行查询方法
	 * 
	 * @param sql    执行的语句
	 * @param rsh    处理结果集的封装，此处只是提供一个规范（接口），由使用者编写具体的实现
	 * @param params 执行语句所需的参数
	 * @return
	 */
	public Object query(String sql, ResultSetHandler rsh, Object... params) {
		// 1.验证数据源是否有值
		if (dataSource == null) {
			throw new NullPointerException("DataSource can not be null!");
		}
		// 2.定义jdbc操作的相关对象
		Connection conn = null;
		PreparedStatement pstm = null;
		ResultSet rs = null;
		try {
			// 3.获取连接
			conn = dataSource.getConnection();
			// 4.获取预处理对象
			pstm = conn.prepareStatement(sql);
			// 5.获取参数的元信息
			ParameterMetaData pmd = pstm.getParameterMetaData();
			// 6.获取sql域中参数的个数
			int parameterCount = pmd.getParameterCount();
			// 7.判断sql语句中是否有参数
			if (parameterCount > 0) {
				// 判断是否提供了参数
				if (params == null) {
					throw new IllegalArgumentException("Parameter can not be null!");
				}
				// 判断是否个数匹配
				if (parameterCount != params.length) {
					throw new IllegalArgumentException("Incorrect parameter size: expected "
							+ String.valueOf(parameterCount) + ", actual " + String.valueOf(params.length));
				}
				// 参数校验通过，给占位符赋值
				for (int i = 0; i < parameterCount; i++) {
					pstm.setObject(i + 1, params[i]);
				}
			}
			// 8.执行sql语句
			rs = pstm.executeQuery();
			// 9.封装
			return rsh.handle(rs);
		} catch (Exception e) {
			throw new RuntimeException(e);
		} finally {
			release(rs, pstm, conn);
		}
	}

	private void release(ResultSet rs, PreparedStatement pstm, Connection conn) {
		if (rs != null) {
			try {
				rs.close();
			} catch (Exception e) {
				e.printStackTrace();
			}

		}

		if (pstm != null) {
			try {
				pstm.close();
			} catch (Exception e) {
				e.printStackTrace();
			}

		}

		if (conn != null) {
			try {
				conn.close();
			} catch (Exception e) {
				e.printStackTrace();
			}

		}
	}
}
```

配置

```java
package config;

import javax.sql.DataSource;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;

import com.alibaba.druid.pool.DruidDataSource;
import com.xuzhihao.jdbc.JdbcTemplate;

/**
 * jdbc
 */
public class JdbcConfig {

	@Value("${jdbc.driver}")
	private String driver;
	@Value("${jdbc.url}")
	private String url;
	@Value("${jdbc.username}")
	private String username;
	@Value("${jdbc.password}")
	private String password;

	/**
	 * 创建数据源并存入Ioc容器
	 * 
	 * @return
	 */
	@Bean
	public DataSource createDataSource() {
		DruidDataSource dataSource = new DruidDataSource();
		dataSource.setDriverClassName(driver);
		dataSource.setUrl(url);
		dataSource.setUsername(username);
		dataSource.setPassword(password);
		return dataSource;
	}

	/**
	 * 创建JdbcTemplate对象
	 * 
	 * @param dataSource
	 * @return
	 */
	@Bean
	public JdbcTemplate createJdbcTemplate(DataSource dataSource) {
		return new JdbcTemplate(dataSource);
	}

}

```

```java
package config;

import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Import;
import org.springframework.context.annotation.PropertySource;

/**
 * spring的配置类，代替applicationContext.xml
 */
@Configuration
@Import(JdbcConfig.class)
@PropertySource("classpath:jdbc.properties")
public class SpringConfiguration {
}

```

单体封装

```java
package com.xuzhihao.jdbc.handler;

import java.beans.PropertyDescriptor;
import java.lang.reflect.Method;
import java.sql.ResultSet;
import java.sql.ResultSetMetaData;

/**
 * 封装到实体类中的结果集处理器
 * 
 * @author admin
 */
public class BeanHandler<T> implements ResultSetHandler {

	// 定义封装哪个实体类的子接口
	private Class<T> requiredType;

	/**
	 * 当创建BeanHandler对象时，就需要提供封装到的实体类字节码
	 * 
	 * @param requiredType
	 */
	public BeanHandler(Class<T> requiredType) {
		this.requiredType = requiredType;
	}

	@Override
	public Object handle(ResultSet rs) {
		// 1.定义返回值
		T bean = null;
		try {
			// 2.判断是否有结果集
			if (rs.next()) {
				// 3.实例化返回值对象
				bean = requiredType.newInstance();
				// 4.获取结果集的元信息
				ResultSetMetaData rsmd = rs.getMetaData();
				// 5.获取结果集的列数
				int columnCount = rsmd.getColumnCount();
				// 6.遍历列的个数
				for (int i = 0; i < columnCount; i++) {
					// 7.取出列的标题
					String columnLabel = rsmd.getColumnLabel(i + 1);
					// 8.取出当前列标题对应的数据内容
					Object value = rs.getObject(columnLabel);
					// 9.借助java的内省机制，使用属性描述器填充
					PropertyDescriptor pd = new PropertyDescriptor(columnLabel, requiredType);
					// 10.获取属性的写方法
					Method method = pd.getWriteMethod();
					// 11.执行方法
					method.invoke(bean, value);
				}
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
		return bean;
	}
}

```

list封装

```java
package com.xuzhihao.jdbc.handler;

import java.beans.PropertyDescriptor;
import java.lang.reflect.Method;
import java.sql.ResultSet;
import java.sql.ResultSetMetaData;
import java.util.ArrayList;

/**
 * 默认结果集处理
 * 
 * @author admin
 */
public class BeanListHandler<T> implements ResultSetHandler {

	// 定义封装哪个实体类的子接口
	private Class<T> requiredType;

	/**
	 * 当创建BeanHandler对象时，就需要提供封装到的实体类字节码
	 * 
	 * @param requiredType
	 */
	public BeanListHandler(Class<T> requiredType) {
		this.requiredType = requiredType;
	}

	@Override
	public Object handle(ResultSet rs) {
		// 1.定义返回值
		ArrayList<Object> list = new ArrayList<>();
		T bean = null;
		try {
			// 2.遍历结果集
			while (rs.next()) {
				// 3.实例化返回值对象
				bean = requiredType.newInstance();
				// 4.获取结果集的元信息
				ResultSetMetaData rsmd = rs.getMetaData();
				// 5.获取结果集的列数
				int columnCount = rsmd.getColumnCount();
				// 6.遍历列的个数
				for (int i = 0; i < columnCount; i++) {
					// 7.取出列的标题
					String columnLabel = rsmd.getColumnLabel(i + 1);
					// 8.取出当前列标题对应的数据内容
					Object value = rs.getObject(columnLabel);
					// 9.借助java的内省机制，使用属性描述器填充
					PropertyDescriptor pd = new PropertyDescriptor(columnLabel, requiredType);
					// 10.获取属性的写方法
					Method method = pd.getWriteMethod();
					// 11.执行方法
					method.invoke(bean, value);
				}
				// 12.把填充好的bean封装到集合中
				list.add(bean);
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
		return list;
	}
}

```

接口封装

```java
package com.xuzhihao.jdbc.handler;

import java.sql.ResultSet;

/**
 * 结果集的处理器
 * 
 * @author admin
 */
public interface ResultSetHandler {

	/**
	 * 处理结果集
	 * 
	 * @param rs
	 * @return
	 */
	Object handle(ResultSet rs);
}

```

```xml
jdbc.driver=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql://debug-registry:3306/mall
jdbc.username=root
jdbc.password=root
```

测试

```java
package com.xuzhihao.test;

import java.util.List;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

import com.xuzhihao.domain.Account;
import com.xuzhihao.jdbc.JdbcTemplate;
import com.xuzhihao.jdbc.handler.BeanHandler;
import com.xuzhihao.jdbc.handler.BeanListHandler;

import config.SpringConfiguration;

/**
 * 
 */
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(classes = SpringConfiguration.class)
public class JdbcTemplateTest {

	@Autowired
	private JdbcTemplate jdbcTemplate;

	@Test
	public void testJdbcTemplate() {
		System.out.println(jdbcTemplate);
	}

	@Test
	public void testSave() {
		jdbcTemplate.update("insert into account(name,money)values(?,?)", "MyJdbcTemplate", 6789d);
	}

	@Test
	public void testUpdate() {
		jdbcTemplate.update("update account set name=?,money=? where id=?", "MyJdbcTemplate", 23456d, 5);
	}

	@Test
	public void testDelete() {
		jdbcTemplate.update("delete from account where id = ? ", 5);
	}

	@Test
	public void testFindOne() {
		Account account = (Account) jdbcTemplate.query("select * from account where id = ?",
				new BeanHandler<Account>(Account.class), 1);
		System.out.println(account);
	}

	@SuppressWarnings("unchecked")
	@Test
	public void testFindAll() {
		List<Account> accountList = (List<Account>) jdbcTemplate.query("select * from account where money > ?",
				new BeanListHandler<Account>(Account.class), 999d);
		for (Account account : accountList) {
			System.out.println(account);
		}
	}
}

```

### 4.8 其他

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>com.xuzhihao</groupId>
	<artifactId>spring-base</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>pom</packaging>
	<modules>
		<module>spring01-configuration</module>
		<module>spring02-componentscan</module>
		<module>spring03-bean</module>
		<module>spring04-import</module>
		<module>spring05-propertysource</module>
		<module>spring06-conditional</module>
		<module>spring07-postprocessor</module>
		<module>spring08-jdbc</module>
		<module>spring09-myjdbc</module>
		<module>spring10-aop</module>
	</modules>
	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<java.version>1.8</java.version>
		<spring.version>5.3.1</spring.version>
		<hutool.version>4.5.7</hutool.version>
		<lombok.version>1.18.12</lombok.version>
		<mysql-connector.version>8.0.16</mysql-connector.version>
		<snakeyaml.version>1.27</snakeyaml.version>
		<druid.version>1.1.10</druid.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-context</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-jdbc</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-test</artifactId>
		</dependency>
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
		</dependency>
		<dependency>
			<groupId>cn.hutool</groupId>
			<artifactId>hutool-all</artifactId>
		</dependency>
		<dependency>
			<groupId>org.projectlombok</groupId>
			<artifactId>lombok</artifactId>
		</dependency>
		<dependency>
			<groupId>org.yaml</groupId>
			<artifactId>snakeyaml</artifactId>
		</dependency>
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>druid</artifactId>
		</dependency>
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>4.12</version>
		</dependency>
	</dependencies>
	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework</groupId>
				<artifactId>spring-framework-bom</artifactId>
				<version>${spring.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
			<!--Hutool Java工具包 -->
			<dependency>
				<groupId>cn.hutool</groupId>
				<artifactId>hutool-all</artifactId>
				<version>${hutool.version}</version>
			</dependency>
			<!--lombok 插件 -->
			<dependency>
				<groupId>org.projectlombok</groupId>
				<artifactId>lombok</artifactId>
				<version>${lombok.version}</version>
			</dependency>
			<!--Mysql数据库驱动 -->
			<dependency>
				<groupId>mysql</groupId>
				<artifactId>mysql-connector-java</artifactId>
				<version>${mysql-connector.version}</version>
			</dependency>
			<!-- yaml解析 -->
			<dependency>
				<groupId>org.yaml</groupId>
				<artifactId>snakeyaml</artifactId>
				<version>${snakeyaml.version}</version>
			</dependency>
			<dependency>
				<groupId>com.alibaba</groupId>
				<artifactId>druid</artifactId>
				<version>${druid.version}</version>
			</dependency>
		</dependencies>
	</dependencyManagement>
	<build>
		<pluginManagement>
			<plugins>
				<plugin>
					<groupId>org.apache.maven.plugins</groupId>
					<artifactId>maven-compiler-plugin</artifactId>
					<version>2.3.2</version>
					<configuration>
						<source>1.8</source>
						<target>1.8</target>
						<encoding>UTF-8</encoding>
					</configuration>
				</plugin>
			</plugins>
		</pluginManagement>
	</build>
</project>
```