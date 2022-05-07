# Spring Cloud Alibaba

## 1. Sentinel分布式系统流量防卫兵

```bash
java -Dserver.port=8858 -Dcsp.sentinel.dashboard.server=localhost:8858 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard-1.7.0.jar
```

```yml
spring:
   cloud:
      inetutils:
         preferred-networks: 192.168.3
      nacos:
         discovery:
            server-addr: xuzhihao:8848
         config:
            server-addr: xuzhihao:8848
            file-extension: yaml
      sentinel:
         eager: true
         transport:
            dashboard: xuzhihao:8858
            clientIp: 192.168.3.100
         datasource:
            ds1:
               nacos:
                  server-addr: xuzhihao:8848
                  dataId: ${spring.application.name}-sentinel
                  groupId: DEFAULT_GROUP
                  data-type: json
                  rule-type: flow
```

### 1.1 @SentinelResource注解参数

通过@SentinelResource来指定出现异常时的处理策略。用于定义资源，并提供可选的异常处理和 fallback 配置项

| 文件夹| 说明 | 
| ----- | ----- | 
| value | 资源名称 |
| entryType | entry类型，标记流量的方向，取值IN/OUT，默认是OUT |
| blockHandler | 处理BlockException的函数名称,函数要求：<br>1. 必须是 public<br>2. 返回类型 参数与原方法一致<br>3. 默认需和原方法在同一个类中。若希望使用其他类的函数，可配置blockHandlerClass ，并指定lockHandlerClass里面的方法。 |
| blockHandlerClass | 存放blockHandler的类,对应的处理函数必须static修饰。 | 
| fallback | 用于在抛出异常的时候提供fallback处理逻辑。fallback函数可以针对所有类型的异常（除了 exceptionsToIgnore 里面排除掉的异常类型）进行处理。函数要求：<br>1. 返回类型与原方法一致<br>2. 参数类型需要和原方法相匹配<br>3. 默认需和原方法在同一个类中。若希望使用其他类的函数，可配置fallbackClass ，并指定fallbackClass里面的方法。 | 
| fallbackClass | 存放fallback的类。对应的处理函数必须static修饰。 | 
| defaultFallback | 用于通用的 fallback 逻辑。默认fallback函数可以针对所有类型的异常进行处理。若同时配置了 fallback 和 defaultFallback，以fallback为准。函数要求：<br>1. 返回类型与原方法一致<br>2. 方法参数列表为空，或者有一个 Throwable 类型的参数。<br>3. 默认需要和原方法在同一个类中。若希望使用其他类的函数，可配置fallbackClass ，并指定 fallbackClass 里面的方法。 | 
| exceptionsToIgnore | 指定排除掉哪些异常。排除的异常不会计入异常统计，也不会进入fallback逻辑，而是原样抛出。 | 
| exceptionsToTrace | 需要trace的异常 | 

本地资源加载配置文件见`RuleConstant.java`

### 1.2 自定义降级限流

```java
package com.xuzhihao.sentinel.fallback;

import com.alibaba.csp.sentinel.slots.block.BlockException;

//对应的BlockException处理的类
/**
 * 测试超出流量限制的部分是否会进入到blockHandler的方法。
 */
public class CustomBlockHandler {
	public static String blockHandler(String id, BlockException e) {
		return id+"自定义限流信息";
	}
}
```

```java
package com.xuzhihao.sentinel.fallback;

//对应的Throwable处理的类
/**
 * sentinel定义的失败调用或限制调用，若本次访问被限流或服务降级，则调用blockHandler指定的接口。
 */
public class CustomFallback {

	// fallback
	// 要求:
	// 1 当前方法的返回值和参数要跟原方法一致
	// 2 但是允许在参数列表的最后加入一个参数BlockException, 用来接收原方法中发生的异常
	public static String fallback( String id, Throwable e) {
		return id+"出现异常呗服务降级";
	}
}
```

