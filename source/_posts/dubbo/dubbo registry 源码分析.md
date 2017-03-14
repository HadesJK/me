---
title: dubbo registry 源码分析
date: 2017-03-14 20:32:24
categories:
---

# 主要类图关系

![](/images/dubbo/main.class.png)

# RegisterFactory
![](/images/dubbo/RegistryFactory.png)

RegisterFactory 是接口，只有一个方法，用来创建Register

实现类有4个：
- DubboRegistryFactory
- ==**ZookeeperRegistryFactory**==
- MulticastRegistryFactory
- RedisRegistryFactory
前三个工厂类又继承自抽象类 ==**AbstractRegistryFactory**==。
分别返回4种类型的 Registry：
DubboRegistry， **==ZookeeperRegistry==** ， MulticastRegistry，RedisRegistry。

# Registry
![](/images/dubbo/RegistryService.png)

Registry 接口继承 RegistryService 接口，主要的5个方法：
```java
void register(URL url);
void unregister(URL url);
void subscribe(URL url, NotifyListener listener);
void unsubscribe(URL url, NotifyListener 
List<URL> lookup(URL url);
```
dubbo 注册中心的核心类主要是以上几个，下面分析加粗的几个类。

---

# AbstractRegistry

## 字段解释

```java
// URL地址分隔符，用于文件缓存中，服务提供者URL分隔
private static final char URL_SEPARATOR = ' ';

// URL地址分隔正则表达式，用于解析文件缓存中服务提供者URL列表
private static final String URL_SPLIT = "\\s+";

// 用来存储注册中心相关信息
private URL registryUrl;

// 本地磁盘缓存文件，注册中心保持了服务的信息，这个文件作为这些服务的本地文件缓存
private File file;

// 本地磁盘缓存，其中特殊的key值.registies记录注册中心列表，其它均为notified服务提供者列表
// Properties对象会读取本地磁盘缓存文件，也会将自身的信息刷入本地磁盘缓存文件
// 从代码的实现中可以看出，会有一个对应fileName.lock的文件来作为进程锁，对file进行写操作
private final Properties properties = new Properties();

// 文件缓存定时写入
private final ExecutorService registryCacheExecutor = Executors.newFixedThreadPool(1, new NamedThreadFactory("DubboSaveRegistryCache", true));

//是否是同步保存文件
private final boolean syncSaveFile ;

// 相当于版本号的功能，防止旧的替换新的
private final AtomicLong lastCacheChanged = new AtomicLong();

// 需要注册的服务集合
// 即使服务注册失败了，这个集合中仍有该服务的信息
// 但是如果服务取消注册了，将从这个集合中移除
private final Set<URL> registered = new ConcurrentHashSet<URL>();

// 需要订阅的服务集合
// 这个数据结构竟然是Map<Key, Set>的形式！
// 为什么value需要是一个Set<NotifyListener> 而不是一个NotifyListener！
// TODO:求解释啊
private final ConcurrentMap<URL, Set<NotifyListener>> subscribed = new ConcurrentHashMap<URL, Set<NotifyListener>>();

private final ConcurrentMap<URL, Map<String, List<URL>>> notified = new ConcurrentHashMap<URL, Map<String, List<URL>>>();
```

---

**==构造器==**
构造器主要做了4件事：
1. 初始化一些参数
2. 创建了本地缓存文件的父级文件夹
- 默认的文件是 **==~/.dubbo/dubbo-registry-${ip}.cache"==**，下面称为cache文件
- 其中${ip}是dubbo自定义的url中获取的
- 文件名中加入了ip的因素是因为dubbo可以采用多注册中心
- 因此这个文件是对注册中心服务的缓存，而不是针对某一个服务提供者的
- 对这个文件的操作采用了进程间锁，cache.lock来实现的，可以防止一个机器部署多个dubbo应用造成对这个文件并发操作
3. 加载上面那个位置的属性文件的数据
4. 执行notify，后面分析这个方法

