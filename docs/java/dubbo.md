# Dubbo源码解析

![](../../assets/_images/java//distributed/dubbo/dubbo-framework.jpg)

优点：

1. 面向接口代理的高性能RPC调用
2. 智能负载均衡
3. 注册发现
4. 高扩展
5. 配置路由规则实现灰度发布
6. 可视化管理

原理：

## 1. 启动解析加载配置信息
  
Dubbo 服务框架的 Schema 的解析通过 DubboNamespaceHandler 和 DubboBeanDefinitionParser 实现。其中，DubboNamespaceHandler 扩展了 Spring 的 `NamespaceHandlerSupport`，通过重写它的 init() 方法给各个标签注册对应的解析器

BeanDefinitionParser->DubboBeanDefinitionParser 

```java
  public class DubboNamespaceHandler extends NamespaceHandlerSupport {
    static {
        Version.checkDuplicate(DubboNamespaceHandler.class);
    }
    public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new DubboBeanDefinitionParser(AnnotationBean.class, true));
    }
}
```

而 DubboBeanDefinitionParser 实现了 Spring 的 BeanDefinitionParser，通过重写 parse() 方法实现将标签解析为对应的 JavaBean

```java
public class DubboBeanDefinitionParser implements BeanDefinitionParser {
    public BeanDefinition parse(Element element, ParserContext parserContext) {
    return parse(element, parserContext, beanClass, required);
    }

    @SuppressWarnings("unchecked")
    private static BeanDefinition parse(Element element,ParserContext parserContext,Class<?> beanClass,boolean required) {
        RootBeanDefinition beanDefinition = new RootBeanDefinition();
        beanDefinition.setBeanClass(beanClass);
        beanDefinition.setLazyInit(false);
        //......省略
        if (ProtocolConfig.class.equals(beanClass)) {
            for (String name : parserContext.getRegistry().getBeanDefinitionNames()) {
                BeanDefinition definition = parserContext.getRegistry().getBeanDefinition(name);
                PropertyValue property = definition.getPropertyValues().getPropertyValue("protocol");
                if (property != null) {
                    Object value = property.getValue();
                    if (value instanceof ProtocolConfig && id.equals(((ProtocolConfig) value).getName())) {
                        definition.getPropertyValues().addPropertyValue("protocol", new RuntimeBeanReference(id));
                    }
                }
            }
        } else if (ServiceBean.class.equals(beanClass)) {
            String className = element.getAttribute("class");
            if(className != null && className.length() > 0) {
                RootBeanDefinition classDefinition = new RootBeanDefinition();
                classDefinition.setBeanClass(ReflectUtils.forName(className));
                classDefinition.setLazyInit(false);
                parseProperties(element.getChildNodes(), classDefinition);
                beanDefinition.getPropertyValues().addPropertyValue("ref", new BeanDefinitionHolder(classDefinition, id + "Impl"));
            }
        } else if (ProviderConfig.class.equals(beanClass)) {
            parseNested(element, parserContext, ServiceBean.class, true, "service", "provider", id, beanDefinition);
        } else if (ConsumerConfig.class.equals(beanClass)) {
            parseNested(element, parserContext, ReferenceBean.class, false, "reference", "consumer", id, beanDefinition);
        }
        //......省略
        return beanDefinition;
    }
}
```
  
## 2. 服务暴露

![](../../assets/_images/java//distributed/dubbo/dubbo-export.jpg)

ServiceBean继承自ServiceConfig，ServiceConfig是服务暴露的具体实现类。另外ServiceBean还实现了InitializingBean，DisposableBean，ApplicationContextAware，ApplicationListener，BeanNameAware，
ApplicationEventPublisherAware几个接口，简单介绍下几个接口的作用：

- 继承InitializingBean实现其afterPropertiesSet方法，对象实例化完毕后，调用该方法。
- 继承ApplicationListener接口，监听上下文监听事件contextrefreshedevent事件，当Spring容器启动完成后会发布该事件，监听到之后会执行onApplicationEvent()方法。
- 继承DisposableBean实现其destory()方法，在实例生命周期结束前，会调用该方法。
- 继承BeanNameWare，用于设置Bean的名称
- 继承ApplicationContextAware，用于获取Spring容器的上下文
- 继承ApplicationEventPublisherAware接口，用于发布事件。

传送器选择 Transporter.class->getTransporter().bind(url,handler)

服务的暴露的入口位置主要在RegistryProtocol.export()方法中，该方法首先会进行服务的暴露，然后会进行服务的注册。如下是该方法的源码,主要完成了三部分的工作：
- 将服务与本地的某个端口号进行绑定，从而实现服务暴露的功能；
- 根据配置得到一个服务注册对象Registry，然后对其进行注册；
- 创建一个配置被重写的监听器，并且注册该监听器，从而实现配置被重写时能够动态的使用新的配置进行服务的配置。

```java
@Override
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
  // 获取服务注册相关的配置数据
  URL registryUrl = getRegistryUrl(originInvoker);
  // 获取provider相关的配置数据
  URL providerUrl = getProviderUrl(originInvoker);

  // 对provider的部分配置信息进行覆盖，重写的工作主要是委托给Configurator进行，
  // 这里OverrideListener的作用主要是在当前服务的配置信息发生更改时，对原有的配置进行重写，
  // 并且会判断是否需要对当前的服务进行重新暴露
  final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
  final OverrideListener overrideSubscribeListener = 
    new OverrideListener(overrideSubscribeUrl, originInvoker);
  overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
  providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);
  
  // 进行服务的本地暴露，本质上就是根据配置使用Netty绑定本地的某个端口，从而完成服务暴露工作
  final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);

  // 根据配置获取对应的Registry对象，常见的有ZookeeperRegistry和RedisRegistry，默认使用的是
  // ZookeeperRegistry，本文则以Zookeeper为例进行讲解
  final Registry registry = getRegistry(originInvoker);
  final URL registeredProviderUrl = getRegisteredProviderUrl(providerUrl, registryUrl);
  // 将当前的Invoker对象注册到一个全局的providerInvokers中进行缓存，
  // 该Map对象保存了所有的已经暴露了的服务
  ProviderInvokerWrapper<T> providerInvokerWrapper = 
    ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, 
      registeredProviderUrl);

  // 除非主动配置不进行注册，那么这里将会返回true
  boolean register = registeredProviderUrl.getParameter("register", true);
  if (register) {
    // 进行服务注册的代码，主要是通过Zookeeper的客户端CuratorFramework进行服务的注册
    register(registryUrl, registeredProviderUrl);
    // 将当前Invoker标识为已经注册完成
    providerInvokerWrapper.setReg(true);
  }

  // 注册配置被更改的监听事件，将配置被更改时将会触发相应的listener
  registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

  // 设置相关的URL对象，并且使用DestroyableExporter对exporter进行封装返回
  exporter.setRegisterUrl(registeredProviderUrl);
  exporter.setSubscribeUrl(overrideSubscribeUrl);
  return new DestroyableExporter<>(exporter);
}
```

```java
private <T> ExporterChangeableWrapper<T> doLocalExport(final Invoker<T> originInvoker, 
      URL providerUrl) {
  // 获取当前Invoker对应的key，默认为group/interface/version的格式
  String key = getCacheKey(originInvoker);

  // 这一段代码看起来比较复杂，其实本质上还是protocol.export()方法的调用，该方法就是进行服务暴露的代码，
  // 而ExporterChangeableWrapper的主要作用则是进行unexport()时的一些清理工作
  return (ExporterChangeableWrapper<T>) bounds.computeIfAbsent(key, s -> {
    Invoker<?> invokerDelegate = new InvokerDelegate<>(originInvoker, providerUrl);
    return new ExporterChangeableWrapper<>((Exporter<T>) protocol.export(invokerDelegate), 
        originInvoker);
  });
}
```

这里的protocol的类型为DubboProtocol，这里我们直接看其export()方法

```java
@Override
public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
  URL url = invoker.getUrl();
  
  String key = serviceKey(url);
  DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
  exporterMap.put(key, exporter);

  // 这里主要是构建Stub的事件分发器，该分发器用于在消费者端进行Stub事件的分发
  Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, 
      Constants.DEFAULT_STUB_EVENT);
  Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
  if (isStubSupportEvent && !isCallbackservice) {
    String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
    if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
      if (logger.isWarnEnabled()) {
        logger.warn(new IllegalStateException("consumer [" 
            + url.getParameter(Constants.INTERFACE_KEY) 
            + "], has set stubproxy support event ,but no stub methods founded."));
      }
    } else {
      stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
    }
  }

  // 开启服务
  openServer(url);
  // 该方法的主要作用是对序列化进行优化，其会获取配置的实现了SerializationOptimizer接口的配置类，
  // 然后通过其getSerializableClasses()方法获取序列化类，通过这些类来进行序列化的优化
  optimizeSerialization(url);

  return exporter;
}
```
export()方法主要做了三件事：a. 注册stub事件分发器；b. 开启服务；c. 注册序列化优化器类。这里openServer()方法是用于开启服务的

```java
private void openServer(URL url) {
  String key = url.getAddress();
  boolean isServer = url.getParameter(Constants.IS_SERVER_KEY, true);
  // 这里采用双检查法来判断对应于当前服务的server是否已经创建，如果没有创建，
  // 则创建一个新的，并且缓存起来
  if (isServer) {
    ExchangeServer server = serverMap.get(key);
    if (server == null) {
      synchronized (this) {
        server = serverMap.get(key);
        if (server == null) {
          // 创建并缓存新服务
          serverMap.put(key, createServer(url));
        }
      }
    } else {
      server.reset(url);
    }
  }
}

private ExchangeServer createServer(URL url) {
  url = URLBuilder.from(url)
    .addParameterIfAbsent(Constants.CHANNEL_READONLYEVENT_SENT_KEY, Boolean.TRUE.toString())
    .addParameterIfAbsent(Constants.HEARTBEAT_KEY, 
         String.valueOf(Constants.DEFAULT_HEARTBEAT))
    .addParameter(Constants.CODEC_KEY, DubboCodec.NAME)
    .build();
  
  // 获取所使用的server类型，默认为netty
  String str = url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_SERVER);
  if (str != null && str.length() > 0 
      && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
    throw new RpcException("Unsupported server type: " + str + ", url: " + url);
  }

  // 通过Exchangers.bind()方法进行服务的绑定
  ExchangeServer server;
  try {
    server = Exchangers.bind(url, requestHandler);
  } catch (RemotingException e) {
    throw new RpcException("Fail to start server(url: " + url + ") " + e.getMessage(), e);
  }

  // 获取client参数所指定的值，该值指定了当前client所使用的传输层服务，比如netty或mina。
  // 然后判断当前SPI所提供的传输层服务是否包含所指定的服务类型，如果不包含，则抛出异常
  str = url.getParameter(Constants.CLIENT_KEY);
  if (str != null && str.length() > 0) {
    Set<String> supportedTypes = ExtensionLoader.getExtensionLoader(Transporter.class)
      .getSupportedExtensions();
    if (!supportedTypes.contains(str)) {
      throw new RpcException("Unsupported client type: " + str);
    }
  }

  return server;
}
```

上面的代码主要是创建ExchangeServer的，使用双检查来检测是否已经存在了对应的服务，如果不存在，则通过Exchangers.bind()方法进行创建。这里最终会将bind()方法的调用委托给HeaderExchanger.bind()方法进行。需要注意的是，上面的代码中传入了一个requestHandler的参数，这是一个ExchangeHandler类型的对象，其主要作用是获取并且调用Invoker，以得到最终的调用结果，这些Handler的作用，我们将在后面的文章中进行讲解，本文主要讲解服务的暴露与注册的过程。下面我们继续阅读HeaderExchanger.bind()方法的源码：

```java
@Override
public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
  return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(
    new HeaderExchangeHandler(handler))));
}
```

这里的bind()方法主要是创建了三个Handler，并且最后一个Handler将传入的ExchangeHandler包裹起来了。相信读者朋友应该很快就能认识到，这里使用的是责任链模式，这几个handler通过统一的构造函数将下一个handler的实例注入到当前handler中。其实我们也就能够理解，最终通过netty进行的调用过程就是基于这些责任链的。这里我们主要看Transporters.bind()方法的实现原理：

```java
public static Server bind(URL url, ChannelHandler... handlers) throws RemotingException {
  if (url == null) {
    throw new IllegalArgumentException("url == null");
  }
  if (handlers == null || handlers.length == 0) {
    throw new IllegalArgumentException("handlers == null");
  }
  
  // 判断传入的Handler是否只有一个，如果只有一个，则直接使用该handler，如果存在多个，
  // 则使用ChannelHandlerDispatcher将这些handler包裹起来进行分发
  ChannelHandler handler;
  if (handlers.length == 1) {
    handler = handlers[0];
  } else {
    handler = new ChannelHandlerDispatcher(handlers);
  }
  
  // 通过配置指定的Transporter进行服务的绑定，这里默认使用的是NettyTransporter
  return getTransporter().bind(url, handler);
}

// NettyTransporter
@Override
public Server bind(URL url, ChannelHandler listener) throws RemotingException {
  // 在NettyTransporter中进行服务绑定时，其只是创建了一个NettyServer以返回，但实际上在创建该对象的
  // 过程中，就完成了Netty服务的绑定。需要注意的是，这里的NettyServer并不是Netty所提供的类，而是
  // Dubbo自己封装的一个服务类，其对Netty的服务进行了封装
  return new NettyServer(url, listener);
}
```

Transporters.bind()方法主要是将服务的绑定过程交由NettyTransporter进行，而其则是创建了一个NettyServer对象，真正的绑定过程就在创建该对象的过程中。下面我们来看其创建的源码：

```java
// AbstractServer
public AbstractServer(URL url, ChannelHandler handler) throws RemotingException {
  super(url, handler);
  localAddress = getUrl().toInetSocketAddress();

  // 获取绑定的ip和端口号等信息
  String bindIp = getUrl().getParameter(Constants.BIND_IP_KEY, getUrl().getHost());
  int bindPort = getUrl().getParameter(Constants.BIND_PORT_KEY, getUrl().getPort());
  if (url.getParameter(Constants.ANYHOST_KEY, false) || NetUtils.isInvalidLocalHost(bindIp)) 
  {
    bindIp = Constants.ANYHOST_VALUE;
  }
  
  // 在本地绑定指定的ip和端口
  bindAddress = new InetSocketAddress(bindIp, bindPort);
  this.accepts = url.getParameter(Constants.ACCEPTS_KEY, Constants.DEFAULT_ACCEPTS);
  this.idleTimeout = url.getParameter(Constants.IDLE_TIMEOUT_KEY, 
      Constants.DEFAULT_IDLE_TIMEOUT);
  try {
    // 通过创建的InetSocketAddress对象，将真正的绑定过程交由子类进行
    doOpen();
    if (logger.isInfoEnabled()) {
      logger.info("Start " + getClass().getSimpleName() + " bind " + getBindAddress() 
          + ", export " + getLocalAddress());
    }
  } catch (Throwable t) {
    throw new RemotingException(url.toInetSocketAddress(), null, "Failed to bind " 
        + getClass().getSimpleName() + " on " + getLocalAddress() + ", cause: " 
        + t.getMessage(), t);
  }

  // 这里的DataStore只是一个本地缓存的数据仓库，主要是对一些大对象进行缓存
  DataStore dataStore = ExtensionLoader.getExtensionLoader(DataStore.class)
      .getDefaultExtension();
  executor = (ExecutorService) dataStore.get(Constants.EXECUTOR_SERVICE_COMPONENT_KEY, 
      Integer.toString(url.getPort()));
}

// NettyServer
@Override
protected void doOpen() throws Throwable {
  bootstrap = new ServerBootstrap();

  // 这里就进入了创建netty服务的过程，bossGroup指定的线程数为1，因为只有一个channel用于接收客户端请求，
  // 而workerGroup线程数则指定为配置文件所设置的线程数，这些线程主要用于进行请求的处理
  bossGroup = new NioEventLoopGroup(1, new DefaultThreadFactory("NettyServerBoss", true));
  workerGroup = new NioEventLoopGroup(getUrl().getPositiveParameter(
    Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS),
    new DefaultThreadFactory("NettyServerWorker", true));

  // 创建NettyServerHandler，这个handler就是用于处理请求用的handler，但是前面我们也讲到了，
  // Dubbo使用了一个handler的责任链来进行消息的处理，第二个参数this就是这个链的链头。需要注意的是，
  // Netty本身提供的责任链与Dubbo这里使用的责任链是不同的，Dubbo只是使用了Netty的链的一个节点来
  // 处理Dubbo所创建的链，这样Dubbo的链其实是可以在多种服务复用的，比如Mina
  final NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this);
  channels = nettyServerHandler.getChannels();

  // 这里是标准的创建Netty的BootstrapServer的过程
  bootstrap.group(bossGroup, workerGroup)
    .channel(NioServerSocketChannel.class)
    .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
    .childOption(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
    .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
    .childHandler(new ChannelInitializer<NioSocketChannel>() {
      @Override
      protected void initChannel(NioSocketChannel ch) throws Exception {
        int idleTimeout = UrlUtils.getIdleTimeout(getUrl());
        NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), 
            NettyServer.this);
        ch.pipeline()
          // 添加用于解码的handler
          .addLast("decoder", adapter.getDecoder())
          // 添加用于编码的handler
          .addLast("encoder", adapter.getEncoder())
          // 添加用于进行心跳监测的handler
          .addLast("server-idle-handler", new IdleStateHandler(0, 0, 
              idleTimeout, MILLISECONDS))
          // 将处理请求的handler添加到pipeline中
          .addLast("handler", nettyServerHandler);
      }
    });

  // 进行服务的绑定
  ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
  channelFuture.syncUninterruptibly();
  channel = channelFuture.channel();

}
```

对于服务的注册，前面我们已经讲到，入口主要在RegistryProtocol.export()方法中，而调用入口则是通过其register()方法进行的，这里我们来看一下该方法的调用过程：

```java
public void register(URL registryUrl, URL registeredProviderUrl) {
  // 通过RegistryFactory获取一个Registry对象，该对象的主要作用是进行服务的注册，
  // 这里默认返回的是ZookeeperRegistry
  Registry registry = registryFactory.getRegistry(registryUrl);
  registry.register(registeredProviderUrl);
}
```

```java
// FailbackRegistry
@Override
public void register(URL url) {
  // 将当前URL对象保存到已注册的URL对象列表中
  super.register(url);
  // 移除之前注册失败的记录
  removeFailedRegistered(url);
  removeFailedUnregistered(url);
  try {
    // 将真正的注册过程委托给ZookeeperRegistry进行
    doRegister(url);
  } catch (Exception e) {
    Throwable t = e;

    // 下面的过程主要是在注册失败的情况下，将当前URL添加到注册失败的URL列表中
    boolean check = getUrl().getParameter(Constants.CHECK_KEY, true)
      && url.getParameter(Constants.CHECK_KEY, true)
      && !Constants.CONSUMER_PROTOCOL.equals(url.getProtocol());
    boolean skipFailback = t instanceof SkipFailbackWrapperException;
    if (check || skipFailback) {
      if (skipFailback) {
        t = t.getCause();
      }
      throw new IllegalStateException("Failed to register " + url + " to registry " 
          + getUrl().getAddress() + ", cause: " + t.getMessage(), t);
    } else {
      logger.error("Failed to register " + url + ", waiting for retry, cause: " 
          + t.getMessage(), t);
    }

    // 将当前URL添加到注册失败的URL列表中
    addFailedRegistered(url);
  }
}

// ZookeeperRegistry
@Override
public void doRegister(URL url) {
  try {
    // 这里是真正的注册过程。需要注意的是这里的zkClient类型为ZookeeperClient，其是Dubbo对
    // 真正使用的CuratorFramework的一个封装
    zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
  } catch (Throwable e) {
    throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() 
        + ", cause: " + e.getMessage(), e);
  }
}
```

上面的代码中首先会对一些缓存数据进行清理，并且将当前URL添加到注册的URL列表中，然后将注册过程委托给ZookeeperClient进行。下面我们来看其是如何进行注册的：

```java
@Override
public void create(String path, boolean ephemeral) {
  // 判断创建的是否为临时节点，如果不是临时节点，则判断是否已经存在该节点，如果存在，则直接返回
  if (!ephemeral) {
    if (checkExists(path)) {
      return;
    }
  }
  
  // 对path进行截取，因为最后一个"/"后面是被编码的URL对象，前面则是serviceKey + category
  // 这里的category指定的是provider还是consumer
  int i = path.lastIndexOf('/');
  if (i > 0) {
    // 创建节点，需要注意的是，这里的create()方法进行的是递归调用，这是因为zookeeper创建节点时
    // 只能一级一级的创建，因而其每次都是取"/"前面的一部分来创建，只有当前节点已经存在的情况下，
    // 上面的checkExists()才会为true，而且这里，由于zookeeper规定，除了叶节点以外，其余所有的
    // 节点都必须为非临时节点，因而这里第二个参数传入的是false，这也是前面的if判断能通过的原因
    create(path.substring(0, i), false);
  }
  
  if (ephemeral) {
    // 创建临时节点，具体的创建工作交由子类进行，也就是下面的代码
    createEphemeral(path);
  } else {
    // 创建持久节点，具体的创建工作交由子类进行，也就是下面的代码
    createPersistent(path);
  }
}
```

```java
@Override
public void createEphemeral(String path) {
  try {
    // 将临时节点的创建工作交由CuratorFramework进行
    client.create().withMode(CreateMode.EPHEMERAL).forPath(path);
  } catch (NodeExistsException e) {
  } catch (Exception e) {
    throw new IllegalStateException(e.getMessage(), e);
  }
}
```

```java
@Override
public void createPersistent(String path) {
  try {
    // 将持久节点的创建工作交由CuratorFramework进行
    client.create().forPath(path);
  } catch (NodeExistsException e) {
  } catch (Exception e) {
    throw new IllegalStateException(e.getMessage(), e);
  }
}
```


## 3. 服务引用

服务引用的入口方法为 ReferenceBean 的 getObject 方法

```java
public Object getObject() throws Exception {
        return get();
    }
```

然后到com.alibaba.dubbo.config.ReferenceConfig#get方法

```java
public synchronized T get() {
    if (destroyed) {
        throw new IllegalStateException("Already destroyed!");
    }
    // 检测 ref 是否为空，为空则通过 init 方法创建
    if (ref == null) {
        // init 方法主要用于处理配置，以及调用 createProxy 生成代理类
        init();
    }
    return ref;
}
```

然后到com.alibaba.dubbo.config.ReferenceConfig#init

```java
private void init() {
    // 避免重复初始化
    if (initialized) {
        return;
    }
    initialized = true;
    // 检测接口名合法性
    if (interfaceName == null || interfaceName.length() == 0) {
        throw new IllegalStateException("interface not allow null!");
    }
 
    // 检测 consumer 变量是否为空，为空则创建
    checkDefault();
    appendProperties(this);
    if (getGeneric() == null && getConsumer() != null) {
        // 设置 generic
        setGeneric(getConsumer().getGeneric());
    }
 
    // 检测是否为泛化接口
    if (ProtocolUtils.isGeneric(getGeneric())) {
        interfaceClass = GenericService.class;
    } else {
        try {
            // 加载类
            interfaceClass = Class.forName(interfaceName, true, Thread.currentThread()
                    .getContextClassLoader());
        } catch (ClassNotFoundException e) {
            throw new IllegalStateException(e.getMessage(), e);
        }
        checkInterfaceAndMethods(interfaceClass, methods);
    }
    
    // -------------------------------分割线1------------------------------
 
    // 从系统变量中获取与接口名对应的属性值
    String resolve = System.getProperty(interfaceName);
    String resolveFile = null;
    if (resolve == null || resolve.length() == 0) {
        // 从系统属性中获取解析文件路径
        resolveFile = System.getProperty("dubbo.resolve.file");
        if (resolveFile == null || resolveFile.length() == 0) {
            // 从指定位置加载配置文件
            File userResolveFile = new File(new File(System.getProperty("user.home")), "dubbo-resolve.properties");
            if (userResolveFile.exists()) {
                // 获取文件绝对路径
                resolveFile = userResolveFile.getAbsolutePath();
            }
        }
        if (resolveFile != null && resolveFile.length() > 0) {
            Properties properties = new Properties();
            FileInputStream fis = null;
            try {
                fis = new FileInputStream(new File(resolveFile));
                // 从文件中加载配置
                properties.load(fis);
            } catch (IOException e) {
                throw new IllegalStateException("Unload ..., cause:...");
            } finally {
                try {
                    if (null != fis) fis.close();
                } catch (IOException e) {
                    logger.warn(e.getMessage(), e);
                }
            }
            // 获取与接口名对应的配置
            resolve = properties.getProperty(interfaceName);
        }
    }
    if (resolve != null && resolve.length() > 0) {
        // 将 resolve 赋值给 url
        url = resolve;
    }
    
    // -------------------------------分割线2------------------------------
    if (consumer != null) {
        if (application == null) {
            // 从 consumer 中获取 Application 实例，下同
            application = consumer.getApplication();
        }
        if (module == null) {
            module = consumer.getModule();
        }
        if (registries == null) {
            registries = consumer.getRegistries();
        }
        if (monitor == null) {
            monitor = consumer.getMonitor();
        }
    }
    if (module != null) {
        if (registries == null) {
            registries = module.getRegistries();
        }
        if (monitor == null) {
            monitor = module.getMonitor();
        }
    }
    if (application != null) {
        if (registries == null) {
            registries = application.getRegistries();
        }
        if (monitor == null) {
            monitor = application.getMonitor();
        }
    }
    
    // 检测 Application 合法性
    checkApplication();
    // 检测本地存根配置合法性
    checkStubAndMock(interfaceClass);
    
	// -------------------------------分割线3------------------------------
    
    Map<String, String> map = new HashMap<String, String>();
    Map<Object, Object> attributes = new HashMap<Object, Object>();
 
    // 添加 side、协议版本信息、时间戳和进程号等信息到 map 中
    map.put(Constants.SIDE_KEY, Constants.CONSUMER_SIDE);
    map.put(Constants.DUBBO_VERSION_KEY, Version.getProtocolVersion());
    map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
    if (ConfigUtils.getPid() > 0) {
        map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
    }
 
    // 非泛化服务
    if (!isGeneric()) {
        // 获取版本
        String revision = Version.getVersion(interfaceClass, version);
        if (revision != null && revision.length() > 0) {
            map.put("revision", revision);
        }
 
        // 获取接口方法列表，并添加到 map 中
        String[] methods = Wrapper.getWrapper(interfaceClass).getMethodNames();
        if (methods.length == 0) {
            map.put("methods", Constants.ANY_VALUE);
        } else {
            map.put("methods", StringUtils.join(new HashSet<String>(Arrays.asList(methods)), ","));
        }
    }
    map.put(Constants.INTERFACE_KEY, interfaceName);
    // 将 ApplicationConfig、ConsumerConfig、ReferenceConfig 等对象的字段信息添加到 map 中
    appendParameters(map, application);
    appendParameters(map, module);
    appendParameters(map, consumer, Constants.DEFAULT_KEY);
    appendParameters(map, this);
    
	// -------------------------------分割线4------------------------------
    
    String prefix = StringUtils.getServiceKey(map);
    if (methods != null && !methods.isEmpty()) {
        // 遍历 MethodConfig 列表
        for (MethodConfig method : methods) {
            appendParameters(map, method, method.getName());
            String retryKey = method.getName() + ".retry";
            // 检测 map 是否包含 methodName.retry
            if (map.containsKey(retryKey)) {
                String retryValue = map.remove(retryKey);
                if ("false".equals(retryValue)) {
                    // 添加重试次数配置 methodName.retries
                    map.put(method.getName() + ".retries", "0");
                }
            }
 
            // 添加 MethodConfig 中的“属性”字段到 attributes
            // 比如 onreturn、onthrow、oninvoke 等
            appendAttributes(attributes, method, prefix + "." + method.getName());
            checkAndConvertImplicitConfig(method, map, attributes);
        }
    }
    
	// -------------------------------分割线5------------------------------
 
    // 获取服务消费者 ip 地址
    String hostToRegistry = ConfigUtils.getSystemProperty(Constants.DUBBO_IP_TO_REGISTRY);
    if (hostToRegistry == null || hostToRegistry.length() == 0) {
        hostToRegistry = NetUtils.getLocalHost();
    } else if (isInvalidLocalHost(hostToRegistry)) {
        throw new IllegalArgumentException("Specified invalid registry ip from property..." );
    }
    map.put(Constants.REGISTER_IP_KEY, hostToRegistry);
 
    // 存储 attributes 到系统上下文中
    StaticContext.getSystemContext().putAll(attributes);
 
    // 创建代理类
    ref = createProxy(map);
 
    // 根据服务名，ReferenceConfig，代理类构建 ConsumerModel，
    // 并将 ConsumerModel 存入到 ApplicationModel 中
    ConsumerModel consumerModel = new ConsumerModel(getUniqueServiceName(), this, ref, interfaceClass.getMethods());
    ApplicationModel.initConsumerModel(getUniqueServiceName(), consumerModel);
}
```

首先是方法开始到分割线1之间的代码。这段代码主要用于检测 ConsumerConfig 实例是否存在，如不存在则创建一个新的实例，然后通过系统变量或 dubbo.properties 配置文件填充 ConsumerConfig 的字段。接着是检测泛化配置，并根据配置设置 interfaceClass 的值。接着来看分割线1到分割线2之间的逻辑。这段逻辑用于从系统属性或配置文件中加载与接口名相对应的配置，并将解析结果赋值给 url 字段。url 字段的作用一般是用于点对点调用。继续向下看，分割线2和分割线3之间的代码用于检测几个核心配置类是否为空，为空则尝试从其他配置类中获取。分割线3与分割线4之间的代码主要用于收集各种配置，并将配置存储到 map 中。分割线4和分割线5之间的代码用于处理 MethodConfig 实例。该实例包含了事件通知配置，比如 onreturn、onthrow、oninvoke 等。分割线5到方法结尾的代码主要用于解析服务消费者 ip，以及调用 createProxy 创建代理对象。

接下来分析 createProxy 创建代理对象,com.alibaba.dubbo.config.ReferenceConfig#createProxy

```java
private T createProxy(Map<String, String> map) {
    URL tmpUrl = new URL("temp", "localhost", 0, map);
    final boolean isJvmRefer;
    if (isInjvm() == null) {
        // url 配置被指定，则不做本地引用
        if (url != null && url.length() > 0) {
            isJvmRefer = false;
        // 根据 url 的协议、scope 以及 injvm 等参数检测是否需要本地引用
        // 比如如果用户显式配置了 scope=local，此时 isInjvmRefer 返回 true
        } else if (InjvmProtocol.getInjvmProtocol().isInjvmRefer(tmpUrl)) {
            isJvmRefer = true;
        } else {
            isJvmRefer = false;
        }
    } else {
        // 获取 injvm 配置值
        isJvmRefer = isInjvm().booleanValue();
    }
 
    // 本地引用
    if (isJvmRefer) {
        // 生成本地引用 URL，协议为 injvm
        URL url = new URL(Constants.LOCAL_PROTOCOL, NetUtils.LOCALHOST, 0, interfaceClass.getName()).addParameters(map);
        // 调用 refer 方法构建 InjvmInvoker 实例
        invoker = refprotocol.refer(interfaceClass, url);
        
    // 远程引用
    } else {
        // url 不为空，表明用户可能想进行点对点调用
        if (url != null && url.length() > 0) {
            // 当需要配置多个 url 时，可用分号进行分割，这里会进行切分
            String[] us = Constants.SEMICOLON_SPLIT_PATTERN.split(url);
            if (us != null && us.length > 0) {
                for (String u : us) {
                    URL url = URL.valueOf(u);
                    if (url.getPath() == null || url.getPath().length() == 0) {
                        // 设置接口全限定名为 url 路径
                        url = url.setPath(interfaceName);
                    }
                    
                    // 检测 url 协议是否为 registry，若是，表明用户想使用指定的注册中心
                    if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        // 将 map 转换为查询字符串，并作为 refer 参数的值添加到 url 中
                        urls.add(url.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                    } else {
                        // 合并 url，移除服务提供者的一些配置（这些配置来源于用户配置的 url 属性），
                        // 比如线程池相关配置。并保留服务提供者的部分配置，比如版本，group，时间戳等
                        // 最后将合并后的配置设置为 url 查询字符串中。
                        urls.add(ClusterUtils.mergeUrl(url, map));
                    }
                }
            }
        } else {
            // 加载注册中心 url
            List<URL> us = loadRegistries(false);
            if (us != null && !us.isEmpty()) {
                for (URL u : us) {
                    URL monitorUrl = loadMonitor(u);
                    if (monitorUrl != null) {
                        map.put(Constants.MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                    }
                    // 添加 refer 参数到 url 中，并将 url 添加到 urls 中
                    urls.add(u.addParameterAndEncoded(Constants.REFER_KEY, StringUtils.toQueryString(map)));
                }
            }
 
            // 未配置注册中心，抛出异常
            if (urls.isEmpty()) {
                throw new IllegalStateException("No such any registry to reference...");
            }
        }
 
        // 单个注册中心或服务提供者(服务直连，下同)
        if (urls.size() == 1) {
            // 调用 RegistryProtocol 的 refer 构建 Invoker 实例
            invoker = refprotocol.refer(interfaceClass, urls.get(0));
            
        // 多个注册中心或多个服务提供者，或者两者混合
        } else {
            List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
            URL registryURL = null;
 
            // 获取所有的 Invoker
            for (URL url : urls) {
                // 通过 refprotocol 调用 refer 构建 Invoker，refprotocol 会在运行时
                // 根据 url 协议头加载指定的 Protocol 实例，并调用实例的 refer 方法
                invokers.add(refprotocol.refer(interfaceClass, url));
                if (Constants.REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                    registryURL = url;
                }
            }
            if (registryURL != null) {
                // 如果注册中心链接不为空，则将使用 AvailableCluster
                URL u = registryURL.addParameter(Constants.CLUSTER_KEY, AvailableCluster.NAME);
                // 创建 StaticDirectory 实例，并由 Cluster 对多个 Invoker 进行合并
                invoker = cluster.join(new StaticDirectory(u, invokers));
            } else {
                invoker = cluster.join(new StaticDirectory(invokers));
            }
        }
    }
 
    Boolean c = check;
    if (c == null && consumer != null) {
        c = consumer.isCheck();
    }
    if (c == null) {
        c = true;
    }
    
    // invoker 可用性检查
    if (c && !invoker.isAvailable()) {
        throw new IllegalStateException("No provider available for the service...");
    }
 
    // 生成代理类
    return (T) proxyFactory.getProxy(invoker);
}
```

Invoker 是 Dubbo 的核心模型，代表一个可执行体。在服务提供方，Invoker 用于调用服务提供类。在服务消费方，Invoker 用于执行远程调用。Invoker 是由 Protocol 实现类构建而来。Protocol 实现类有很多，本节会分析最常用的两个，分别是 RegistryProtocol 和 DubboProtocol，其他的大家自行分析。下面先来分析 DubboProtocol 的 refer 方法源码。如下：

```java
public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
    optimizeSerialization(url);
    // 创建 DubboInvoker
    DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
    invokers.add(invoker);
    return invoker;
}
```

其中有一个方法getClients(url)，这个方法用于获取客户端实例，实例类型为 ExchangeClient。ExchangeClient 实际上并不具备通信能力，它需要基于更底层的客户端实例进行通信。比如 NettyClient、MinaClient 等，默认情况下，Dubbo 使用 NettyClient 进行通信。接下来，我们简单看一下 getClients 方法的逻辑。

com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol#getClients

```java
private ExchangeClient[] getClients(URL url) {
    // 是否共享连接
    boolean service_share_connect = false;
  	// 获取连接数，默认为0，表示未配置
    int connections = url.getParameter(Constants.CONNECTIONS_KEY, 0);
    // 如果未配置 connections，则共享连接
    if (connections == 0) {
        service_share_connect = true;
        connections = 1;
    }
 
    ExchangeClient[] clients = new ExchangeClient[connections];
    for (int i = 0; i < clients.length; i++) {
        if (service_share_connect) {
            // 获取共享客户端
            clients[i] = getSharedClient(url);
        } else {
            // 初始化新的客户端
            clients[i] = initClient(url);
        }
    }
    return clients;
}
```

这里根据 connections 数量决定是获取共享客户端还是创建新的客户端实例，默认情况下，使用共享客户端实例。getSharedClient 方法中也会调用 initClient 方法，因此下面我们一起看一下这两个方法。

```java
private ExchangeClient getSharedClient(URL url) {
    String key = url.getAddress();
    // 获取带有“引用计数”功能的 ExchangeClient
    ReferenceCountExchangeClient client = referenceClientMap.get(key);
    if (client != null) {
        if (!client.isClosed()) {
            // 增加引用计数
            client.incrementAndGetCount();
            return client;
        } else {
            referenceClientMap.remove(key);
        }
    }
 
    locks.putIfAbsent(key, new Object());
    synchronized (locks.get(key)) {
        if (referenceClientMap.containsKey(key)) {
            return referenceClientMap.get(key);
        }
 
        // 创建 ExchangeClient 客户端
        ExchangeClient exchangeClient = initClient(url);
        // 将 ExchangeClient 实例传给 ReferenceCountExchangeClient，这里使用了装饰模式
        client = new ReferenceCountExchangeClient(exchangeClient, ghostClientMap);
        referenceClientMap.put(key, client);
        ghostClientMap.remove(key);
        locks.remove(key);
        return client;
    }
}
``` 

上面方法先访问缓存，若缓存未命中，则通过 initClient 方法创建新的 ExchangeClient 实例，并将该实例传给 ReferenceCountExchangeClient 构造方法创建一个带有引用计数功能的 ExchangeClient 实例。

```java
private ExchangeClient initClient(URL url) {
 
    // 获取客户端类型，默认为 netty
    String str = url.getParameter(Constants.CLIENT_KEY, url.getParameter(Constants.SERVER_KEY, Constants.DEFAULT_REMOTING_CLIENT));
 
    // 添加编解码和心跳包参数到 url 中
    url = url.addParameter(Constants.CODEC_KEY, DubboCodec.NAME);
    url = url.addParameterIfAbsent(Constants.HEARTBEAT_KEY, String.valueOf(Constants.DEFAULT_HEARTBEAT));
 
    // 检测客户端类型是否存在，不存在则抛出异常
    if (str != null && str.length() > 0 && !ExtensionLoader.getExtensionLoader(Transporter.class).hasExtension(str)) {
        throw new RpcException("Unsupported client type: ...");
    }
 
    ExchangeClient client;
    try {
        // 获取 lazy 配置，并根据配置值决定创建的客户端类型
        if (url.getParameter(Constants.LAZY_CONNECT_KEY, false)) {
            // 创建懒加载 ExchangeClient 实例
            client = new LazyConnectExchangeClient(url, requestHandler);
        } else {
            // 创建普通 ExchangeClient 实例
            client = Exchangers.connect(url, requestHandler);
        }
    } catch (RemotingException e) {
        throw new RpcException("Fail to create remoting client for service...");
    }
    return client;
}
```

initClient 方法首先获取用户配置的客户端类型，默认为 netty。然后检测用户配置的客户端类型是否存在，不存在则抛出异常。最后根据 lazy 配置决定创建什么类型的客户端。这里的 LazyConnectExchangeClient 代码并不是很复杂，该类会在 request 方法被调用时通过 Exchangers 的 connect 方法创建 ExchangeClient 客户端

下面我们分析一下 Exchangers 的 connect 方法。com.alibaba.dubbo.remoting.exchange.Exchangers#connect(com.alibaba.dubbo.common.URL, com.alibaba.dubbo.remoting.exchange.ExchangeHandler)

```java
public static ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    if (handler == null) {
        throw new IllegalArgumentException("handler == null");
    }
    url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
    // 获取 Exchanger 实例，默认为 HeaderExchangeClient
    return getExchanger(url).connect(url, handler);
}
```

如上，getExchanger 会通过 SPI 加载 HeaderExchangeClient 实例，这个方法比较简单，大家自己看一下吧。接下来分析 HeaderExchangeClient 的实现。com.alibaba.dubbo.remoting.exchange.support.header.HeaderExchanger#connect

```java
public ExchangeClient connect(URL url, ExchangeHandler handler) throws RemotingException {
    // 这里包含了多个调用，分别如下：
    // 1. 创建 HeaderExchangeHandler 对象
    // 2. 创建 DecodeHandler 对象
    // 3. 通过 Transporters 构建 Client 实例
    // 4. 创建 HeaderExchangeClient 对象
    return new HeaderExchangeClient(Transporters.connect(url, new DecodeHandler(new HeaderExchangeHandler(handler))), true);
}
```

这里的调用比较多，我们这里重点看一下 Transporters 的 connect 方法。

```java
public static Client connect(URL url, ChannelHandler... handlers) throws RemotingException {
    if (url == null) {
        throw new IllegalArgumentException("url == null");
    }
    ChannelHandler handler;
    if (handlers == null || handlers.length == 0) {
        handler = new ChannelHandlerAdapter();
    } else if (handlers.length == 1) {
        handler = handlers[0];
    } else {
        // 如果 handler 数量大于1，则创建一个 ChannelHandler 分发器
        handler = new ChannelHandlerDispatcher(handlers);
    }
    
    // 获取 Transporter 自适应拓展类，并调用 connect 方法生成 Client 实例
    return getTransporter().connect(url, handler);
}
```

如上，getTransporter 方法返回的是自适应拓展类，该类会在运行时根据客户端类型加载指定的 Transporter 实现类。若用户未配置客户端类型，则默认加载 NettyTransporter，并调用该类的 connect 方法。如下：

```java
public Client connect(URL url, ChannelHandler listener) throws RemotingException {
    // 创建 NettyClient 对象
    return new NettyClient(url, listener);
}

public NettyClient(final URL url, final ChannelHandler handler) throws RemotingException {
    super(url, wrapChannelHandler(url, handler));
}

protected static ChannelHandler wrapChannelHandler(URL url, ChannelHandler handler) {
    url = ExecutorUtil.setThreadName(url, CLIENT_THREAD_POOL_NAME);
    url = url.addParameterIfAbsent(Constants.THREADPOOL_KEY, Constants.DEFAULT_CLIENT_THREADPOOL);
    return ChannelHandlers.wrap(handler, url);
}

public static ChannelHandler wrap(ChannelHandler handler, URL url) {
    return ChannelHandlers.getInstance().wrapInternal(handler, url);
}

protected ChannelHandler wrapInternal(ChannelHandler handler, URL url) {
    return new MultiMessageHandler(new HeartbeatHandler(ExtensionLoader.getExtensionLoader(Dispatcher.class)
            .getAdaptiveExtension().dispatch(handler, url)));
}
```

com.alibaba.dubbo.remoting.transport.netty4.NettyClient#doOpen

```java
@Override
protected void doOpen() throws Throwable {
    NettyHelper.setNettyLoggerFactory();
    final NettyClientHandler nettyClientHandler = new NettyClientHandler(getUrl(), this);
    bootstrap = new Bootstrap();
    bootstrap.group(nioEventLoopGroup)
            .option(ChannelOption.SO_KEEPALIVE, true)
            .option(ChannelOption.TCP_NODELAY, true)
            .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
            //.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, getTimeout())
            .channel(NioSocketChannel.class);

    if (getTimeout() < 3000) {
        bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000);
    } else {
        bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, getTimeout());
    }

    bootstrap.handler(new ChannelInitializer() {

        protected void initChannel(Channel ch) throws Exception {
            NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyClient.this);
            ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                    .addLast("decoder", adapter.getDecoder())
                    .addLast("encoder", adapter.getEncoder())
                    .addLast("handler", nettyClientHandler);
        }
    });
}
```

channelhandler.wrap 里面会调用Dispatcher 的扩展，进行dispatch操作，实际是对handler 的包装动态化

## 4. 服务调用

## 5. SPI机制原理

## 6. 容错原理

超时策略：
- 精确优先（方法优先，接口次之，全局配置再次）
- 消费方优先（级别一样，消费方优先，提供方再次）

重试策略（不包含第一次）
- 幂等操作可设置重试次数，非幂等不可重试

**服务降级**

**容错**

## 7. 负载均衡

1. 权重随机
2. 权重轮询
3. 最少活跃数
4. 一致性hash

## 路由原理