```java
package com.xuzhihao.sentinel.service.impl;

import org.springframework.stereotype.Component;

import com.alibaba.csp.sentinel.annotation.SentinelResource;
import com.xuzhihao.sentinel.fallback.CustomBlockHandler;
import com.xuzhihao.sentinel.fallback.CustomFallback;
import com.xuzhihao.sentinel.service.UserService;

@Component
public class UserServiceImpl implements UserService {

	int i = 0;

	@SentinelResource(value = "create", 
			blockHandlerClass = CustomBlockHandler.class, 
			blockHandler = "blockHandler", 
			fallbackClass = CustomFallback.class, 
			fallback = "fallback")
	@Override
	public String create(String id) {
		i++;
        if (i % 3 == 0) {
            throw new RuntimeException();
        }
		return id;
	}

}
```


### 1.3 @Feign整合

fallback方式

```java
@FeignClient(
value = "service-product",fallback = ProductServiceFallBack.class)
public interface ProductService {
	@RequestMapping("/product/{pid}")
	Product findByPid(@PathVariable Integer pid);
}
```

```java
//当feign调用出现问题的时候,就会进入到当前类中同名方法中
@Service
public class ProductServiceFallback implements ProductService {
    @Override
    public Product findByPid(Integer pid) {
        Product product = new Product();
        product.setPid(-100);
        product.setPname("商品微服务调用出现异常了,已经进入到了容错方法中");
        return product;
    }
}
```

fallbackFactory方式
```java
@FeignClient(
value = "service-product",fallbackFactory = ProductServiceFallbackFactory.class)
public interface ProductService {
	@RequestMapping("/product/{pid}")
	Product findByPid(@PathVariable Integer pid);
}
```
```java

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import feign.hystrix.FallbackFactory;

@Component
public class ProductServiceFallbackFactory implements FallbackFactory<ProductService> {
	
  	private static final Logger LOGGER = LoggerFactory.getLogger(HystrixClientFactory.class);
	
    //Throwable  这就是fegin在调用过程中产生异常
    @Override
    public ProductService create(Throwable throwable) {
		LOGGER.info("fallback; reason was: {}", cause.getMessage());
        return new ProductService() {
            @Override
            public Product findByPid(Integer pid) {
                log.error("{}",throwable);
                Product product = new Product();
                product.setPid(-100);
                product.setPname("商品微服务调用出现异常了,已经进入到了容错方法中");
                return product;
            }
        };
    }
}
```

`fallbackFactory 是fallback的一个升级版，可以打印回退异常的原因`

### 1.4 原理分析

首先看SentinelFeignAutoConfiguration中自动配置

```java
/**
 * @author <a href="mailto:fangjian0423@gmail.com">Jim</a>
 */
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ SphU.class, Feign.class })
public class SentinelFeignAutoConfiguration {
	@Bean
	@Scope("prototype")
	@ConditionalOnMissingBean
	@ConditionalOnProperty(name = "feign.sentinel.enabled")
	public Feign.Builder feignSentinelBuilder() {
		return SentinelFeign.builder();
	}
}
```

@ConditionalOnProperty 中 feign.sentinel.enabled 起了决定性作用，这也就是为什么我们需要在配置文件中指定 `feign.sentinel.enabled=true`

接下来看 SentinelFeign.builder 里面的实现：

build方法中重新实现了super.invocationHandlerFactory方法，也就是动态代理工厂，构建的是InvocationHandler对象。

build中会获取Feign Client中的信息，比如fallback,fallbackFactory等，然后创建一个SentinelInvocationHandler，SentinelInvocationHandler继承了InvocationHandler

