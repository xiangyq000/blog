---
title: Python 基础编程
date: 2018-03-23 00:54:17
tags: Python
categories: [读书笔记, 编程语言]
---

## 1. 基础知识

<!--more-->

1. 双斜线整除（适用于浮点数）：

   ```python
   >>> 5.2 // 2.0
   2.0
   >>> 5 // 2
   2
   ```

2. 乘方运算符：

   ```python
   >>> 2 ** 3
   8
   ```

3. `str`：把值转换为字符串；`repr`：把值变为合法的 Python 表达式

   ```python
   >>> str(10000L)
   '10000'
   >>> print str(10000L)
   10000
   >>> repr(10000L)
   '10000L'
   >>> print repr(10000L)
   10000L
   ```

4. `input` 和 `raw_input`：`input` 假设用户输入的是合法的 Python 表达式（例如字符串需要用引号括起来），而 `raw_input` 把所有的输入当做字符串处理。

5. 长字符串：使用 `'''` 三引号：

   ```python
   >>> print '''hello
   ... world'''
   hello
   world
   ```

6. 原始字符串：

   ```python
   >>> print r'C:\nowhere'
   C:\nowhere
   >>> print r"Let's go \n\n"
   Let's go \n\n
   ```

## 2. 列表和元组

1. 分片

   ```python
   >>> nums = range(10)
   >>> nums
   [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
   >>> nums[2:7]
   [2, 3, 4, 5, 6]
   >>> nums[:]
   [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
   >>> nums[5:]
   [5, 6, 7, 8, 9]
   >>> nums[-5:-1]
   [5, 6, 7, 8]
   >>> nums[:6]
   [0, 1, 2, 3, 4, 5]
   >>> nums[-3:]
   [7, 8, 9]
   >>> nums[-3:2]
   []
   >>> nums[::3]
   [0, 3, 6, 9]
   >>> nums[9:2:-2]
   [9, 7, 5, 3]
   >>> nums[::-3]
   [9, 6, 3, 0]
   >>> nums[5::-3]
   [5, 2]
   >>> nums[:5:-3]
   [9, 6]
   >>> nums[1:5:-1]
   []
   ```

2. 序列相加：

   ```python
   >>> [1, 2, 3] + [4, 5]
   [1, 2, 3, 4, 5]
   ```

3. 序列相乘：

   ```python
   >>> [2] * 3
   [2, 2, 2]
   >>> [2, 3] * 3
   [2, 3, 2, 3, 2, 3]
   >>> [None] * 10
   [None, None, None, None, None, None, None, None, None, None]
   ```

4. `list` 函数：

   ```python
   >>> list('hello')
   ['h', 'e', 'l', 'l', 'o']
   >>> ''.join(list('hello'))
   'hello'
   ```

5. 分片赋值：

   ```python
   >>> name = list('Gil')
   >>> name[3:] = list('gamesh')
   >>> name
   ['G', 'i', 'l', 'g', 'a', 'm', 'e', 's', 'h']
   >>> name[5:5] = ['XYZ']
   >>> NAME
   Traceback (most recent call last):
     File "<stdin>", line 1, in <module>
   NameError: name 'NAME' is not defined
   >>> name
   ['G', 'i', 'l', 'g', 'a', 'XYZ', 'm', 'e', 's', 'h']
   >>> name[:] = []
   >>> name
   []
   ```

6. `append`：追加元素

7. `count`：统计某个元素出现次数

8. `extend`：在列表末尾一次性追加另一个序列中的多个值。

9. `index`：查找某个值第一个匹配项的索引位置。

10. `insert`：将对象插入到列表指定位置。

11. `pop`：移除元素（默认是最后一个）并返回该元素值。

12. `remove`：移除某个值的第一个匹配项。方法无返回值。

13. `reverse`：反向存放。

