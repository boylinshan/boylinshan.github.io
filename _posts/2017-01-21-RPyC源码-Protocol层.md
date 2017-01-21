---
layout: post
title: 'RPyC源码-Protocol层'
category: Python
---

# RPyC源码-Protocol层
Protocol层是RPyC结构中较为重要的一层。 Protocol层制定了通信的规范并提供了通信的方式。

```python
class Connection(object):
	def __init__(self, service, channel, config = {}, _lazy = False):
	'''
	service: RPyC服务器所提供的服务。
	channel: 传递信息的通道。
	config: connection的配置文件，规定了connection的行为。
	'''
```

Connection类提供了接收请求的方法，分为serve和poll两种。

```python
	def serve(self, timeout = 1):
		data = self._recv(timeout, wait_for_lock = True)
        if not data:
            return False
        self._dispatch(data)
        return True
        
    def poll(self, timeout = 0):
        data = self._recv(timeout, wait_for_lock = False)
        if not data:
            return False
        self._dispatch(data)
        return True
```

Connection类也提供了发送请求的方法，分为同步和异步两种。

```python
	def sync_request(self, handler, *args):
        """Sends a synchronous request (waits for the reply to arrive)

        :raises: any exception that the requets may be generated
        :returns: the result of the request
        """
        seq = self._get_seq_id()
        self._send_request(seq, handler, args)
        start_time = time.time()

        while seq not in self._sync_replies:
            remaining_time = self.SYNC_REQUEST_TIMEOUT - (time.time() - start_time)
            if remaining_time < 0:
                raise socket.timeout

            # lock or wait for signal
            if self._sync_lock.acquire(False):
                self._sync_event.clear()
                try:
                    self.serve(remaining_time)
                finally:
                    self._sync_lock.release()
                    self._sync_event.set()
            else:
                self._sync_event.wait(remaining_time)

        isexc, obj = self._sync_replies.pop(seq)
        if isexc:
            raise obj
        else:
            return obj
            
  def async_request(self, handler, *args, **kwargs):
        """Send an asynchronous request (does not wait for it to finish)

        :returns: an :class:`rpyc.core.async.AsyncResult` object, which will
                  eventually hold the result (or exception)
        """
        timeout = kwargs.pop("timeout", None)
        res = AsyncResult(weakref.proxy(self))
        self._async_request(handler, args, res)
        if timeout is not None:
            res.set_expiry(timeout)
        return res
```

同步和异步请求的区别在于，发送一个sync_request后，需要等待结果返回或者timeout。而发送一个async_request后，函数则直接返回一个AsyncResult对象，AsyncResult对象在取value值时，会调用connection的server方法，获取网络传输过来的数据，也即时对于异步而言，最明显的区别在于，数据在网络中传输的这段时间程序可以继续执行别的工作。

Connection类还负责创建远程对象代理的工作，使得上层在访问的远程对象时，就像在使用本地对象一样。创建和解析对象的方法为_box和_unbox两个

```python
	def _box(self, obj):
        """store a local object in such a way that it could be recreated on
        the remote party either by-value or by-reference"""
        if brine.dumpable(obj):
            return consts.LABEL_VALUE, obj
        if type(obj) is tuple:
            return consts.LABEL_TUPLE, tuple(self._box(item) for item in obj)
        elif isinstance(obj, netref.BaseNetref) and obj.____conn__() is self:
            return consts.LABEL_LOCAL_REF, obj.____oid__
        else:
            self._local_objects.add(obj)
            try:
                cls = obj.__class__
            except Exception:
                # see issue #16
                cls = type(obj)
            if not isinstance(cls, type):
                cls = type(obj)
            return consts.LABEL_REMOTE_REF, (id(obj), cls.__name__, cls.__module__)

    def _unbox(self, package):
        """recreate a local object representation of the remote object: if the
        object is passed by value, just return it; if the object is passed by
        reference, create a netref to it"""
        label, value = package
        if label == consts.LABEL_VALUE:
            return value
        if label == consts.LABEL_TUPLE:
            return tuple(self._unbox(item) for item in value)
        if label == consts.LABEL_LOCAL_REF:
            return self._local_objects[value]
        if label == consts.LABEL_REMOTE_REF:
            oid, clsname, modname = value
            if oid in self._proxy_cache:
                return self._proxy_cache[oid]
            proxy = self._netref_factory(oid, clsname, modname)
            self._proxy_cache[oid] = proxy
            return proxy
        raise ValueError("invalid label %r" % (label,))
```

在box和unbox中，将对象分为四种VALUE, TUPLE, REMOTE_REF以及LOCAL_REF。
VALUE:值传递的对象，因此直接去传递过来的数据即可。
TUPLE:循环解析里面的数据类型。
LOCAL_REF:本地对象，即对于对端来说的REMOTE_REF对象，当对端发送一个REMOTE_REF对象时，会传递对象的本地ID，local端使用ID获取存储在本地的对象即可。
REMOTE_REF:代理对象，即当发送一个本地对象时，会告诉对端数据类型属于REMOTE_REF，此时对端将根据传送过来的数据，建议对象的代理，一个BaseNetref类。BaseNetref类中对属性的访问都将是一个网络传输，也就是说BaseNetref隐藏了获取远程数据的实际步骤，使代理表现的与本地对象一致。