```java
@Override
	public Feign build() {
		super.invocationHandlerFactory(new InvocationHandlerFactory() {
			@Override
			public InvocationHandler create(Target target,
					Map<Method, MethodHandler> dispatch) {
				// 得到Feign Client Bean
				Object feignClientFactoryBean = Builder.this.applicationContext
						.getBean("&" + target.type().getName());
				// 得到fallback类
				Class fallback = (Class) getFieldValue(feignClientFactoryBean,
						"fallback");
				// 得到fallbackFactory类
				Class fallbackFactory = (Class) getFieldValue(feignClientFactoryBean,
						"fallbackFactory");
				// 得到调用的服务名称
				String beanName = (String) getFieldValue(feignClientFactoryBean,
						"contextId");
				if (!StringUtils.hasText(beanName)) {
					beanName = (String) getFieldValue(feignClientFactoryBean, "name");
				}

				Object fallbackInstance;
				FallbackFactory fallbackFactoryInstance;
				// 检查 fallback 和 fallbackFactory 属性
				if (void.class != fallback) {
					fallbackInstance = getFromContext(beanName, "fallback", fallback,
							target.type());
					return new SentinelInvocationHandler(target, dispatch,
							new FallbackFactory.Default(fallbackInstance));
				}
				if (void.class != fallbackFactory) {
					fallbackFactoryInstance = (FallbackFactory) getFromContext(
							beanName, "fallbackFactory", fallbackFactory,
							FallbackFactory.class);
					return new SentinelInvocationHandler(target, dispatch,
							fallbackFactoryInstance);
				}
				return new SentinelInvocationHandler(target, dispatch);
			}

			...
		});

		super.contract(new SentinelContractHolder(contract));
		return super.build();
	}
```

SentinelInvocationHandler中的invoke方法里面进行熔断限流的处理。

```java
// 得到资源名称（GET:http://user-service/user/get）
String resourceName = methodMetadata.template().method().toUpperCase()
				+ ":" + hardCodedTarget.url() + methodMetadata.template().path();
		Entry entry = null;
		try {
			ContextUtil.enter(resourceName);
			entry = SphU.entry(resourceName, EntryType.OUT, 1, args);
			result = methodHandler.invoke(args);
		}
		catch (Throwable ex) {
			// fallback handle
			if (!BlockException.isBlockException(ex)) {
				Tracer.trace(ex);
			}
			if (fallbackFactory != null) {
				try {
					// 回退处理
					Object fallbackResult = fallbackMethodMap.get(method)
							.invoke(fallbackFactory.create(ex), args);
					return fallbackResult;
				}
				catch (IllegalAccessException e) {
					throw new AssertionError(e);
				}
				catch (InvocationTargetException e) {
					throw new AssertionError(e.getCause());
				}
			}
			else {
				// throw exception if fallbackFactory is null
				throw ex;
			}
		}
```

总结

RestTemplate的整合其实和Ribbon中的@LoadBalanced原理差不多，这次的Feign的整合其实我们从其他框架的整合也是可以参考出来的，最典型的就是Hystrix了

InvocationHandlerFactory是用于创建动态代理的工厂，有默认的实现，也有Hystrix的实现feign.hystrix.HystrixFeign。

```java
Feign build(final FallbackFactory<?> nullableFallbackFactory) {
      super.invocationHandlerFactory(new InvocationHandlerFactory() {
        @Override public InvocationHandler create(Target target,
            Map<Method, MethodHandler> dispatch) {
          return new HystrixInvocationHandler(target, dispatch, setterFactory, nullableFallbackFactory);
        }
      });
      super.contract(new HystrixDelegatingContract(contract));
      return super.build();
}
```
上面这段代码跟Sentinel包装的类似，不同的是Sentinel构造的是SentinelInvocationHandler ，Hystrix构造的是HystrixInvocationHandle。在HystrixInvocationHandler的invoke方法中进行HystrixCommand的包装。



## 2. Nacos

### 2.1 配置管理

#### 2.1.1 安装配置

下载地址：https://github.com/alibaba/nacos/releases

```bash
sh startup.sh -m standalone
```

数据库配置
```
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://172.17.17.137:3305/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user.0=root
db.password.0=root
```

下载curl的windows版本：curl-7.66.0_2-win64-mingw，下载地址：https://curl.haxx.se/windows/ 

