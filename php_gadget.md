# php gadget

一般挖掘思路是找__destruct、__wakeup、__toString等方法
```
eg：
echo unserialize($data)  会触发 __toString 方法
```

## 魔术方法
```
__construct   当一个对象创建时被调用，
__toString   当一个对象被当作一个字符串被调用。
__wakeup()   使用unserialize时触发
__get()    用于从不可访问的属性读取数据
#难以访问包括：（1）私有属性，（2）没有初始化的属性
__invoke()   当脚本尝试将对象调用为函数时触发
__wakeup() //使用unserialize时触发
__sleep() //使用serialize时触发
__destruct() //对象被销毁时触发
__call() //在对象上下文中调用不可访问的方法时触发
__callStatic() //在静态上下文中调用不可访问的方法时触发
__get() //用于从不可访问的属性读取数据
__set() //用于将数据写入不可访问的属性
__isset() //在不可访问的属性上调用isset()或empty()触发
__unset() //在不可访问的属性上使用unset()时触发
__toString() //把类当作字符串使用时触发
__invoke() //当脚本尝试将对象调用为函数时触发
```

## 触发__toString
1. echo($obj) / print($obj) 将其打印出来的时候。
2. "I am {$obj}" / 'test ' . $obj 字符串连接。
3. sprintf("I am %s", $obj) 格式化字符串。
4. if($obj == 'admin') 与字符串进行==比较的时候（从此也可以印证，PHP进行==比较的时候会转换参数类型）。
5. 格式化SQL语句，绑定参数的时候会被调用。
6. in_array($obj, ['admin', 'guest'])，数组中有字符串的时候会被调用。


