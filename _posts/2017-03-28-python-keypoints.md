---
layout: post
title: "Python 语言要点"
tags: 编程语言
category: 计算机
---



#### 程序入口

```python
if __name__ == "__main__":
    # execute only if run as a script or "python -m", but not when it is imported
    main()
```

模块中的内置变量 `__name__` 保存是模块的完整名称，import 系统把它当做模块的唯一标识。

如果直接作为脚本执行或者用 `python -m module`  方式调用，它的值就是 `__main__` 。



#### 把模块当脚本执行

规范 [PEP 338 -- Executing modules as scripts](https://www.python.org/dev/peps/pep-0338/) ：

```sh
# python <options> -m <module> <args>
python -m SimpleHTTPServer 8000      # 将在当前目录映射到 Web
```



#### PEPs

Python Enhancement Proposals，即 Python 增强提议。

[PEP 0](https://www.python.org/dev/peps/) 是所有 PEPs 的一个目录。



#### 一切皆对象

```python
>>> a=1
>>> b=a             # 传递的是引用，而非数值
>>> a is b
True
>>> id(a); id(b)    # 引用相同的整数对象
140702335940872
140702335940872
```



#### 整数缓存 (Integer Cache)

Python 会缓存范围在 `[-5, 256]` 的整数，因为这个范围的整数被认为是会频繁使用的，缓存起来可以节省空间。

```python
>>> a=256
>>> b=256
>>> a is b
True
>>> c=257
>>> d=257
>>> c is d
False
>>> id(c); id(d)
140297333980944
140297333980824
```

另外，Python 解释器也会对字节码进行优化，从而使用缓存的整数：

```python
>>> e=257; f=257       # 这条语句被解释器优化了，引用了同一个对象
>>> e is f
True
```

即便是分为两行写，如果放入文件或函数中执行，一样会被当做一个整体进行优化。

缓存只对整数起作用：

```python
>>> a=(1,2); b=(1,2)
>>> a is b
False
>>> a[0] is b[0]
True
>>> a[1] is b[1]
True
```

详细分析可参考 StackOverflow 的问题：[Weird Integer Cache inside Python](http://stackoverflow.com/questions/15171695/weird-integer-cache-inside-python) 。



#### 原地交换

本质是自动封装与解封装。

```python
>>> a,b = b,a

>>> a = 1,2
>>> a
(1, 2)

>>> a,b = (1,2)
>>> print a,b
1 2
```



#### 推导式 (Comprehension)

```python
[x for x in range(0,5)]         # 列表推导式
[x for x in range(0,5) if x%2]  # [1,3] （可加条件判断）
{k:v for (k,v) in iterable}     # 字典推导式

(x for x in range(0,5))         # 返回 generator !!!
```



#### 偏函数 (Partial Function)

通过返回一个包装 (wrapper) 函数来减少传入的参数。

```python
>>> int('101', base=2)
5

>>> import functools
>>> b2i = functools.partial(int, base=2)
>>> b2i('101')
5

>>> def btoi(x, base=2):
...     return int(x, base)
... 
>>> btoi('101')
5
```



#### Lambda 表达式与函数式编程

lambda、filter()、map()、reduce()：

```python
>>> f = lambda x,y: pow(x,y)
>>> f
<function <lambda> at 0x10397c578>
>>> f(2,10)
1024

>>> filter(lambda x: x>2, range(0,5))
[3, 4]

>>> map(lambda x: x>2, range(0,5))
[False, False, False, True, True]
>>> map(lambda x: x*2, range(0,5))
[0, 2, 4, 6, 8]

>>> reduce(lambda x,y: x+y, range(0,5))     # 0+1+2+3+4=10
10
```



#### 静态方法 (staticmethod)、类方法 (classmethod)、普通方法

静态方法和类方法从调用上来说没区别，都可以直接用类调用 `Cls.foo()` 或者用实例调用 `obj.foo()`。

区别在于类方法的第一个参数必须是类，一般用 `cls`，它是个隐性参数，调用时由解释器传入。

所以，如果该方法需要访问它所在的类，就需要使用类方法，否则用静态方法即可。

普通方法的第一个参数必须是实例，一般用 `self`，它也是隐性参数，调用时自动传入。

```python
class Test(object):
    @staticmethod
    def foo1():
        pass
    # 或者： foo1 = staticmethod(foo1)
    
    @classmethod
    def foo2(cls):
        pass
    # 或者：foo2 = classmethod(foo2)
    
    def foo3(self):
        pass
```



#### 类变量

```python
class Test(object):
    __a = "class variable"
    __b = None
    def foo(self):
        Test.__a = "changed variable"
        print Test.__a                       # changed variable
        del Test.__a
        print hasattr(Test, "_Test__a")      # Flase
        print hasattr(Test, "_Test__b")      # True

t = Test()
t.foo()
```



#### \_\_new\_\_()、\_\_init\_\_()、\_\_del\_\_()

`__del__()` 是解构器 (destructor) 方法，在实例被释放前调用。因为 Python 有垃圾回收机制，所以这个方法要等到实例对象的所有引用都被清除以后才会调用。也就是说要删除掉所有引用后才会被执行且仅执行一次。

`__new__()` 是真正的构造器，它的第一个参数是类，且必须返回实例对象，然后 `__init__()` 才被调用，它的一个参数是实例，无返回值。

```python
class RoundFloat(float):
    def __new__(cls, val):
        print "Create an object."
        return float.__new__(cls, round(val, 2))
    
    def __init__(self, val):
        super(RoundFloat, self).__init__(val)      # 在子类中调用，该语句应放在第一行
        print "Do something here."
-----------------------
>>> RoundFloat(1.234567)
Create an object.
Do something here.
1.23
```



#### 自省

```python
dir(), vars()
getattr(), hasattr(), setattr(), delattr()
type(), isinstance(), issubclass()
super()          # 必须是新式类
------------
dir(obj)  # 显示实例变量和继承过来的所有方法和属性
dir(cls)  # 显示类和基类的 __dict__ 内容
dir(mod)  # 显示模块的 __dict__ 内容
dir()     # 显示调用者的局部变量
vars(arg) # 返回参数对象中的 __dict__ 内容，如果没有该属性，就会引发 TypeError 异常
vars()    # 返回本地名字空间中的属性和值，等于调用 locals()
```



#### 下划线

```python
__foo__      # 内部定义。
__foo        # 解析成 _ClassName__foo，防止与父类名字空间冲突。
_foo         # 私有，import * 无法导入。
```



#### 字符串的格式化

```python
"%s+%s=%s" % (1,1,2)
"{0}+{1}={2}".format(1,1,2)
"{a}+{b}={sum}".format(a=1,b=1,sum=2)
```

或者使用模板

```python
from string import Template; 
s = Template('There are {howmany} {what}!'); 
print s.substitute(what='people', howmany=3);
```



#### 生成器 (Generator)、迭代器 (Iterator)

迭代器：需要取下一项时，调用迭代器的 `next()` 方法即可，当全部取出后再调用就会引发 `StopIteration` 异常。

生成器：就是一个带 `yield` 语句的函数，该语句能返回一个值并暂停执行，下次调用生成器对象的 `next()` 方法时能从暂停处继续执行。

两者的区别：生成器的目的是用 yield 语句返回中间值，而迭代器的目的是遍历序列对象，但它们都是可迭代的，都可用于 for 循环。

协同程序（协程）：可以运行的独立函数调用，可以暂停或挂起，并从程序离开的地方继续或重新开始，调用者和被调用者还可以通信。简单来说就是非内核层面的、更上层的任务调度。所以，生成器就是协程。

```python
def intGen():
    for n in range(0,2):
        yield n
---------------
>>> g = intGen()    # 返回生成器
>>> g.next()
0
>>> g.next()
1
>>> g.next()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```

从 Python 2.5 开始，生成器又多了两个方法，`send()` 方法可以将值传给生成器，为了接收值，yield 语句就要像 `val = (yield count)` 这样写，另外 `close()` 方法可以终结生成器。



#### 变参 (*args、**kwargs)

位置参数：必须按照声明的顺序传进去，且数量必须一致。除非最后的几个位置参数使用了默认值，则可以不传。

非关键字可变长参数（元组）：一般声明为 `*args`，多余的位置参数会被放入元组。

关键字变量参数（字典）：一般声明为 `**kwargs`，多余的 `key=value` 参数会被放入字典。



#### 装饰器

与面向切面编程 (AOP) 是一个概念，本质就是闭包。

闭包就是一个内部函数，可以访问外部函数的变量或参数，比如下面的 log_func 函数：

```python
def log(func):
    def log_func():
        print "write func %s to log" % func.__name__
        return func()
    return log_func

@log
def foo():
    pass
--------------
>>> foo()
write func foo to log
```



#### 鸭子类型

即只要看起来像鸭子、走起来像鸭子，那就把它当鸭子。

Python 的参数不用指明类型，方法都是在调用时才绑定的，所以不管传进来的是什么对象，有相应的属性或方法就可以一样的调用。



#### 新式类、旧式类

首先，Python 2.2 开始引入新式类，新式类只能继承自 object 或者另一个新式类，为了兼容，无继承的类或者继承自旧式类的类依然是旧式类。而在 Python 3 中旧式类被彻底移除了。

旧式类的缺点是不能继承标准类型，因为在旧式类中，类型 (type) 和类 (class) 是两码事儿。新式类中将两者合并了，现在的 `int()` 等函数已经变成工厂，返回的是类实例。

```python
# 旧式类
type(obj)     # <type 'instance'>
type(0)       # <type 'int'>
type(Cls)     # <type 'class'>
type(int)     # <type 'builtin_function_or_method'>

# 新式类
type(obj)     # <class '__main__.MyClass'>
type(0)       # <type 'int'>
type(Cls)     # <type 'type'>
type(int)     # <type 'type'>
```

新式类的 `type(o)` 返回的是 `o.__class__` 的内容，而旧式类永远返回 `<type 'instance'>` 。



#### 单例模式

方法一：使用类变量保存实例的引用，没有就创建，有就直接返回该引用。

```python
class Singleton(object):
    __instance = None
    def __new__(cls,*args,**kwargs):
        if Singleton.__instance is None:
            Singleton.__instance=object.__new__(cls,*args,**kwargs)
        return Singleton.__instance

a = Singleton()
b = Singleton()

print a is b      # True
```

方法二：通过装饰器

```python
def singleton(cls, *args, **kw):  
    instances = {}  
    def _singleton():  
        if cls not in instances:  
            instances[cls] = cls(*args, **kw)  
        return instances[cls]  
    return _singleton

@singleton  
class Test(object):
    pass
```



#### GIL

Global Interpreter Lock，全局解释器锁。

多个线程在解释器内部是串行执行的，因此多线程无法利用多核来提升性能。



#### 深拷贝 (deep copy)、浅拷贝 (swallow copy)

```python
>>> import copy
>>> a = [1,2,3,[4,5]]
>>> b = copy.copy(a)         # 浅拷贝
>>> c = copy.deepcopy(a)     # 深拷贝
>>> id(a[3])
4315649880
>>> id(b[3])                 # b[3] 的只是引用了 [4,5]
4315649880
>>> id(c[3])                 # c[3] 复制了一份新的
4315651176
```



#### is 与 ==

`is` 是比较引用（即地址），`==` 是比较值。

```python
# 接着上面的例子
>>> a[3] is b[3]
True
>>> a[3] is c[3]
False
>>> a[3] == b[3]
True
>>> a[3] == c[3]
True
```



#### 读写文件

```python
str = fd.read([size])       # 返回所有或指定字节，返回空串则表示到文件结尾了
str = fd.readline([size])   # 从文件读取一行，也可以指定大小，-1 表示读到行尾，会包含换行符
lst = fd.readlines()        # 返回列表，包含所有行
fd.write(str)               # 把字符串写入文件
fd.writelines(list)         # 将字符串列表写入文件，要自己给每行加换行符
```

迭代文件中每一行更好的方法：

```python
for eachLine in f:
    # ...
```



#### with 语句

```python
with open("/etc/passwd", 'r') as f:
    # ...
```

不用自己写异常捕捉语句。执行 with 语句时就会获得一个上下文管理器，通过它的 `__context__()` 方法创建一个上下文对象，然后调用它的 `__enter__()` 方法，接着执行 `with ... as` 之间的语句，最后再调用 `__exit__()` 方法。

所以，分配&释放资源、异常处理都在这两个方法里面实现了。

目前支持 with 语句的除了 file 对象，还有 threading 模块中的锁、条件变量、信号量等对象。



#### 内存管理与垃圾回收

内存管理：当创建对象并将引用赋值给变量时，该对象的引用计数会加 1。对象引用被销毁时引用计数减 1。函数运行结束，所有局部变量被销毁。当变量被赋值给另外一个对象时，原对象引用计数减 1。del 可以删除一个变量，即删除一个引用。

垃圾收集：引用计数为 0 的对象占用的空间会被释放掉。



#### .pyc 与 .pyo

`.pyc` 是编译 (compile) 后的字节码文件，`.pyo` 是优化 (optimize) 后的字节码文件。

```sh
python -m py_compile file.py       # .pyc
python -O -m py_compile file.py    # .pyo
python -m compileall /dir          # 编译目录下的所有 .py 文件
```



#### \_\_builtins\_\_、\_\_builtin\_\_、builtins

內建模块在 Python2 中叫 `__builtin__`，在 Python3 中叫 `builtins`。

而 `__builtins__` 只是个引用，在主模块 (`__main__`) 中它引用的是內建模块，在非主模块中它引用的是內建模块的 `__dict__` 属性。



#### easy_install、pip

easy_install 是一个包管理器，在 setuptools 包中，可以从 PyPI 安装其它软件包。setuptools 还包含一些打包工具。

pip (2008) 是 easy_install (2004) 的替代品，功能更强大，包安装机制也不同，两者区别见[这里](https://packaging.python.org/pip_easy_install/)。

pip 在一个单独的包中，对于不同版本的 Python 应该使用对应版本的 pip，比如 pip2 或 pip3。

```sh
yum install python2-pip        # 安装 pip,pip2,pip2.7
yum install python34-pip       # 安装 pip3,pip3.4
```



#### 空语句

```python
pass    # 无操作，用来占位
```



#### range() 与 xrange()

前者返回列表，后者返回迭代器。当列表范围很大时应该用 xrange()，它不用创建完整的拷贝，效率高很多。

Python 3 已废弃 xrange()，调用 range() 将返回一个可迭代的 range 对象。



#### 持久存储

```python
import pickle
val = pickle.load(tmpfile)
pickle.dump(val, tmpfile)
```



#### MRO

Method Resolution Order，方法解析顺序。

经典类采用深度优先，新类采用广度优先。

新式类的基类都是 object，如果还用深度优先，多重继承且重写 `__init__()` 时就会调用 `object.__init__()`，而不是父类重写的那一个，也就是菱形 MRO 问题，而使用广度优先则是正确的。

查看 MRO：通过新式类的 `__mro__` 属性。



#### vars() 与 dir()

两者相似，只是 `vars()` 需要给定的对象必须有 `__dict__` 属性，然后返回它的值，没有就会报错。

当 `dir()` 不传参数时，将显示局部名字空间中的属性，即 `locals().keys()` 的值。



#### \_\_slots\_\_ 与 \_\_dict\_\_

`__dict__` 是字典，`__slots__` 是元组。

两者的作用相同，不过字典更占空间，尤其当属性数量很少且实例很多时使用 `__slots__` 更节省空间，它是元组所以不可修改，使用了 `__slots__` 以后就没有 `__dict__` 了。



#### \_\_getattribute\_\_() 与 \_\_getattr\_\_() 的区别

前者是只要访问属性就会被调用，后者是当在局部名字空间和类空间中找不到被访问的属性时才被调用。

如果属性被前者找到，后者不会被调用。

`__getattr__()` 可以用来做路由函数。



#### locals() 与 globas()

globals() 和 locals() 分别返回调用者全局或局部名称空间的字典。

在全局名称空间下，两者返回值相同。在函数内部，locals() 仅返回在函数中定义的名字。

```sh
# {'__builtins__': <module '__builtin__' (built-in)>, '__name__': '__main__', '__file__': 'test.py', '__doc__': None, '__package__': None}
print "globals: %s" % globals()
print "locals:  %s" % locals()

def test():
    a = 1 
    print "locals 1: %s" % locals()       # {'a': 1}
    b = 2
    print "locals 2: %s" % locals()       # {'a': 1, 'b': 2}

test()
```



#### 描述符 (Descriptor) 与属性 (Property)

描述符 (Descriptor) 可以理解为访问对象属性的代理，也可以将它理解为一种类型或是一个协议，它是个新式类且至少实现了三个特殊方法 `__get__()、__set__()、__delete__()` 中的一个。

当我们用 `x.foo` 的方式访问描述符类型的属性时，最终都会被 `__getattribute__()` 转化为 `type(x).__dict__['foo'].__get__(x, type(x))`。

数据描述符：实现了 `__get__()` 和 `__set__()` 方法。

方法描述符：仅实现了 `__get__()` 方法。

通过內建的 `property()` 方法能轻松创建描述符，当像属性一样使用描述符时，实际调用的是相应的方法：

```python
class C(object):
    def getx(self): return self.__x
    def setx(self, value): self.__x = value
    def delx(self): del self.__x
    x = property(getx, setx, delx, "I'm the 'x' property.")

print type(C.x)      # <type 'property'>
```

还可以用修饰符来指定描述符的方法：

```python
class C(object):
    def __init__(self):
        self._x = None

    @property
    def x(self):
        """I'm the 'x' property."""
        return self._x

    @x.setter
    def x(self, value):
        self._x = value

    @x.deleter
    def x(self):
        del self._x
```



#### \_\_init\_\_.py 的作用

`__init__.py` 文件用于表示一个目录是一个包，也就是说，包中必须包含该文件，那么目录中其它的 .py 文件自然就被当做包中的模块。该文件一般内容为空，但也有一些作用。

当导入模块时，会先执行 `__init__.py` 文件，为了控制 `from pkgname import *` 自动导入的范围，可以加入：

```python
__all__ = ['mod1', 'mod2', 'mod3']
```

为了控制 `import pkgname` 自动导入的范围，可以加入：

```python
import mod1
import mod2
import mod3
```



#### Python 2 与 Python 3 的区别

| 区别               | Python 2                    | Python 3                              |
| :--------------- | :-------------------------- | :------------------------------------ |
| print            | print "hello"               | print("hello")                        |
| xrange(),range() | xrange() 返回迭代器，range() 返回列表 | 没有 xrange()，且 range() 返回可迭代的 range 对象 |
| 新旧类              | 默认旧类，新类必须继承 object          | 默认新类，没有旧类                             |
| MRO              | 深度优先                        | 广度优先                                  |
| bytes()          | 数字转字符串，bytes(2) => '2'      | 工厂方法，bytes(2) => b'\x00\x00'          |
| 函数注释             | 无                           | 有                                     |
| 默认编码             | ASCII                       | utf-8                                 |
| 除法               | 3/2 => 1                    | 3/2 => 1.5                            |
| 不等于              | 可用 <> 或者 !=                 | 只有 !=                                 |
| 输入               | 输入用 raw_input()，求值用 input() | raw_input 更名为 input()                 |
| 字符串              | 有 str 和 unicode 两种类型        | 只有 str 类                              |
| type 与 class     | 都有。 type(2) => type 'int'   | 只有 class。 type(2) => class 'int'      |
| zip()            | 返回列表                        | 返回 zip 对象                             |



#### Python 2 中的编码问题

编码涉多个方面：I/O 编码、内部编码、源码文件的编码。

* 源文件的编码

  下面的 test.py 包含了非 ASCII 字符，编辑器默认会保存为 utf-8 文本格式，这时应该在首行告知解释器源码文件的编码格式，否则会报错。在 Python 3 中则不需要。

  test.py：

  ```python
  # -*- coding: utf-8 -*-
  print "你好"
  ```

  文件格式：

  ```sh
  $ file test.py 
  test.py: UTF-8 Unicode text
  ```

  详细规范参考 [PEP 263](https://www.python.org/dev/peps/pep-0263/) 。



* 内部编码

  ```python
  >>> type('abc')
  <type 'str'>
  >>> type(u'abc')         # 用 u 或 U 将字符串转为 unicode 对象
  <type 'unicode'>
  >>> print ur'a\nb'       # 使用原始字符串时，r 必须在 u 后面
  a\nb
  ```

  Python 3 的字符串为 unicode 编码，移除了 unicode 对象：

  ```python
  >>> type('abc')
  <class 'str'>
  >>> type(u'abc')
  <class 'str'>
  ```



* I/O 编码

  输出 GBK 编码的文件：

  ```python
  # coding:utf8
  with open("tmp.txt", 'wb') as f:
      f.write(u'你好'.encode('gbk'))      # 非 ASCII 字符必须用 Unicode 对象转码，否则会报错
  ```

  文件格式：

  ```sh
  $ file tmp.txt 
  tmp.txt: ISO-8859 text, with no line terminators
  ```

  读取 GBK 文件并打印：

  ```python
  with open("tmp.txt", 'rb') as f:
      print f.read().decode('gbk')       # 必须用相同的方式解码，否则会显示乱码
  ```

  以上的 encode()/decode() 方法实际是将字符串转为了字节串。

  在 Python 3 中，可用相同的方式读写 GBK 文件，也可以在 open() 中指定编码。




#### zip()

```python
>>> zip('123', 'abc')
[('1', 'a'), ('2', 'b'), ('3', 'c')]
>>> zip(*zip('123', 'abc'))             # 参数解包
[('1', '2', '3'), ('a', 'b', 'c')]
```



#### \`obj\` 、repr(obj)、str(obj)

repr(obj) 和 \`obj\` 完全相同。

`repr()` 对 Python 友好，一般可通过求值运算 (比如 `eval()`) 重新得到该对象，但并非绝对。`str()` 则是对用户友好，更适合 print 语句。两者的输出分别在 `__repr__()` 和 `__str__()` 中定义。

Python 3 中取消了 \`obj\` 的写法。



#### ~num 等于多少？

`~num = -(num+1)`，即 ~45 = -46。



#### type(('123')) 与 type(('123',))

前者是字符串，后者是元组



#### 安装软件包

```sh
pip install 'SomeProject'           # 安装到 /usr/local/lib/python2.7/site-packages
pip install 'SomeProject==1.4'
pip install --user SomeProject      # 安装到 ~/Library/Python/2.7/lib/python/site-packages/
pip install -r requirements.txt     # 安装依赖包
pip install -e git+https://git.repo/some_pkg.git#egg=SomeProject          # from git
pip install -e git+https://git.repo/some_pkg.git@feature#egg=SomeProject  # from a branch
pip install --index-url http://my.package.repo/simple/ SomeProject        # other index
pip install --extra-index-url http://my.package.repo/simple SomeProject
pip install <path>                  # from source
pip install -e <path>               # 可编辑
pip install ./downloads/SomeProject-1.0.4.tar.gz      # from local archive
pip install --pre SomeProject       # 安装开发版，默认只装稳定版
```



#### 多重赋值、多元赋值、链式赋值、增量赋值

多重赋值：x=y=z=1

多元赋值：x,y,z=1,2,'a string' 或者 (x,y,z)=(1,2,'a string')

链式赋值：y=x=x+1

增量赋值：x+=1



#### 代码对象、帧对象、跟踪记录对象、省略对象

代码对象：编译过的 Python 源码片段，是可执行对象。可用 `compile()` 得到代码对象，并用 `exec` 或 `eval()` 执行。代码对象本身不包含任何环境信息，被执行时动态获得上下文。它是函数的其中一个属性。

帧对象：每次函数调用产生一个新的帧。跟踪记录对象会用到帧对象。

跟踪记录对象：当异常发生时就会创建一个跟踪记录对象，它包含针对异常的栈跟踪信息。

省略对象：用于扩展切片语法中，起记号作用，名字是 Ellpisis，布尔值为 True。



#### 长整型

Python2.2 以后，在必要时整型会自动转为长整型。Python 长整型只跟内存大小有关，与机器字长无关，而 C 语言的长整型是跟机器字长相关的。



#### 十进制浮点计算

```python
from decimal import Decimal
print Decimal('.1') + Decimal('1.0')   # 1.1
```

因为 C 语言双精度实现都遵守 IEEE 规范，只有 52 位用于底，因此只有 52 位的精度，精度太长就会被截断。



#### 数值运算

```python
coerce(1,134L)    # 转为相同类型：(1L, 134L)
pow(2,5)          # 32
abs(-1)           # 1
divmod(10,3)      # (3,1)
round(-3.4)       # 四舍五入：-3.0
```



#### 基本类型及操作

字符串：

```python
for i,c in enumerate('str'): pass            # 遍历字符串
```

字典：

```python
dict([['x',1],['y',2]])       # 从列表创建字典：{‘y’:2, 'x':1}
{}.fromkeys(('x','y'), None)  # 创建默认字典：{'y': None, 'x': None}

dict.keys(); dict.values(); dict.items();
dict.iterkeys(); dict.itervalues(); dict.iteritems();   # 返回迭代器

del dict         # 删除
del dict[key]; dict.pop(key)    # 删除元素
dict.clear();    # 清空
dict1.update(dict2)   # 合并两个字典

key [not] in dict

dict.setdefault(key,default=None)    # 如果 key 不存在就新建
dict.get(key,default=None)
dict.pop(key[,default])
```

集合：

```python
set('123')            # set(['1', '3', '2'])
frozenset('123')

set.add(t)    # 添加元素
set.remove(t)   # 删除元素 (元素不存在时会报错)
set.update(t)   # 
set.discard(t)  # 删除元素 (不会报错)
set.pop()     # 删除并返回任意一个元素
set.clear()   # 清空
set.issubset(t); set.issuperset(t)    # 子集、超集

set.union(t)                     # 并集
set.intersection(t)              # 交集
set.difference(t)                # 差集
set.symmetric_difference(t)      # 两个集合的不相交部分
```

序列：

```python
a = [1,2,3,4,5]
a*2        # [1, 2, 3, 4, 5, 1, 2, 3, 4, 5] (重复)
a[1:3]     # [2, 3] （切片slice，只取到停止位的前一个）
a[::-1]    # [5, 4, 3, 2, 1] (翻转)
a[::-2]    # [5, 3, 1] (翻转，2-step)
a[-2::-2]  # [4, 2]
a[3::-2]   # [4, 2]
[i for i in reversed(a)]     # [5, 4, 3, 2, 1] (翻转，reversed()返回迭代器)
sum(a)     # 15 (求和)
```

列表：

```python
sorted(list)      # 返回列表
reversed(list)    # 返回迭代器
list.sort()       # 原地排序
list.reverse()    # 原地翻转
sum(list)         # 返回求和值
cmp(list1,list2)  # 0相等，-1小于，1大于
list1.extend(list2)     # 等于 list1 + list2
list1 + list2     # 拼接
list.insert(0,e)
list.append(e)
del list
del list[1]
list.remove(123)
list.pop(123)
```



#### 命令行参数

`sys.argv` 是命令行参数的列表，长度为 `len(sys.argv)`。



#### 异常处理

语句格式：`try-except-else-finally`，finally 无论何时都执行。

手动触发异常：`raise`。

使用断言：`assert 1==1`

Ctrl+Z：KeyboardInterrupt

所有异常的基类：BaseException

获取异常信息：通过 `except ... as e: ...` 中的 e，或者执行 `sys.exc_info()` 返回三元组（异常类型、异常类实例、跟踪记录对象）。



#### 导入模块

```python
from a import b
from .a import b
from ..a import b
from a.zip import b
import a.b
import *
```

大小写：导入模块时默认是区分大小写的，如果想不区分就必须设置环境变量 `PYTHONCASEOK`。

查看已导入的模块：在 `sys.modules` 里，它是一个字典 `{模块名：模块地址}`。

模块搜索路径：搜索环境变量 `PYTHONPATH` 中的路径，解释器启动后它的值会保存到 `sys.path` 里面，如果无法 import 可以修改该值后再试。



#### 展开"~/path"

```python
os.path.expanduser('~/Documents')
```



#### 特殊的属性或方法

特殊属性：

```python
__name__   # 对象名称
__doc__    # 对象的文档字符串
__bases__  # 包含所有父类的元组
__dict__   # 对象的属性
__module__ # 对象定义所在的模块
__class__  # 实例对应的类
__mro__    # 新式类才有，查看MRO
__slots__  # 是个元组，可替代 __dict__，更省空间
__self__   # 方法绑定的对象
```

特殊方法：

```python
__new__()     # 构造器，必须返回要创建的对象
__init__()    # 创建对象后第一个执行的函数，可用来初始化
__del__()     # 当对象的引用计数为0时，在销毁对象前调用一次
__getattribute__()   # 所有的属性访问都会调用该方法
__getattr__()        # 当 __getattribute__() 找不到属性时就会调用它
__call__()           # 实现该方法后类的实例可以像函数一样调用。foo(args) => foo.__call__(foo, args)
```

函数的属性：

```python
func_code       # 代码对象
func_defaults   # 默认参数元组
func_globals    # 全局名称空间字典，和在内部调用 globals() 一样
func_dict       # 属性字典
func_closure    # 闭包引用的自由变量元组
```

方法的属性：

```python
im_class    # 方法定义的类
im_func     # 方法的函数对象
im_self     # 绑定的实例，非绑定方法为 None
```



#### 代码对象

生成代码对象：

```python
compile(代码字符串、保存文件名或''、代码对象类型)
```

代码对象类型有三种：可求值表达式 (eval)、单一可执行语句 (single)、可执行语句组 (exec)。

执行代码对象：

```python
eval('1+1')    # 2 （对表达式字符串求值或者执行 compile() 返回的代码对象，并返回结果）
exec obj       # 只执行代码对象，无返回值。也可以是包含 Python 代码的文件对象。
input()        # 等价于 eval(raw_input())，对用户输入求值
execfile(filename) # 与 exec f 相同，不用打开文件
```



#### 判断对象是否可调用

```python
callable(obj)
```



#### 执行 Shell 命令

1. `os.system()`，返回值是退出状态，0表示成功。

   ```python
   os.system("ls")
   ```

2. `os.popen()`，从管道文件读取命令输出。

   ```python
   f=os.popen('ls')
   data=f.readline()
   f.close()
   print data
   ```

3. `subprocess.call()`，返回命令输出，可用来替代 os.system()。

   ```python
   data=subprocess.call(('cat', '/etc/fstab'))
   ```

4. `subprocess.Popen`，从管道读取，可用来替代 os.popen()。

   ```python
   f=subprocess.Popen(('uname', '-a'), stdout=PIPE).stdout
   data=f.readline()
   print data
   ```




#### 退出程序

`sys.exit(status=0)`，它会引发 `SystemExit` 异常，该异常不算错误。

`os._exit(status)` 不执行清理就退出，而且必须指定退出状态码，不推荐使用。

想在退出程序前执行一些清理工作，可改写 `sys.exitfunc()`，如果它之前已经被定义了，就先执行老的，再添加自己的代码。

给进程发送信号：`os.kill()`



#### 正则表达式

```python
import re
m = re.match('bat|bet|bit', 'He bit me!')
if m is not None: 
    m.group()
----------------
m.group()     # 返回匹配到的字符串
m.groups()    # 返回元组，包含所有所有匹配的子组
re.findall()  # 返回列表，包含所有匹配的字符串
re.match()    # 返回匹配对象 (MatchObject) 或 None
re.search()   # 返回首次匹配的位置
re.sub()      # 查找并替换，返回替换后的字符串
re.subn()     # 查找并替换，返回元组：('替换后的字符串'，替换次数)
re.split()    # 分割字符串
```

使用原始字符串来解决正则符号和 ASCII 特殊字符之间的冲突：

```python
re.match(r'\bblow', 'blow')
```