进入curl-7.66.0_2-win64-mingw的bin目录，进行下边的测试，通过测试可判断nacos是否正常工作：

```bash
curl -X POST "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test&content=HelloWorld" # 发布配置
curl -X GET "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=nacos.cfg.dataId&group=test"					   # 获取配置
```

#### 2.1.2 客户端获取

```xml
<dependency>
	<groupId>com.alibaba.nacos</groupId>
	<artifactId>nacos‐client</artifactId>
	<version>1.1.3</version>
</dependency>
```

```java
package com.itheima.nacos;

import java.util.Properties;
import java.util.concurrent.Executor;

import com.alibaba.nacos.api.NacosFactory;
import com.alibaba.nacos.api.config.ConfigService;
import com.alibaba.nacos.api.config.listener.Listener;
import com.alibaba.nacos.api.exception.NacosException;

/**
 * @author Administrator
 * @version 1.0
 **/
public class SimpleDemoMain {
    public static void main(String[] args) throws NacosException {
        // 使用nacos client远程获取nacos服务上的配置信息
        // nacos server地址
        String serverAddr = "127.0.0.1:8848";
        // data id
        String dataId = "nacos-simple-demo.yaml";
        // group
        String group = "DEFAULT_GROUP";

        // String namespace = "c67e4a97-a698-4d6d-9bb1-cfac5f5b51c4";
        Properties properties = new Properties();
        properties.put("serverAddr", serverAddr);
        // properties.put("namespace", namespace);
        // 获取配置
        ConfigService configService = NacosFactory.createConfigService(properties);
        String config = configService.getConfig(dataId, group, 5000);
        System.out.println(config);
        configService.addListener(dataId, group, new Listener() {
            public Executor getExecutor() {
                return null;
            }

            // 当配置有变化 时候获取通知
            public void receiveConfigInfo(String s) {
                System.out.println(s);
            }
        });

        while (true) {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

    }
}
```

#### 2.1.3 登录管理

```xml
<dependency>
	<groupId>org.springframework.security</groupId>
	<artifactId>spring‐security‐core</artifactId>
	<version>5.1.4.RELEASE</version>
</dependency>
```

```java
new BCryptPasswordEncoder().encode("123")
```

```sql
INSERT INTO users (username, password, enabled) VALUES ('nacos1','$2a$10$SmtL5C6Gp2sLjBrhrx1vj.dJAbJLa4FiJYZsBb921/wfvKAmxKWyu', TRUE);
INSERT INTO roles (username, role) VALUES ('nacos1', 'ROLE_ADMIN');
```

关闭认证修改配置application.properties
```
spring.security.enabled=false
```

#### 2.1.4 动态刷新

单文件

```
server:
  port: 8080 #启动端口命令行注入

spring:
  application:
    name: admin
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848 # 配置中心地址
        file-extension: yaml #dataid 的名称就是application的name加file-extension   admin.yaml
        namespace: c67e4a97-a698-4d6d-9bb1-cfac5f5b51c4 # 开发环境
        group: TEST_GROUP # 测试组
```


自定义扩展 Data Id 配置

```yaml
server:
  port: 8000 #启动端口 命令行注入

spring:
  application:
    name: admin
  cloud:
    nacos:
      config:
        #enabled: false #关闭配置
        server-addr: 127.0.0.1:8848,127.0.0.1:8849,127.0.0.1:8850 # 配置中心地址
        file-extension: yaml #dataid 的名称就是application的name加file-extension   service1.yaml
        namespace: c67e4a97-a698-4d6d-9bb1-cfac5f5b51c4 # 开发环境  指定 具体的namespace
        group: TEST_GROUP # 测试组
        ext-config[0]:
          data-id: ext-config-common01.properties
        ext-config[1]:
          data-id: ext-config-common02.properties
          group: GLOBALE_GROUP
        ext-config[2]:    # 数字大 优先级高 
          data-id: ext-config-common03.properties
          group: REFRESH_GROUP
          refresh: true  #动态刷新配置
```