**dubbo register创建的文件**
![](/images/dubbo/dubbo.cache.png)

上图中，.cache结尾的是对注册中心的服务信息本地磁盘文件缓存
.lock是对.cache文件的进程间锁实现，所以这个文件没有任何数据

```java
public AbstractRegistry(URL url) {
    setUrl(url);
    // 启动文件保存定时器
    syncSaveFile = url.getParameter(Constants.REGISTRY_FILESAVE_SYNC_KEY, false);
    String filename = url.getParameter(Constants.FILE_KEY, System.getProperty("user.home") + "/.dubbo/dubbo-registry-" + url.getHost() + ".cache");
    File file = null;
    if (ConfigUtils.isNotEmpty(filename)) {
        file = new File(filename);
        if(! file.exists() && file.getParentFile() != null && ! file.getParentFile().exists()){
	    // 创建父级文件夹，如果失败，抛出异常
            if(! file.getParentFile().mkdirs()){
                throw new IllegalArgumentException("Invalid registry store file " + file + ", cause: Failed to create directory " + file.getParentFile() + "!");
            }
        }
    }
    this.file = file;
    loadProperties();
    notify(url.getBackupUrls());
}
```
dubbo在使用过程中，采用了注册中心，而注册中心即可以是分布式系统，也可以是单节点，为了避免单点问题，dubbo消费者缓存了提供者的信息。从构造器中看出，dubbo不仅内存中缓存了服务提供者的信息，而且还通过本地磁盘文件缓存这些信息！这样子，即使注册中心宕机了，只要服务提供者依然可用，消费者仍然可以调用提供者的服务，提高了系统的可用性。
问题：本地文件缓存用了一个线程去保存，但是感觉没什么必要，完全可以内存中保存。
dubbo文档对这个的解释如下：【我感觉画蛇添足的作用】
![](/images/dubbo/dubbo.cache.intro.png)

**AbstractRegistry#doSaveProperties(long version)**
该方法主要用来主动加载本地磁盘缓存的数据，即加载cache文件中的属性，加载完成之后，结合本地的属性值，全量写入到cache文件中，同时失败的话(多jvm实例下，获取不到进程锁，进程锁通过.cache.lock实现)，通过执行器(执行器是单线程)异步重试。

**AbstractRegistry#notify(List<URL> urls)**
// TODO:
结合RegistryDirectory#notify【consumer】和RegistryProtocol$OverrideListener#notify【provider】方法讲

---

# FailbackRegistry

failback
出故障时，自动恢复
failover
出故障时，设备失效

## 字段

```
// 定时任务执行器
private final ScheduledExecutorService retryExecutor = Executors.newScheduledThreadPool(1, new NamedThreadFactory("DubboRegistryFailedRetryTimer", true));

// 失败重试定时器，定时检查是否有请求失败，如有，无限次重试
private final ScheduledFuture<?> retryFuture;

/**
* 一下5个集合针对 RegistryService 接口的5个方法操作失败情况下
* 记录下操作失败的URL，然后使用重试任务异步重试
*/
// 注册失败的接口
private final Set<URL> failedRegistered = new ConcurrentHashSet<URL>();
// 取消注册失败的接口
private final Set<URL> failedUnregistered = new ConcurrentHashSet<URL>();
// 订阅失败的接口
private final ConcurrentMap<URL, Set<NotifyListener>> failedSubscribed = new ConcurrentHashMap<URL, Set<NotifyListener>>();
// 取消订阅失败的接口
private final ConcurrentMap<URL, Set<NotifyListener>> failedUnsubscribed = new ConcurrentHashMap<URL, Set<NotifyListener>>();
private final ConcurrentMap<URL, Map<NotifyListener, List<URL>>> failedNotified = new ConcurrentHashMap<URL, Map<NotifyListener, List<URL>>>();
```

## 构造方法

