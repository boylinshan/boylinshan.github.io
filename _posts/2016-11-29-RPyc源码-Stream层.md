---
layout: post
title: 'RPyC源码-Stream层'
category: Python
---

# RPyc源码-Stream层
Stream层中最基本的是Stream类，用于提供公共接口。在Stream类大部分函数并没有实现，只是提供了接口，具体的实现方式视不同的Stream而不同。

### Stream类
```python
class Stream(object):
	"""Base Stream"""
	
    __slots__ = ()
    
	def close(self):
	'''closes the stream, release any system resource associated with it''' 
		raise NotImplementedError()
	
	@property
    def closed(self):
    """tests whether the stream is closed or not"""
        raise NotImplementedError()
        
    def poll(self, timeout):
        """indicates whether the stream has data to read (within *timeout*
        seconds)"""
        try:
            p = poll()   # from lib.compat, it may be a select object on non-Unix platforms
            p.register(self.fileno(), "r")
            while True:
                try:
                    rl = p.poll(timeout)
                except select_error:
                    ex = sys.exc_info()[1]
                    if ex.args[0] == errno.EINTR:
                        continue
                    else:
                        raise
                else:
                    break
        except ValueError:
            # if the underlying call is a select(), then the following errors may happen:
            # - "ValueError: filedescriptor cannot be a negative integer (-1)"
            # - "ValueError: filedescriptor out of range in select()"
            # let's translate them to select.error
            ex = sys.exc_info()[1]
            raise select_error(str(ex))
        return bool(rl)
        
    def read(self, count):
        """reads **exactly** *count* bytes, or raise EOFError

        :param count: the number of bytes to read

        :returns: read data
        """
        raise NotImplementedError()
        
    def write(self, data):
        """writes the entire *data*, or raise EOFError

        :param data: a string of binary data
        """
        raise NotImplementedError()	    
```
在Stream类中, 有一些值得记录的地方。
1. 使用 ____slots____ 阻止 默认____dict____的生成，与其相比，使用____slot____可以提供访问属性的速度并减少内存使用。然而值得注意的是，在多重继承中____slot____有许多限制。具体参见 [Usage of ____slots____](http://stackoverflow.com/questions/472000/usage-of-slots)。
2. @property是python内置decorator的一种，用于把对方法的调用变成对属性的调用。这样从外面看起来就像访问一个属性一样，而在类的实现中，我们可以对这个属性的访问加以限制。
3. poll用于监听多个fd。然而就目前而看，针对每一个client, server一般创建一个process或者thread，也就是说一个client其中只会对应一个socket, 这样是否有使用poll的必要？

### SocketStream类
SocketStream类是常用的stream。主要是对socket的recv和send方法作一些异常处理。
```python
class SocketStream(Stream):
    """A stream over a socket"""

    __slots__ = ("sock",)
    MAX_IO_CHUNK = 8000
    def __init__(self, sock):
        self.sock = sock
        
    @classmethod
    def _connect(cls, host, port, family = socket.AF_INET, socktype = socket.SOCK_STREAM,
            proto = 0, timeout = 3, nodelay = False, keepalive = False):
        family, socktype, proto, _, sockaddr = socket.getaddrinfo(host, port, family,
            socktype, proto)[0]
        s = socket.socket(family, socktype, proto)
        "一些对socket的设置"
        return s

    @classmethod
    def connect(cls, host, port, **kwargs):
        if kwargs.pop("ipv6", False):
            kwargs["family"] = socket.AF_INET6
        return cls(cls._connect(host, port, **kwargs))
        
   def read(self, count):
        data = []
        while count > 0:
            try:
                buf = self.sock.recv(min(self.MAX_IO_CHUNK, count))
            except socket.timeout:
                continue
            except socket.error:
                ex = sys.exc_info()[1]
                if get_exc_errno(ex) in retry_errnos:
                    # windows just has to be a bitch
                    continue
                self.close()
                raise EOFError(ex)
            if not buf:
                self.close()
                raise EOFError("connection closed by peer")
            data.append(buf)
            count -= len(buf)
        return BYTES_LITERAL("").join(data)
        
    def write(self, data):
        try:
            while data:
                count = self.sock.send(data[:self.MAX_IO_CHUNK])
                data = data[count:]
        except socket.error:
            ex = sys.exc_info()[1]
            self.close()
            raise EOFError(ex)
```
SocketStream类提供两种实例化方式，____init____方法接受socket参数，或者使用类方法connect传入host和port的信息，由connect方法创建socket并实例化SocketStream类.

MAX_IO_CHUNK: socket buffer有大小限制，分隔large data到多次recv or send 操作.

BYTES_LITERAL用于将收到的byte转化成utf8格式。
```python
def BYTES_LITERAL(text):
    return bytes(text, "utf8")
```