14. `sort`：原地排序。方法无返回值。可以传入排序函数参数，以及 `key` 和 `reverse` 两个关键字参数。参数 `key` 指定一个排序过程中使用的函数，用该函数为每个列表元素创建一个键，然后所有元素根据键来排序。`reverse` 提供一个布尔值，用来指明列表是否需要反向排序。

    ```python
    >>> x = [4, 1, 3, 6, 5, 2, 8]
    >>> x.append(7)
    >>> x
    [4, 1, 3, 6, 5, 2, 8, 7]
    >>> y = [7, 9]
    >>> x.extend(y)
    >>> x
    [4, 1, 3, 6, 5, 2, 8, 7, 7, 9]
    >>> x.count(7)
    2
    >>> x.index(6)
    3
    >>> x.insert(0, 10)
    >>> x
    [10, 4, 1, 3, 6, 5, 2, 8, 7, 7, 9]
    >>> x.pop()
    9
    >>> x
    [10, 4, 1, 3, 6, 5, 2, 8, 7, 7]
    >>> x.pop(1)
    4
    >>> x
    [10, 1, 3, 6, 5, 2, 8, 7, 7]
    >>> x.remove(7)
    >>> x
    [10, 1, 3, 6, 5, 2, 8, 7]
    >>> x.reverse()
    >>> x
    [7, 8, 2, 5, 6, 3, 1, 10]
    >>> y = sorted(x)
    >>> y
    [1, 2, 3, 5, 6, 7, 8, 10]
    >>> y = x[:]
    >>> y.sort()
    >>> y
    [1, 2, 3, 5, 6, 7, 8, 10]
    >>> y = x[:]
    >>> y.sort(reverse=True)
    >>> y
    [10, 8, 7, 6, 5, 3, 2, 1]
    >>> def mycmp(a, b):
    ...     return -cmp(a, b)
    ...
    >>> y = x[:]
    >>> y.sort(mycmp)
    >>> y
    [10, 8, 7, 6, 5, 3, 2, 1]
    >>> x = ['abc', 'd', 'ef']
    >>> x.sort(key=len)
    >>> x
    ['d', 'ef', 'abc']
    >>> x.reverse()
    >>> x
    ['abc', 'ef', 'd']
    ```

15. 创建元组：

    ```python
    >>> x = 1, 2, 3
    >>> x
    (1, 2, 3)
    >>> x = (1,)
    >>> x
    (1,)
    >>> x[:]
    (1,)
    ```

16. 元组的分片还是元组，就像列表的分片还是列表。

17. `tuple` 函数以一个序列作为参数并把它转换为元组：

    ```python
    >>> x = [1, 2, 3]
    >>> tuple(x)
    (1, 2, 3)
    ```

18. 元组的存在意义：

    - 在映射中作为键存在（不可变对象）
    - 作为很多内建函数和方法的返回值

## 3. 字符串

1. 字符串常量：

   ```python
   >>> import string
   >>> print string.digits
   0123456789
   >>> print string.letters
   ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz
   >>> print string.lowercase
   abcdefghijklmnopqrstuvwxyz
   >>> print string.uppercase
   ABCDEFGHIJKLMNOPQRSTUVWXYZ
   >>> print string.printable
   0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ!"#$%&'()*+,-./:;<=>?@[\]^_`{|}~
   >>> print string.punctuation
   >> !"#$%&'()*+,-./:;<=>?@[]^_`{|}~
   ```


2. `find` ：查找子串

   ```python
   >>> "My name is Gil".find('Gil')
   11
   ```

3. `replace`：替换

   ```python
   >>> "My name is Gil".replace('Gil', 'Yamato')
   'My name is Yamato'
   ```

4. `join`：

   ```phthon
   >>> '+'.join(['1', '2', '3'])
   '1+2+3'
   ```

5. `split`:

   ```python
   >>> 'hello world'.split()
   ['hello', 'world']
   >>> '1+2+3'.split('+')
   ['1', '2', '3']
   ```

6. `split`：

   ```python
   >>> 'hello world'.split()
   ['hello', 'world']
   >>> '1+2+3'.split('+')
   ['1', '2', '3']
   ```

7. `strip`：

   ```python
   >>> '      wait for stripping    '.strip()
   'wait for stripping'
   >>> '      !!!!another case !!!!    '.strip(' !')
   'another case'
   ```

## 4. 字典

1. `dict` 函数通过其他映射或者键值对的序列建立字典：

   ```python
   >>> items = [('k1', 'k2'), ('v1', 'v2')]
   >>> d = dict(items)
   >>> d
   {'v1': 'v2', 'k1': 'k2'}
   >>> items = [('k1', 'v1'), ('k2', 'v2')]
   >>> d = dict(items)
   >>> d
   {'k2': 'v2', 'k1': 'v1'}
   >>> d = dict(name = 'Gil', age = 17)
   >>> d
   {'age': 17, 'name': 'Gil'}

   # 字典的格式化字符串
   >>> "%(name)s is %(age)d years old" % d
   'Gil is 17 years old'

   >>> len(d)
   2
   ```

2. `clear` 方法清除字典中所有的项。

3. `copy` 方法返回一个具有相同键值对的新字典（浅复制，深复制需要使用内置的 `deepcopy` 函数）。

4. `fromkeys` 使用给定的键建立新的字典，每个键都对应一个默认的值 `None`。

5. `get` 方法按键取值。如果键不存在，则得到 `None`。可以在方法中传入默认值：

   ```python
   >>> d.get('none')
   >>> d.get('none', 'N/A')
   'N/A'
   ```

6. `items` 方法将字典的所有的项以列表方式返回，列表中的每一项都表示为（键，值）对的形式：

   ```python
   >>> d.items()
   [('age', 17), ('name', 'Gil')]
   ```

7. `iteritems` 类似于 `items`，但是会返回一个迭代器对象而不是列表：

   ```python
   >>> d.iteritems()
   <dictionary-itemiterator object at 0x10f3ce838>
   ```

8. `keys` 方法将字典中的键以列表形式返回，而 `iterkeys` 则返回针对键的迭代器。

9. `values` 方法以列表的形式返回字典中的值，而 `itervalues` 返回值的迭代器。与 `keys` 方法不同的是，`values` 方法返回的值的列表中可以包含重复的元素。

10. `pop` 方法用来获得对应于给定键的值，然后将这个键值对从字典中移除：

    ```python
    >>> d.pop('name')
    'Gil'
    ```

11. `popitem` 弹出一个键值对：

    ```python
    >>> d.popitem()
    ('age', 17)
    ```

12. `setdefault` 某种程度上类似于 `get`，能够获得与给定键相关联的值。除此之外，`setdefault` 还能在字典中不含有给定键的情况下设定相应的键值。当键不存在时，`setdefault` 返回默认值并且响应地更新字典。如果键存在，那么就返回与其相应的值，但不更新字典。默认值是可选的，如果不设定，则默认使用 `None`：

    ```python
    >>> d.setdefault('nobody', 'Adam')
    'Adam'
    >>> d.setdefault('nobody', 'Berserker')
    'Adam'
    >>> d.setdefault('burning')
    >>>
    ```

13. `update` 方法可以利用一个字典项更新另一个字典：

    ```python
    >>> e = {'another': 'zzz'}
    >>> d.update(e)
    >>> d
    {'nobody': 'Adam', 'another': 'zzz', 'burning': None}
    ```

14. 遍历字典：

    ```python
    >>> for k, v in d.items():
    ...     print k, v
    ...
    nobody Adam
    another zzz
    burning None
    ```

## 5. 条件、循环和其他语句

1. 序列解包：

   ```python
   >>> x, y, z = (1, 2, 3)
   >>> x
   1
   >>> x, y = y, x
   >>> x, y
   (2, 1)
   ```

   解包序列中的元素数量必须和放置在赋值符号左边的变量数量完全一致。

2. 链式赋值：

   ```python
   x = y = somefunc()

   # 相当于
   y = somefunc()
   x = y
   ```

3. 下列值在作为布尔表达式的时候，会被解释器看做假：

   ```python
   False, None, 0, "", (), [], {}s
   ```


4. `bool` 函数可以用来转换其他值：

   ```python
   >>> bool("I am Gil")
   True
   >>> bool(0)
   False
   ```


5. 相等和同一运算符：

   ```python
   >>> x = y = [1, 2, 3]
   >>> z = [1, 2, 3]
   >>> x == y
   True
   >>> x == z
   True
   >>> x is y
   True
   >>> x is z
   False
   ```


6. 序列的比较：

   ```python
   >>> "alpha" < "beta"
   True
   >>> [1, 2] < [2, 1]
   True
   >>> [1, [2, 3]] < [1, [2, 4]]
   True
   ```


7. 断言：

   ```python
   >>> age = 10
   >>> assert 0 <= age <= 100
   >>> assert age > 100
   Traceback (most recent call last):
     File "<stdin>", line 1, in <module>
   AssertionError
   ```

8. `zip` 函数：并行迭代

   ```python
   >>> names = ['anne', 'beth', 'claire']
   >>> ages = [17, 18, 19, 20]
   >>> zip(names, ages)
   [('anne', 17), ('beth', 18), ('claire', 19)]

   >>> zip(range(5), range(10), xrange(100000))
   [(0, 0, 0), (1, 1, 1), (2, 2, 2), (3, 3, 3), (4, 4, 4)]
   ```

   `zip` 可以处理不等长的序列，当最短的序列用完时就会停止。

9. `enumerage` 函数：按索引迭代

   ```python
   >>> for i, item in enumerage(a_list):
   ...     if(item == 'xxx'):
   ...             a_list[i] = 'yyy'
   ```

10. `sorted` 和 `reversed`：排序和翻转可迭代对象

    ```python
    >>> sorted('Hello, world!')
    [' ', '!', ',', 'H', 'd', 'e', 'l', 'l', 'l', 'o', 'o', 'r', 'w']
    >>> list(reversed('Hello, world!'))
    ['!', 'd', 'l', 'r', 'o', 'w', ' ', ',', 'o', 'l', 'l', 'e', 'H']
    >>>
    >>> ''.join(reversed('Hello, world!'))
    '!dlrow ,olleH'
    ```

11. 循环中的 `else` 字句：仅当循环没有被 `break` 时执行

    ```python
    >>> for i in range(0, 10):
    ...     if(i == 1):
    ...             break
    ... else:
    ...     print('did not find 1')
    ```

12. 列表推导式：

    ```python
    >>> [x ** x for x in range(1, 6)]
    [1, 4, 27, 256, 3125]
    >>> [x ** x for x in range(1, 6) if x > 3]
    [256, 3125]
    ```

13. `del` 函数：

    ```python
    >>> x = y = [1, 2]
    >>> del x
    >>> x
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    NameError: name 'x' is not defined
    >>> y
    [1, 2]
    ```

    `del` 只删除引用，而不删除对象本身（由垃圾回收清理）。

14. `exec`：

    ```python
    >>> exec "print 'Hello, world!'"
    Hello, world!

    >>> from math import sqrt
    >>> scope = {}
    >>> exec 'sqrt = 1' in scope
    >>> sqrt(4)
    2.0
    >>> scope['sqrt']
    1
    ```

15. `eval` 函数：

    ```python
    >>> scope = {}
    >>> scope['x'] = 2
    >>> exec 'y = 3' in scope
    >>> eval('x * y', scope)
    6
    ```

## 6. 抽象

1. 判断函数是否可调用：`callable`

   ```python
   >>> callable(math.sqrt)
   True
   ```

2. 如果在函数的开头写下字符串，它就会作为函数的一部分进行存储，这称为文档字符串，通过 `__doc__` 属性访问：

   ```python
   >>> math.sqrt.__doc__
   'sqrt(x)\n\nReturn the square root of x.'
   ```

3. 通过 `help` 函数获取对象的信息：

   ```python
   >>>help(math.sqrt)
   Help on built-in function sqrt in module math:

   sqrt(...)
       sqrt(x)

       Return the square root of x.
   ```

4. Python 形式上不支持函数重载。

5. 位置参数与默认参数：

   ```python
   >>> def foo(name="Gil", age=17):
   ...     print '%s is %d years old.' % (name, age)
   ...
   >>> foo()
   Gil is 17 years old.
   >>> foo("Lyn")
   Lyn is 17 years old.
   >>> foo("Allen", 20)
   Allen is 20 years old.
   >>> foo(age=25)
   Gil is 25 years old.
   >>> foo(20, "Allen")
   Traceback (most recent call last):
     File "<stdin>", line 1, in <module>
     File "<stdin>", line 2, in foo
   TypeError: %d format: a number is required, not str
   >>> foo(age=20, name="Allen")
   Allen is 20 years old.
   ```

6. 不定参数：

   ```python
   >>> def foo(argv, *params, **kw):
   ...     print argv, params, kw
   ...
   >>> foo(0, 1, 2, 3, k='v', p='q')
   0 (1, 2, 3) {'p': 'q', 'k': 'v'}
   >>> foo(0, *(1, 2, 3), **{'k':'v', 'p':'q'})
   0 (1, 2, 3) {'p': 'q', 'k': 'v'}
   >>> foo(*(1, 2, 3), **{'k':'v', 'p':'q'}, argv=0)
     File "<stdin>", line 1
       foo(*(1, 2, 3), **{'k':'v', 'p':'q'}, argv=0)
                                           ^
   SyntaxError: invalid syntax
   ```

   在和位置参数配合时可能会出问题。

7. `vars` 函数：查看局部作用域字典。

8. `globals` 函数：查看全局作用域字典。

9. 函数内部使用 `global` 声明一个变量为全局变量。

10. 闭包：

    ```python
    >>> def outermulti(factor):
    ...     def innermulti(number):
    ...             return number * factor
    ...     return innermulti
    ...
    >>> foo = outermulti(10)
    >>> foo(2.33)
    23.3
    >>> outermulti(5)(4)
    20
    ```

11. 函数式编程：

    ```python
    >>> map(str, range(10))
    ['0', '1', '2', '3', '4', '5', '6', '7', '8', '9']

    >>> seq = ["foo", "x41", "?!", "***"]
    >>> filter(lambda x : x.isalnum(), seq)
    ['foo', 'x41']
    ```

## 7. 更加抽象

1. `self` 参数事实上正是方法和函数的区别。方法（更专业一点可以称为绑定方法）将他们的第一个参数绑定到所属的实例上。

2. 为了让方法或者特性变为私有，只要在它的名字前面加上双下划线。类的内部定义中，所有以双下划线开始的名字都被”翻译“成前面加上单下划线的类名的形式：

   ```python
   >>> class A:
   ...     def __inaccessible(self):
   ...             print 'inaccessible'
   ...     def accessible(self):
   ...             print 'accessible'
   ...
   >>> a = A()
   >>> a.accessible()
   accessible
   >>> a.__inaccessible()
   Traceback (most recent call last):
     File "<stdin>", line 1, in <module>
   AttributeError: A instance has no attribute '__inaccessible'
   >>> a._A__inaccessible()
   inaccessible
   ```

3. 类的命名空间：

   ```python
   >>> class Incre:
   ...     n = 0
   ...     def __init__(self):
   ...             Incre.n += 1
   ...
   >>> m = Incre()
   >>> m.n
   1
   >>> x = Incre()
   >>> x.n
   2
   >>> m.n = 10
   >>> x.n
   2
   ```

4. 检查继承：

   ```python
   >>> class SubA(A):
   ...     pass
   ...
   >>> issubclass(SubA, A)
   True
   >>> SubA.__bases__
   (<class __main__.A at 0x10f3b5a78>,)
   >>> x = SubA()
   >>> isinstance(x, SubA)
   True
   >>> isinstance(x, A)
   True
   >>> x.__class__
   <class __main__.SubA at 0x10f3b5b48>
   ```

5. 如果一个方法从多个超类继承（多个具有相同名字的不同方法），那么先继承的类中的方法会重写后继承的类中的方法。

6. 查看对象内所有存储的值，可以使用 `__dict__` 属性：

   ```python
   >>> A.__dict__
   {'accessible': <function accessible at 0x10f3cf578>, '__module__': '__main__', '_A__inaccessible': <function __inaccessible at 0x10f3cfc80>, '__doc__': None}
   ```

## 8. 异常

```python
try:
    doSomething()
