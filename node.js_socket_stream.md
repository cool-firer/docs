net socket与stream事件  


测试程序

tcp_server.js

```js
const net = require('net');

net.createServer(function(c) {
  console.log('conneceted');

  c.on('finish', function() {
    console.log('finish 111');
  })
  
  c.on('close', function() {
    console.log('close');
  })
  
  c.on('finish', function() {
    console.log('finish 222');
  })

  c.on('end', function() {
    console.log('end');
  });

}).listen(9988);

console.log('listen on 9988', ' pid:', process.pid)
```

  


tcp_client.js

```js
const net = require('net');

const c = net.createConnection({
  port: 9988
})

c.on('finish', function() {
  console.log('finish 111');
})

c.on('close', function() {
  console.log('close');
})

c.on('finish', function() {
  console.log('finish 222');
})
```

  


启动server，再启动cilent，ctrl +c 直接退出client，server端打印出：

```shell
$ node tcp_server.js 
listen on 9988  pid: 27157
conneceted
end
finish 111
finish 222
close

```

  


需要查socket的文档和stream的文档，再配合tcp的四次挥手理解。


  
    


socket的end事件：

https://nodejs.org/docs/latest-v10.x/api/net.html#net_event_end

> Emitted when the other end of the socket sends a FIN packet, thus ending the readable side of the socket.
>
> By default (`allowHalfOpen` is `false`) the socket will send a FIN packet back and destroy its file descriptor once it has written out its pending write queue. However, if `allowHalfOpen` is set to `true`, the socket will not automatically [`end()`](https://nodejs.org/docs/latest-v10.x/api/net.html#net_socket_end_data_encoding_callback) its writable side, allowing the user to write arbitrary amounts of data. The user must call [`end()`](https://nodejs.org/docs/latest-v10.x/api/net.html#net_socket_end_data_encoding_callback) explicitly to close the connection (i.e. sending a FIN packet back).

意思是收到了对端发来的FIN包就触发'end'事件，表示不再可读。

为了更好的理解，看一下stream的end事件：

https://nodejs.org/docs/latest-v10.x/api/stream.html#stream_event_end

> The `'end'` event is emitted when there is no more data to be consumed from the stream.

注意，只有Readable stream才有end事件。

socket是Duplex stream，可以看到socket的end与Readable stream的end意义上是对应的，表示不再有数据可读。

![socket_stream_normal](./images/socket_stream_normal.png)

所以，在1触发，最先打印出了end。

  
  


socket没有finish事件，那么只能是stream里的：

https://nodejs.org/docs/latest-v10.x/api/stream.html#stream_event_finish

> The `'finish'` event is emitted after the [`stream.end()`](https://nodejs.org/docs/latest-v10.x/api/stream.html#stream_writable_end_chunk_encoding_callback) method has been called, and all data has been flushed to the underlying system.

意思是所有的内部buffer数据都被刷到底层系统。同时注意，只有Writable stream才有finish事件。可以猜测，只有当前socket端不再可写时，才会触发，而这正是当前socket向对端发送FIN后。

![socket_stream_normal](./images/socket_stream_normal.png)

对应2，打印出finish。

  
  


之后，socket的close事件:

https://nodejs.org/docs/latest-v10.x/api/net.html#net_event_close_1

> Added in: v0.1.90
>
> - `hadError` [](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Boolean_type) `true` if the socket had a transmission error.
>
> Emitted once the socket is fully closed. The argument `hadError` is a boolean which says if the socket was closed due to a transmission error.

socket完全被关闭时触发。

同时Readable stream和Writable stream都有close事件，看一下：

Readable:

https://nodejs.org/docs/latest-v10.x/api/stream.html#stream_event_close_1

> The `'close'` event is emitted when the stream and any of its underlying resources (a file descriptor, for example) have been closed. The event indicates that no more events will be emitted, and no further computation will occur.
>
> A [`Readable`](https://nodejs.org/docs/latest-v10.x/api/stream.html#stream_class_stream_readable) stream will always emit the `'close'` event if it is created with the `emitClose` option.

  


Writable:

https://nodejs.org/docs/latest-v10.x/api/stream.html#stream_event_close

> The `'close'` event is emitted when the stream and any of its underlying resources (a file descriptor, for example) have been closed. The event indicates that no more events will be emitted, and no further computation will occur.
>
> A [`Writable`](https://nodejs.org/docs/latest-v10.x/api/stream.html#stream_class_stream_writable) stream will always emit the `'close'` event if it is created with the `emitClose` option.

Readable和Writable两种流对close事件的描述高度一致，都是说流的底层资源（文件描述符）被关闭了，这也与socket的close事件相对应。

![socket_stream_normal](./images/socket_stream_normal.png)

对应3，打印close。

  