自定义共享 Data Id 配置
```yaml
  cloud:
    nacos:
      config:
        shared-dataids: ext-config-common01.properties,ext-config-common02.properties,ext-config-common03.properties
        refreshable-dataids: ext-config-common01.properties
```

#### 2.1.5 集群配置

nacos/的conf目录下cluster.conf

```xml
192.168.16.101:8847
192.168.16.102:8848
192.168.16.103:8849
```

application.properties配置主备

```
spring.datasource.platform=mysql

db.num=2
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.url.1=jdbc:mysql://127.0.0.1:3306/nacos1?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user=root
db.password=root
```

两种配置：

1. 将所有节点添加到配置文件中
```yml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: http://nacos1:8848,http://nacos2:8848,http://nacos3:8848
      config:
        server-addr: http://debug-registry:8848
        file-extension: yaml
```

2. 将三个节点映射出一套VIP地址，配置文件中写一个即可


### 2.2 服务发现




## 3. RocketMQ

[消息队列/RocketMQ](/share/distributed/mq?id=rocketmq)

## 4. Dubbo：Apache Dubbo

[Dubbo源码解析](/share/distributed/dubbo)

## 5. Seata微服务分布式事务解决方案

[分布式事务Seata](/share/distributed/seata)
  
## 6. Alibaba Cloud ACM

## 7. Alibaba Cloud OSS

## 8. Alibaba Cloud SchedulerX

## 9. Alibaba Cloud SMS

```java
package cn.com.rocketmq.utils;

import com.aliyuncs.DefaultAcsClient;
import com.aliyuncs.IAcsClient;
import com.aliyuncs.dysmsapi.model.v20170525.SendSmsRequest;
import com.aliyuncs.dysmsapi.model.v20170525.SendSmsResponse;
import com.aliyuncs.profile.DefaultProfile;
import com.aliyuncs.profile.IClientProfile;
import lombok.extern.slf4j.Slf4j;

@Slf4j
public class SmsUtil {
	// 替换成自己申请的accessKeyId
	private static String accessKeyId = "xxx";
	// 替换成自己申请的accessKeySecret
	private static String accessKeySecret = "xxx";

	static final String product = "Dysmsapi";
	static final String domain = "dysmsapi.aliyuncs.com";

	/**
	 * 发送短信
	 *
	 * @param phoneNumbers 要发送短信到哪个手机号
	 * @param signName     短信签名[必须使用前面申请的]
	 * @param templateCode 短信短信模板ID[必须使用前面申请的]
	 * @param param        模板中${code}位置传递的内容
	 */
	public static void sendSms(String phoneNumbers, String signName, String templateCode, String param) {
		try {
			System.setProperty("sun.net.client.defaultConnectTimeout", "10000");
			System.setProperty("sun.net.client.defaultReadTimeout", "10000");

			// 初始化acsClient,暂不支持region化
			IClientProfile profile = DefaultProfile.getProfile("cn-hangzhou", accessKeyId, accessKeySecret);
			DefaultProfile.addEndpoint("cn-hangzhou", "cn-hangzhou", product, domain);
			IAcsClient acsClient = new DefaultAcsClient(profile);

			SendSmsRequest request = new SendSmsRequest();
			request.setPhoneNumbers(phoneNumbers);
			request.setSignName(signName);
			request.setTemplateCode(templateCode);
			request.setTemplateParam(param);
			request.setOutId("yourOutId");
			SendSmsResponse sendSmsResponse = acsClient.getAcsResponse(request);
			if (!"OK".equals(sendSmsResponse.getCode())) {
				log.info("发送短信失败,{}", sendSmsResponse);
				throw new RuntimeException(sendSmsResponse.getMessage());
			}
		} catch (Exception e) {
			log.info("发送短信失败,{}", e);
			throw new RuntimeException("发送短信失败");
		}
	}
}
```
