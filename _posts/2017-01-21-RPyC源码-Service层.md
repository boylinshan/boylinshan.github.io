---
layout: post
title: 'RPyC源码-Service层'
category: Python
---

# RPyC源码-Service层
Service层作为RPyC的最高层，描述了服务的具体细节，即在某个Connection上，可以执行的操作有哪些，这些操作有什么作用都有Service层来规定，RPyC提供了一个Service基类，程序可以根据自己的需求编写对应的Service类。
```python
class Service(object):
    __slots__ = ["_conn"]
    ALIASES = ()

    def __init__(self, conn):
        self._conn = conn
    def on_connect(self):
        """called when the connection is established"""
        pass
    def on_disconnect(self):
        """called when the connection had already terminated for cleanup
        (must not perform any IO on the connection)"""
        pass

    def _rpyc_getattr(self, name):
        if name.startswith("exposed_"):
            name = name
        else:
            name = "exposed_" + name
        return getattr(self, name)
    def _rpyc_delattr(self, name):
        raise AttributeError("access denied")
    def _rpyc_setattr(self, name, value):
        raise AttributeError("access denied")

    @classmethod
    def get_service_aliases(cls):
        """returns a list of the aliases of this service"""
        if cls.ALIASES:
            return tuple(str(n).upper() for n in cls.ALIASES)
        name = cls.__name__.upper()
        if name.endswith("SERVICE"):
            name = name[:-7]
        return (name,)
        
    @classmethod
    def get_service_name(cls):
        """returns the canonical name of the service (which is its first 
        alias)"""
        return cls.get_service_aliases()[0]

    exposed_get_service_aliases = get_service_aliases
    exposed_get_service_name = get_service_name
```
Service类中开放了类属性的访问权限，并屏蔽了set和del操作。Service类并没有提供任何的功能函数，具体的服务内容可由继承Service类的child类自己实现。