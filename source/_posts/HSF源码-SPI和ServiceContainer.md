## HSF2.0Դ��ѧϰ�ʼ�-SPI��ServiceContainer

### SPI
#### API
��������������������**Ӧ�ó����̽ӿ�**������������ǵı�̻����У��Ӳ���ϵͳ�����ϵͳ���ã��ײ㺯���⣬JDK��Ӧ�ÿ�ܣ��ٵ���Ⱥͨ�Žӿڡ�����ʹ��API��ʵ�����������Ĺ��ܣ������Ӳ�̶�ȡһ���ļ�������һ�� *HTTP* ���󣬻�����Ⱥ�е�ĳ̨������ѯ��������״̬��**API���ṩ���ǹ̶��ģ������ǿ�Ԥ��ģ���һ��**��

#### SPI
> Service Provider Interface (SPI) is an API intended to be implemented or extended by a third party. It can be used to enable framework extension and replaceable components.
>                                                       --from wikipedia
                                                                                                              
�������˵��SPI������һ����׼��
��Java�������׼���涨���������Ҫ�ṩ��Щ��Ҫ�����ԣ���������ôȥʵ�֣���ʲô���ԣ���ʲô�㷨ȥGC���������ܡ�SPIҲһ������ֻ��������ӿ�Ҫ�ṩʲô����**˭���ṩ��who��**��**��ô�ṩ��how��**��SPI�ǲ�����ġ�

#### SPI�ô�
**�ӿ���ʵ�ֽ���**��
![](http://gtms03.alicdn.com/tps/i3/TB1JBXwIFXXXXXfXVXXqL.M1FXX-323-412.jpg)
��ͼ����**�������**����ʱ��*Router* ������ƽڵ���԰������ĳһ�������**�����ṩ��**�ṩ�ķ���
���**����**���п����ǵ��÷�ָ��������Ҫʹ����һ������Ҳ�п�����RouterΪ��ʵ��ĳЩϵͳ���ԣ������������Provider��


#### ServiceLoader
ServiceLoader�������Jdk�ṩ�ģ����� /META-INF/services·���µ������ļ�������ָ���ķ���ʵ���ࡣ

����һ���ӿڣ�
```
public interface Hello{
	public String sayHello();
}
```

����ʵ����
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

�� */META-INF/services* ·��������һ�������ļ����Է���ӿڵ�**ȫ�޶�����**��Ϊ�ļ������� **com.taobao.tae.service.Hello**, �ڸ��ļ��б���ָ��ʵ�����**ȫ�޶�����**:
```
com.taobao.tae.service.impl.HsfHello
com.taobao.tae.service.impl.DuboolHello
```

���������ǿ���μ���������ʵ���ࣺ
```
@Test
public void serviceLoaderTest(){
	ServiceLoader<Hello> serviceLoader = ServiceLoader.load(Hello.getClass);
	for(Hello hello : ServiceLoader){
		System.out.println(hello.sayHello());
	}
}
```
��ؼ���һ�㣬�����ļ��ͷ���ʵ�����ǿ��Է���ġ�Ҳ����˵ʵ�����������Ҫ��ʱ����Jar������ʽ�ӵ�CLASS_PATH�С�

### ServiceContainer

#### �����߼�
����˼�壬�������һ��**���������**���������úõ�**Service Provider**��
���ڲ��ṩ���ֻ��棺
* Ĭ��**Service Provider**��ʵ������
* ����**Service Provider**��Class����
* ����**Service Provider**��ʵ������
```
ConcurrentHashMap<Class<?>, Object> INSTANCE_CACHE = new ConcurrentHashMap<Class<?>, Object>(); //���������ļ��е�һ��ʵ�����ʵ��

ConcurrentHashMap<Class<?>, Map<String, Class<?>>> MAP_CLASSES_CACHE = new ConcurrentHashMap<Class<?>, Map<String, Class<?>>>(); //���������ļ��е�����ʵ�����Class����

ConcurrentHashMap<Class<?>, ConcurrentHashMap<String, Object>> MAP_INSTANCEES_CACHE = new ConcurrentHashMap<Class<?>, ConcurrentHashMap<String, Object>>(); //���������ļ�������ʵ�����ʵ������
```

�����ṩ�Ľӿڣ�
```
public static <T> T getInstance(Class<T> classType);//����Ĭ��ʵ�����ʵ��
public static <T> List<T> getInstances(Class<T> classType);//��������ʵ�����ʵ��
public static <T> T getInstance(Class<T> classType, String key);//����ָ��ʵ�����ʵ��
```

ʵ���������Ҫע��ļ��㣺
* ������
* ������

```java
instance = ServiceLoader.load(classType,
HSFServiceContainer.class.getClassLoader()).iterator().next();

//��������ʱ ֻ�����һ��ʵ��
INSTANCE_CACHE.putIfAbsent(classType, instance);
return (T) INSTANCE_CACHE.get(classType);
```

```java
//��������ʱ ֻ�����һ��ʵ��
Object newObj = clazz.newInstance();
Object provious = objMap.putIfAbsent(key, newObj);
return null == provious ? (T) newObj : (T) provious;
```
#### �ܽ�
HSFͨ��������������ķ����������λ������Spring�е�Context��
