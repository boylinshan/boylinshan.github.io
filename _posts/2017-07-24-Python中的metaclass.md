---
layout: post
title: "Python中的metaclass"
category: Python
---

# Python中的metaclass

```python
class MetaClass(type):
	def __new__(cls, name ,bases, dict):
		print 'metaclass __new__'
		return super(MetaClass, cls).__new__(cls, name, bases, dict)

	def __init__(cls, name, bases, dict):
		print 'metaclass __init__'
		super(MetaClass, cls).__init__(name, bases, dict)
		cls._instances = {}
		
	def __call__(cls, *args, **kargs):
		print 'metaclass __callA__'
		if cls not in cls._instances:
			cls._instances[cls] = super(MetaClass, cls).__call__(*args, **kargs)
		print 'metacalss __callB__'
		return cls._instances[cls]

class Class(object):
	__metaclass__ = MetaClass

	def __new__(cls):
		print 'class __new__'
		return super(Class, cls).__new__(cls)

	def __init__(self):
		print 'class __init__'
		super(Class, self).__init__()

	def __call__(self, *args, **kargs):
		print 'class __call__'

cls = Class()
cls()

'metaclass __new__'
'metaclass __init__'
'metaclass __callA__'
'class __new__'
'class __init__'
'metaclass __callB__'
'class _call__'
```
Python中，__new__函数控制实例的创建，而__init__函数控制类的初始化，也就是说__new__中的cls的类型是类，而__init__中的self的类型则已经是创建好的实例了。__call__则是使用实例时调用的函数。
在Python中Class也是一种Object，因此我们可以控制Class的创建，也就是使用metaclass控制创建类的行为。换言之，metaclass的__init__函数里传递的self变量是创建好的Class类。
metaclass的修改会影响的所有类的实例的创建。因此一般很少对其进行修改，在此记录一下使用到的场合。
- 单例: 参见 [Python中的单例模式](https://boylinshan.github.io/python/2016/07/31/Python中的单例模式/)
- 拆分Class: 在某个Class较大时，可以根据功能将Class拆分到几个文件中，再在创建类的时候，使用__import__以及inspect模块动态的将类的方法和变量加载进来， 示例代码如下。(TODO: 测试效率)

```python

class MetaClass(type):
	member_name_list = ("A", "B")

	def __init__(self, name, bases, dic):
		super(MetaClass, self).__init__(name, bases, dic)
		member_list = []
		for m_name in CostumeMeta.member_name_list:
			import class
			add to member_list

		for inherit in member_list:
			add methods and variables to self
```