每隔一定的时间去检查5个集合中的操作失败的服务，然后尝试执行。
```java
public FailbackRegistry(URL url) {
    super(url);
    // 每隔 ${retryPeriod} 毫秒，都会去重试，默认每隔5000毫秒
    int retryPeriod = url.getParameter(Constants.REGISTRY_RETRY_PERIOD_KEY, Constants.DEFAULT_REGISTRY_RETRY_PERIOD);
    // 提交一个异步任务
    this.retryFuture = retryExecutor.scheduleWithFixedDelay(new Runnable() {
        public void run() {
            try {
                retry();
            } catch (Throwable t) { // 防御性容错
                logger.error("Unexpected error occur at failed retry, cause: " + t.getMessage(), t);
            }
        }
    }, retryPeriod, retryPeriod, TimeUnit.MILLISECONDS);
}
```

看 **==retry==** 方法，对于5个失败的集合，分别执行方法
doRegister
doUnregister
doSubscribe
doUnsubscribe
listener.notify()

而doXXX方法是由子类实现的

---

# ZookeeperRegistry

**很关键**

## 字段

```java
// zk的默认端口
private final static int DEFAULT_ZOOKEEPER_PORT = 2181;

// dubbo默认在zk下创建的根结点是 /dubbo, 其它所有节点都是这个节点的子节点
private final static String DEFAULT_ROOT = "dubbo";
// 默认情况下 root 就是 /dubbo
private final String        root;

private final Set<String> anyServices = new ConcurrentHashSet<String>();

// key 是具体的服务， value是这些服务的watch抽象
private final ConcurrentMap<URL, ConcurrentMap<NotifyListener, ChildListener>> zkListeners = new ConcurrentHashMap<URL, ConcurrentMap<NotifyListener, ChildListener>>();

// zk client    
private final ZookeeperClient zkClient;
```

## 构造方法

1. 调用父类的构造方法，即执行了AbstractRegistry，FailbackRegistry
2. 初始化dubbo的根结点，默认是/dubbo，实际使用中，这个参数一般使用默认值
3. zkClient连接zk集群，此时已和zk保持TCP长连接
4. 添加一个watch，当连接建立后，会执行 recover方法
这个watch中对连接建立后的异步处理是：
zk连接异步建立，可能需要很长时间（dubbo服务已经开始发布和订阅【dubbo与zk建立连接需要时间】，甚至发布和订阅已经执行完了，但是zk集群竟然都没启动），这时候对之前的服务需要重新发布与订阅，但是这个动作是交给FailbackRegistry的重试任务去执行的。

## doRegister
在zk上创建一个临时节点

## doUnregister
在zk上删除一个临时节点

## doSubscribe
即使是服务提供者，也会订阅节点configurators，因此这个方法消费者和提供者都会用到。
- 对于consumer，传入的是 RegistryDirectory 类型
- 对于provider，传入的是 OverrideListener 类型

方法的参数：url，接口；NotifyListener，其实就是watch的抽象

分析代码段
```java
// 只监听指定的分组
/**
 * provider 会监听 /dubbo/api/configurators 节点
 * consumer 会监听 /dubbo/api/下的3个节点【providers, configurators, routers】
 */
List<URL> urls = new ArrayList<URL>();
for (String path : toCategoriesPath(url)) { // path是需要监听的节点，如 /dubbo/${interfaceName}/provider
    ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
    if (listeners == null) {
        zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
        listeners = zkListeners.get(url);
    }
    ChildListener zkListener = listeners.get(listener);
    if (zkListener == null) {
        listeners.putIfAbsent(listener, new ChildListener() {
            public void childChanged(String parentPath, List<String> currentChilds) {
                ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds));
            }
        });
        zkListener = listeners.get(listener);
    }
    // 这个节点不存在的话，创建它
    zkClient.create(path, false);
    // 监听这个节点
    // 因此这个几点发生变化，真正执行的是代码是：
    // 上面的 ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds));
    List<String> children = zkClient.addChildListener(path, zkListener);
    if (children != null) {
        urls.addAll(toUrlsWithEmpty(url, path, children));
    }
}
// 对于provider和consumer，执行
notify(url, listener, urls);
```