---
layout: post
title: 'RPyC源码-Channel层'
category: Python
---

# RPyC源码-Channel层

虽然在网络传输中TCP协议可以保证数据按照发送的顺序到达，但并不能保证一次发送和一次接受的数据完全相同，因为中间可能经历分包、合包等操作。因此socket在读取数据时，都需要给出所要读取数据的大小。而服务端在接收请求的时候无法计算出一个完成请求的大小，因此Channel层中把data的大小添加到data的头部，将数据封装成packet的形式。从而使得在上层看来，发送和接收的是一个个packet，而不是Byte Stream。

```python
class Channel(object):
    """Channel implementation.
    
    Note: In order to avoid problems with all sorts of line-buffered transports, 
    we deliberately add ``\\n`` at the end of each frame.
    """
    
    COMPRESSION_THRESHOLD = 3000
    COMPRESSION_LEVEL = 1
    FRAME_HEADER = Struct("!LB")
    FLUSHER = BYTES_LITERAL("\n") # cause any line-buffered layers below us to flush
    __slots__ = ["stream", "compress"]

    def __init__(self, stream, compress = True):
        self.stream = stream
        if not zlib:
            compress = False
        self.compress = compress
    def recv(self):
        """Receives the next packet (or *frame*) from the underlying stream.
        This method will block until the packet has been read completely
        
        :returns: string of data
        """
        header = self.stream.read(self.FRAME_HEADER.size)
        length, compressed = self.FRAME_HEADER.unpack(header)
        data = self.stream.read(length + len(self.FLUSHER))[:-len(self.FLUSHER)]
        if compressed:
            data = zlib.decompress(data)
        return data
    def send(self, data):
        """Sends the given string of data as a packet over the underlying 
        stream. Blocks until the packet has been sent.
        
        :param data: the byte string to send as a packet
        """
        if self.compress and len(data) > self.COMPRESSION_THRESHOLD:
            compressed = 1
            data = zlib.compress(data, self.COMPRESSION_LEVEL)
        else:
            compressed = 0
        header = self.FRAME_HEADER.pack(len(data), compressed)
        buf = header + data + self.FLUSHER
        self.stream.write(buf)

```

1.	Struct用于将数据压缩成2进制，并指定字节序，避免因大小端而出现错误。
2.	zlib压缩数据，减少网络通信量。
3.	源码中故意的将\n添加到每一个请求的最后，已刷新缓存区。