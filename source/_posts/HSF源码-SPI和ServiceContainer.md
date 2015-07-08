## HSF2.0源码学习笔记-SPI和ServiceContainer

### SPI
#### API
字面意义上来讲，就是**应用程序编程接口**。它充斥在我们的编程环境中：从操作系统层面的系统调用，底层函数库，JDK，应用框架，再到集群通信接口。我们使用API来实现我们期望的功能，例如从硬盘读取一个文件，发送一次 *HTTP* 请求，或者向集群中的某台机器问询它的运行状态。**API的提供者是固定的，功能是可预测的，单一的**。

#### SPI
> Service Provider Interface (SPI) is an API intended to be implemented or extended by a third party. It can be used to enable framework extension and replaceable components.
>                                                       --from wikipedia
                                                                                                              
形象的来说，SPI更像是一个标准。
像Java虚拟机标准，规定了虚拟机需要提供哪些必要的特性，第三方怎么去实现，用什么语言，用什么算法去GC，它都不管。SPI也一样，我只定义这个接口要提供什么服务，**谁来提供（who）**，**怎么提供（how）**，SPI是不在意的。

#### SPI好处
**接口与实现解耦**。
![](http://gtms03.alicdn.com/tps/i3/TB1JBXwIFXXXXXfXVXXqL.M1FXX-323-412.jpg)
如图，当**服务调用**到来时，*Router* 这个控制节点可以按需调用某一个具体的**服务提供者**提供的服务。
这个**按需**，有可能是调用方指定的需求，要使用哪一个服务；也有可能是Router为了实现某些系统策略，来决定具体的Provider。


#### ServiceLoader
ServiceLoader这个类是Jdk提供的，根据 /META-INF/services路径下的配置文件来加载指定的服务实现类。

声明一个接口：
```
public interface Hello{
	public String sayHello();
}
```

两个实现类
```
public class HsfHello{
	public String sayHello(){
		return "Hello HSF!";
	}
}

public class DubboHello{
	public String sayHello(){
		return "Hello Dubbo";
	}
}
```

在 */META-INF/services* 路径下增加一个配置文件，以服务接口的**全限定类名**作为文件名，如 **com.taobao.tae.service.Hello**, 在该文件中保存指定实现类的**全限定类名**:
```
com.taobao.tae.service.impl.HsfHello
com.taobao.tae.service.impl.DuboolHello
```

接下来我们看如何加载这两个实现类：
```
@Test
public void serviceLoaderTest(){
	ServiceLoader<Hello> serviceLoader = ServiceLoader.load(Hello.getClass);
	for(Hello hello : ServiceLoader){
		System.out.println(hello.sayHello());
	}
}
```
最关键的一点，配置文件和服务实现类是可以分离的。也就是说实现类可以在需要的时候，以Jar包的形式加到CLASS_PATH中。

### ServiceContainer

#### 主体逻辑
顾名思义，这个类是一个**服务的容器**，缓存配置好的**Service Provider**。
它内部提供三种缓存：
* 默认**Service Provider**的实例缓存
* 各类**Service Provider**的Class缓存
* 各类**Service Provider**的实例缓存
```
ConcurrentHashMap<Class<?>, Object> INSTANCE_CACHE = new ConcurrentHashMap<Class<?>, Object>(); //缓存配置文件中第一个实现类的实例

ConcurrentHashMap<Class<?>, Map<String, Class<?>>> MAP_CLASSES_CACHE = new ConcurrentHashMap<Class<?>, Map<String, Class<?>>>(); //缓存配置文件中的所有实现类的Class对象

ConcurrentHashMap<Class<?>, ConcurrentHashMap<String, Object>> MAP_INSTANCEES_CACHE = new ConcurrentHashMap<Class<?>, ConcurrentHashMap<String, Object>>(); //缓存配置文件中所有实现类的实例对象
```

对外提供的接口：
```
public static <T> T getInstance(Class<T> classType);//加载默认实现类的实例
public static <T> List<T> getInstances(Class<T> classType);//加载所有实现类的实例
public static <T> T getInstance(Class<T> classType, String key);//加载指定实现类的实例
```

实现这个缓存要注意的几点：
* 并发性
* 懒加载

```java
instance = ServiceLoader.load(classType,
HSFServiceContainer.class.getClassLoader()).iterator().next();

//并发访问时 只保存第一个实例
INSTANCE_CACHE.putIfAbsent(classType, instance);
return (T) INSTANCE_CACHE.get(classType);
```

```java
//并发访问时 只保存第一个实例
Object newObj = clazz.newInstance();
Object provious = objMap.putIfAbsent(key, newObj);
return null == provious ? (T) newObj : (T) provious;
```
#### 总结
HSF通过它来管理自身的服务组件，地位类似于Spring中的Context。