except ExceptionA, ea:
    handlerA(ea)
except (ExceptionB1, ExceptionB2), eb:
    handleB(eb)
except:
    handleOthers()
else:
    handleNoException()
finally:
    doDefaultLogic()
```

## 9. 魔法方法、属性和迭代器

1. 静态方法和类方法：

   ```python
   >>> class MyClass(object):
   ...     class_var = "class var"
   ...     @staticmethod
   ...     def smeth():
   ...              print "This is a static method"
   ...     @classmethod
   ...     def cmeth(cls):
   ...              print "This is a class method", cls, cls.class_var
   ...
   >>> MyClass.smeth()
   This is a static method
   >>> MyClass.cmeth()
   This is a class method <class '__main__.MyClass'> class var
   ```

2. `__getattribute__(self, name)`：当特性 `name` 被访问时自动调用。`__getattribute__(self, name)` 拦截所有特性的访问，也拦截对 `__dict__` 的访问。访问 `__getattribute__` 中与 `self` 有关的特性时，使用超类的 `__getattribute__` 方法（使用 `super` 函数）是唯一安全的途径。

3. `__getattr__(self, name)__`：当特性 `name` 被访问且对象没有相应的特性时被自动调用。

4. `__setattr__(self, name, value)`：当试图给特性 `name` 赋值时会被自动调用。

5. `__delattr__(self, name)`：当试图删除特性 `name` 时被自动调用。

6. 一个实现了 `__iter__` 方法的对象是可迭代的，一个实现了 `next` 方法的对象则是迭代器。`__iter__` 方法会返回一个迭代器（即自身）。

7. 生成器是一个包含 `yield` 关键字的函数。当它被调用时，在函数体中的代码不会被执行，而会返回一个迭代器。每次请求一个值，就会执行生成器中的代码，直到遇到一个 `yield` 或者 `return` 语句。`yield` 意味着应该生成一个值，而 `return` 语句意味着生成器要停止执行。

## 10. 自带电池

1. 模块在第一次导入到程序中时被执行。

2. 在主程序中，变量 `__name__` 的值是 `__main__`，而在导入的模块中，这个值就被设定为模块的名字。

3. 序列和映射是对象的集合。为了实现它们基本的行为，如果对象是不可变的，那么就需要使用两个魔法方法，如果是可变的则需要 4 个：

   - `__len__(self)`：这个方法返回集合中所含项目的数量。对于序列来说，这就是元素的个数；对于映射来说，则是键值对的数量。如果返回 0（并且没有实现重写该行为的 `__nonzero__`），对象会被当做一个布尔变量中的假值进行处理。
   - `__getitem__(self.key)`：这个方法返回与所给键对应的值。对于一个序列，键应该是一个 0～n-1的整数，n 是序列的长度；对于映射来说，可以使用任何种类的键。
   - `__setitem__(self, key, value)` ：这个方法应该按一定的方式存储和 `key` 相关的 `value`，该值随后可使用 `__getitem__` 来获取。当然，只能为可以修改的对象定义这个方法。
   - `__delitem__(self, key)`：这个方法在对一部分对象使用 `del` 语句时被调用，同时必须删除和元素相关的键。这个方法也是为可修改的对象定义的（并不是删除全部的对象，而只删除一些需要移除的元素）。

   对这些方法的附加要求如下：

   - 对于一个序列来说，如果键是负整数，那么要从末尾开始计数。换句话说，`x[-n]` 和 `[x[len(x) - n]]` 是一样的。
   - 如果键是不合适的类型，会引发一个 `TypeError` 异常。
   - 如果序列的索引是正确的类型，但超出了范围，应该引发一个 `IndexError` 异常。

4. 当模块存储在文件中时（扩展名 `.py`），包就是模块所在的目录。它必须包含一个命名为 `__init__.py` 的文件。假设要建立一个名为 drawing 的包，其中包括名为 shapes 和 colors 的模块。

   ```python
   import drawing
   import drawing.colors
   from drawing import shapes
   ```

   第一条语句中， `__init__` 模块的内容是可用的，但 shapes 和 `colors` 模块则不可用。执行第二条语句后，`colors` 模块可用了，但只能通过全名 `drawing.colors` 来使用。在执行第三条语句之后，`shapes` 模块可用，可以通过短名调用。

5. `dir()` 函数可以打印对象的所有特性（以及模块的所有函数、类、变量等）：

   ```python
   >>> dir(math)
   ['__doc__', '__file__', '__name__', '__package__', 'acos', 'acosh', 'asin', 'asinh', 'atan', 'atan2', 'atanh', 'ceil', 'copysign', 'cos', 'cosh', 'degrees', 'e', 'erf', 'erfc', 'exp', 'expm1', 'fabs', 'factorial', 'floor', 'fmod', 'frexp', 'fsum', 'gamma', 'hypot', 'isinf', 'isnan', 'ldexp', 'lgamma', 'log', 'log10', 'log1p', 'modf', 'pi', 'pow', 'radians', 'sin', 'sinh', 'sqrt', 'tan', 'tanh', 'trunc']
   ```

6. `__all__` 变量定义了模块的公有接口。准确地说，它告诉解释器：从模块导入所有名字代表什么含义。如果没有设定 `__all__`，用 `import *` 语句默认将会导入模块中所有不以下划线开头的全局名称。


## 11. 文件和流

1. `open` 函数用来打开文件：

   `open(name[, mode[, buffering]])`

2. 模式参数


   | 模式 | 描述                                                         |
   | ---- | ------------------------------------------------------------ |
   | r    | 以只读方式打开文件。文件的指针将会放在文件的开头。这是默认模式。 |
   | rb   | 以二进制格式打开一个文件用于只读。文件指针将会放在文件的开头。这是默认模式。一般用于非文本文件如图片等。 |
   | r+   | 打开一个文件用于读写。文件指针将会放在文件的开头。           |
   | rb+  | 以二进制格式打开一个文件用于读写。文件指针将会放在文件的开头。一般用于非文本文件如图片等。 |
   | w    | 打开一个文件只用于写入。如果该文件已存在则将其覆盖。如果该文件不存在，创建新文件。 |
   | wb   | 以二进制格式打开一个文件只用于写入。如果该文件已存在则将其覆盖。如果该文件不存在，创建新文件。一般用于非文本文件如图片等。 |
   | w+   | 打开一个文件用于读写。如果该文件已存在则将其覆盖。如果该文件不存在，创建新文件。 |
   | wb+  | 以二进制格式打开一个文件用于读写。如果该文件已存在则将其覆盖。如果该文件不存在，创建新文件。一般用于非文本文件如图片等。 |
   | a    | 打开一个文件用于追加。如果该文件已存在，文件指针将会放在文件的结尾。也就是说，新的内容将会被写入到已有内容之后。如果该文件不存在，创建新文件进行写入。 |
   | ab   | 以二进制格式打开一个文件用于追加。如果该文件已存在，文件指针将会放在文件的结尾。也就是说，新的内容将会被写入到已有内容之后。如果该文件不存在，创建新文件进行写入。 |
   | a+   | 打开一个文件用于读写。如果该文件已存在，文件指针将会放在文件的结尾。文件打开时会是追加模式。如果该文件不存在，创建新文件用于读写。 |
   | ab+  | 以二进制格式打开一个文件用于追加。如果该文件已存在，文件指针将会放在文件的结尾。如果该文件不存在，创建新文件用于读写。 |

3. 如果缓冲参数是 0（或者 `False`），I/O 就是无缓冲的；如果是 1 或者 `True`，I/O 就是有缓冲的。大于 1 的数字代表缓冲区的大小（单位是字节），-1 （或者其他负数）代表使用默认的缓冲区大小。

4. 下表列出了 file 对象常用的函数：


   | 序号 | 方法及描述                                                   |
   | ---- | ------------------------------------------------------------ |
   | 1    | `file.close()` 关闭文件。关闭后文件不能再进行读写操作。      |
   | 2    | `file.flush()` 刷新文件内部缓冲，直接把内部缓冲区的数据立刻写入文件，而不是被动的等待输出缓冲区写入。 |
   | 3    | `file.fileno()` 返回一个整型的文件描述符（file descriptor FD 整型），可以用在如 os 模块的 `read` 方法等一些底层操作上。 |
   | 4    | `file.isatty()` 如果文件连接到一个终端设备返回 True，否则返回 False。 |
   | 5    | `file.next()` 返回文件下一行。                               |
   | 6    | `file.read([size])` 从文件读取指定的字节数，如果未给定或为负则读取所有。 |
   | 7    | `file.readline([size])` 读取整行，包括 "\n" 字符。           |
   | 8    | `file.readlines([sizehint])` 读取所有行并返回列表，若给定 sizeint>0，则是设置一次读多少字节，这是为了减轻读取压力。 |
   | 9    | `[file.seek(offset[, whence])` 设置文件当前位置。            |
   | 10   | `file.tell()` 返回文件当前位置。                             |
   | 11   | `[file.truncate([size])` 截取文件，截取的字节通过 size 指定，默认为当前文件位置。 |
   | 12   | `file.write(str)` 将字符串写入文件，没有返回值。             |
   | 13   | `file.writelines(sequence)` 向文件写入一个序列字符串列表，如果需要换行则要自己加入每行的换行符。 |

## 14. 网络编程

1. 用 `socket()` 函数来创建套接字，语法格式如下：

   ```python
   socket.socket([family[, type[, proto]]])
   ```

   - family：套接字家族可以使 `AF_UNIX` 或者 `AF_INET`
   - type：套接字类型可以根据是面向连接的还是非连接分为 `SOCK_STREAM` 或 `SOCK_DGRAM`
   - protocol：一般不填默认为0

2. API：


   | 函数                                 | 描述                                                         |
   | ------------------------------------ | ------------------------------------------------------------ |
   | 服务器端套接字                       |                                                              |
   | s.bind()                             | 绑定地址 (host, port) 到套接字， 在 `AF_INET` 下，以元组  (host, port) 的形式表示地址。 |
   | s.listen()                           | 开始 TCP 监听。backlog 指定在拒绝连接之前，操作系统可以挂起的最大连接数量。该值至少为 1，大部分应用程序设为 5 就可以了。 |
   | s.accept()                           | 被动接受 TCP 客户端连接，（阻塞式）等待连接的到来            |
   | 客户端套接字                         |                                                              |
   | s.connect()                          | 主动初始化 TCP 服务器连接。一般 address 的格式为元组 (hostname, port)，如果连接出错，返回 `socket.error` 错误。 |
   | s.connect_ex()                       | `connect()` 函数的扩展版本，出错时返回出错码，而不是抛出异常 |
   | 公共用途的套接字函数                 |                                                              |
   | s.recv()                             | 接收 TCP 数据，数据以字符串形式返回，bufsize指定要接收的最大数据量。flag 提供有关消息的其他信息，通常可以忽略。 |
   | s.send()                             | 发送 TCP 数据，将 string 中的数据发送到连接的套接字。返回值是要发送的字节数量，该数量可能小于 string 的字节大小。 |
   | s.sendall()                          | 完整发送 TCP 数据，完整发送 TCP 数据。将 string 中的数据发送到连接的套接字，但在返回之前会尝试发送所有数据。成功返回 None，失败则抛出异常。 |
   | s.recvfrom()                         | 接收 UDP 数据，与 `recv()` 类似，但返回值是 (data, address)。其中 data 是包含接收数据的字符串，address 是发送数据的套接字地址。 |
   | s.sendto()                           | 发送 UDP 数据，将数据发送到套接字，address 是形式为 (ipaddr, port) 的元组，指定远程地址。返回值是发送的字节数。 |
   | s.close()                            | 关闭套接字                                                   |
   | s.getpeername()                      | 返回连接套接字的远程地址。返回值通常是元组 (ipaddr, port)。  |
   | s.getsockname()                      | 返回套接字自己的地址。通常是一个元组 (ipaddr, port)。        |
   | s.setsockopt(level,optname,value)    | 设置给定套接字选项的值。                                     |
   | s.getsockopt(level,optname[.buflen]) | 返回套接字选项的值。                                         |
   | s.settimeout(timeout)                | 设置套接字操作的超时期，timeout 是一个浮点数，单位是秒。值为 None 表示没有超时期。一般，超时期应该在刚创建套接字时设置，因为它们可能用于连接的操作（如 `connect()`） |
   | s.gettimeout()                       | 返回当前超时期的值，单位是秒，如果没有设置超时期，则返回None。 |
   | s.fileno()                           | 返回套接字的文件描述符。                                     |
   | s.setblocking(flag)                  | 如果 flag 为 0，则将套接字设为非阻塞模式，否则将套接字设为阻塞模式（默认值）。非阻塞模式下，如果调用 `recv()` 没有发现任何数据，或 `send()` 调用无法立即发送数据，那么将引起 `socket.error` 异常。 |
   | s.makefile()                         | 创建一个与该套接字相关连的文件                               |
