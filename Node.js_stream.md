Node.js stream模块翻译

前文

2018年[翻译过](https://www.cnblogs.com/cool-fire/p/8403567.html)一遍，今天再来翻译一遍。

版本: Node.js v17.3.0





# Stream

源码: [lib/stream.js]( https://github.com/nodejs/node/blob/v17.3.0/lib/stream.js)

stream -- 流，是一个抽象接口，用来处理流式数据。

Node.js里有很多流对象，比如: HTTP服务的request对象、process.stdout。

可以从流里读数据、往流里写数据。所有的流都继承了[`EventEmitter`](https://nodejs.org/api/events.html#class-eventemitter)。

```js

const stream = require('stream');

```

大部分情况下，没啥必要用这个模块。



## Stream的种类

Node.js里有4种流:

- [`Writable`](https://nodejs.org/api/stream.html#class-streamwritable): 只写流，只能往这个流里写数据。比如 [`fs.createWriteStream()`](https://nodejs.org/api/fs.html#fscreatewritestreampath-options).
- [`Readable`](https://nodejs.org/api/stream.html#class-streamreadable): 只读流，只能从这个流里读数据。比如[`fs.createReadStream()`](https://nodejs.org/api/fs.html#fscreatereadstreampath-options).
- [`Duplex`](https://nodejs.org/api/stream.html#class-streamduplex): 双工流，可读可写，合并了上面两个。比如[`net.Socket`](https://nodejs.org/api/net.html#class-netsocket).
- [`Transform`](https://nodejs.org/api/stream.html#class-streamtransform): 双工流，有个勾子，在读和写的时候修改数据。比如[`zlib.createDeflate()`](https://nodejs.org/api/zlib.html#zlibcreatedeflateoptions).

此外，stream模块包含了工具函数：[`stream.pipeline()`](https://nodejs.org/api/stream.html#streampipelinesource-transforms-destination-callback)、[`stream.finished()`](https://nodejs.org/api/stream.html#streamfinishedstream-options-callback)、[`stream.Readable.from()`](https://nodejs.org/api/stream.html#streamreadablefromiterable-options)、和[`stream.addAbortSignal()`](https://nodejs.org/api/stream.html#streamaddabortsignalsignal-stream)。

### Promise式的API

v15.0.0新增

处理流可以不用callback了，用它返回的promise。这样引入：

```js
require('stream/promises')
或者
require('stream').promises
```



### 对象模式(Object mode)

Node.js API创建的所有的流只能操作string和Buffer。然而，自定义流的实现时，可能需要处理其他类型的JavaScript对象，这样的流就处于对象模式。

使用`objectMode`选项把流对象转换到对象模式。注意：如果流对象已经存了，再把它转成对象模式，是一个不安全操作。



### Buffering

[`Writable`](https://nodejs.org/api/stream.html#class-streamwritable)和[`Readable`](https://nodejs.org/api/stream.html#class-streamreadable)这两种流在内部都会有一个buffer缓冲，用来存储数据。

![rw_buffer01](./images/rw_buffer01.png)

对于实现了`Readable`接口的流，调用[`stream.push(chunk)`](https://nodejs.org/api/stream.html#readablepushchunk-encoding)方法时，就会填充内部buffer。如果流没有调用[`stream.read()`](https://nodejs.org/api/stream.html#readablereadsize)方法消费，数据就一直在内部buffer。

一旦内部buffer的数据量达到了高水位（`highWaterMark`），流就会暂停从底层资源读数据，直到数据被消费。（换句话说，流会停止调用内部[`readable._read()`](https://nodejs.org/api/stream.html#readable_readsize)方法，这个方法是用来填充内部buffer的）。

对于`Writable`流，调用[`writable.write(chunk)`](https://nodejs.org/api/stream.html#writablewritechunk-encoding-callback)时，就会填充内部buffer。当内部buffer < 高水位，返回true，否则返回false。

stream的API的一个关键目标是，把内部buffer的数据量限制在一个合理的水平，不至于内存爆炸。比如[`stream.pipe()`](https://nodejs.org/api/stream.html#readablepipedestination-options)方法，就需要协调两个流不同的速度。

高水位标志`highWaterMark`是一个阈值，而不是强制性的限制。自定义实现的流可以强制限制。

对于`Duplex`和`Transform`流，两个都实现了`Readable`和`Writable`，内部有两个buffer，读buffer和写buffer。两个buffer独立分开。比如说，[`net.Socket`](https://nodejs.org/api/net.html#class-netsocket)对象就是Duplex流，读端从socket读数据，写端写数据到socket。

内部buffer机制可能会在后面的版本改变。

哦对了，高水位标志`highWaterMark`可以在实例化流的时候传入构造函数，代表字节数量。而对于对象模式流，代表对象数量。[默认](https://github.com/nodejs/node/blob/cc0342a517279c9c52543e3a37e11da3fc6cdb36/lib/_stream_writable.js?spm=a2c6h.12873639.0.0.12ab6b6ajzlAjY#L40)是16KB和16个对象。



## API

下面的例子用流实现了一个HTTP服务

```js
const http = require('http');

const server = http.createServer((req, res) => {
  // `req` is an http.IncomingMessage, which is a readable stream.
  // `res` is an http.ServerResponse, which is a writable stream.

  let body = '';
  // Get the data as utf8 strings.
  // If an encoding is not set, Buffer objects will be received.
  req.setEncoding('utf8');

  // Readable streams emit 'data' events once a listener is added.
  req.on('data', (chunk) => {
    body += chunk;
  });

  // The 'end' event indicates that the entire body has been received.
  req.on('end', () => {
    try {
      const data = JSON.parse(body);
      // Write back something interesting to the user:
      res.write(typeof data);
      res.end();
    } catch (er) {
      // uh oh! bad json!
      res.statusCode = 400;
      return res.end(`error: ${er.message}`);
    }
  });
});

server.listen(1337);

// $ curl localhost:1337 -d "{}"
// object
// $ curl localhost:1337 -d "\"foo\""
// string
// $ curl localhost:1337 -d "not json"
// error: Unexpected token o in JSON at position 1

```

`Writable`流提供了`write()`、`end()`方法来写数据。

`Readable`流使用`EventEmitter`的API通知应用数据可读。



### Writable streams(写流)

写流是对数据目的地的抽象。

写流有：

- [HTTP requests](https://nodejs.org/api/http.html#class-httpclientrequest)
- [HTTP responses](https://nodejs.org/api/http.html#class-httpserverresponse)
- [fs write streams](https://nodejs.org/api/fs.html#class-fswritestream)
- [zlib streams](https://nodejs.org/api/zlib.html)
- [crypto streams](https://nodejs.org/api/crypto.html)
- [TCP sockets](https://nodejs.org/api/net.html#class-netsocket)
- [child process stdin](https://nodejs.org/api/child_process.html#subprocessstdin)
- [`process.stdout`](https://nodejs.org/api/process.html#processstdout), [`process.stderr`](https://nodejs.org/api/process.html#processstderr)

所有的写流都实现了[stream.Writable](https://github.com/nodejs/node/blob/v17.3.0/lib/internal/streams/writable.js)类的方法。

所有的写流都遵循下面的基本使用方式：

```js
const myStream = getWritableStreamSomehow();
myStream.write('some data');
myStream.write('some more data');
myStream.end('done writing data');
```



#### Class: `stream.Writable`

v0.9.4新增

##### **Event:** `'close'`

当流或者流底层的资源（比如文件描述符）被关闭时，触发。

close事件表明后面不会再有任何其他事件触发，也不会有任何计算。

写流总是会触发close事件。



##### **Event:** `'drain'`

v0.9.4新增

当调用[`stream.write(chunk)`](https://nodejs.org/api/stream.html#writablewritechunk-encoding-callback)返回false时（表示内部buffer到了高水位），此时将停止写入。直到buffer被消费~~且低于高水位时~~，drain事件会触发，此时正好可以恢复继续写入过程。

```js
// Write the data to the supplied writable stream one million times.
// Be attentive to back-pressure.
function writeOneMillionTimes(writer, data, encoding, callback) {
  let i = 1000000;
  write();
  function write() {
    let ok = true;
    do {
      i--;
      if (i === 0) {
        // Last time!
        writer.write(data, encoding, callback);
      } else {
        // See if we should continue, or wait.
        // Don't pass the callback, because we're not done yet.
        ok = writer.write(data, encoding);
      }
    } while (i > 0 && ok);
    if (i > 0) {
      // Had to stop early!
      // Write some more once it drains.
      writer.once('drain', write);
    }
  }
}
```



##### **Event:** `'error'`

v0.9.4新增

- [Error](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error)

写入或管道传输出错时触发，传递`Error`参数。

`error`事件触发后，流会被自动关闭。可以通过设置[`autoDestroy`](https://nodejs.org/api/stream.html#new-streamwritableoptions)参数避免自动关闭行为。

`error`事件触发后，后续的事件只能是`close`。



##### **Event:** `'finish'`

v0.9.4新增

调用[`stream.end()`](https://nodejs.org/api/stream.html#writableendchunk-encoding-callback)方法后触发，所有的数据会冲到底层系统。

```js
const writer = getWritableStreamSomehow();
for (let i = 0; i < 100; i++) {
  writer.write(`hello, #${i}!\n`);
}
writer.on('finish', () => {
  console.log('All writes are now complete.');
});
writer.end('This is the end\n');
```



##### **Event:** `'pipe'`

v0.9.4新增

-  src [<stream.Readable>](https://nodejs.org/api/stream.html#class-streamreadable) 源流

[`stream.pipe()`](https://nodejs.org/api/stream.html#readablepipedestination-options)调用后触发。加入到源流的目的列表。

```js
const writer = getWritableStreamSomehow();
const reader = getReadableStreamSomehow();
writer.on('pipe', (src) => {
  console.log('Something is piping into the writer.');
  assert.equal(src, reader);
});
reader.pipe(writer);
```



##### **Event:** `'unpipe'`

v0.9.4新增

-  src [<stream.Readable>](https://nodejs.org/api/stream.html#class-streamreadable) 源流

[`stream.unpipe()`](https://nodejs.org/api/stream.html#readableunpipedestination)调用后或者写流发生错误时触发。移除源流的目的列表。

```js
const writer = getWritableStreamSomehow();
const reader = getReadableStreamSomehow();
writer.on('unpipe', (src) => {
  console.log('Something has stopped piping into the writer.');
  assert.equal(src, reader);
});
reader.pipe(writer);
reader.unpipe(writer);
```



##### **writable.cork()**

v0.11.2新增

`writable.cork`()方法强制把写入的数据缓存到内部buffer。调用[`stream.uncork()`](https://nodejs.org/api/stream.html#writableuncork)或[`stream.end()`](https://nodejs.org/api/stream.html#writableendchunk-encoding-callback)方法后才刷到目的地。

`cork()`最初的用途是在这样的场景：快速连续的写入小块数据，cork做缓冲，不用马上把小块数据导向底层设备。

`uncork()`方法会调用`writable._writev()`方法，传入缓冲的数据。这可能防止队头阻塞问题。

使用`cork()`方法，但没有实现`writable._write()`，可能对吞吐量有不好的影响。



##### **writable.destroy([error])**

- `error` [<Error>](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error) 触发error事件时传递过去

- Returns: [<this>](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)

销毁流。可选参数error，用来触发error事件。

触发`close`事件（设置`emitClose`为`false`不触发）。

调用`destroy()`后，写流会被终止，之后再调用`write()`、`end()`会报`ERR_STREAM_DESTROYED`错误。

`destroy()`会马上销毁流，并且极其危险，因为之前的`write()`方法写入的数据可能还没被消费，并且可能会触发`ERR_STREAM_DESTROYED`错误。

当需要把缓存的数据刷到目的地，使用end()方法或者等待`drain`事件触发后再`destroy()`。

```js
const { Writable } = require('stream');

const myStream = new Writable();

const fooErr = new Error('foo error');
myStream.destroy(fooErr);
myStream.on('error', (fooErr) => console.error(fooErr.message)); // foo error
```



```js
const { Writable } = require('stream');

const myStream = new Writable();

myStream.destroy();
myStream.on('error', function wontHappen() {});
```



```js
const { Writable } = require('stream');

const myStream = new Writable();
myStream.destroy();

myStream.write('foo', (error) => console.error(error.code));
// ERR_STREAM_DESTROYED
```

一旦调用了`destroy()`，后续的调用都将是无效操作。

自定义实现的流不应该覆盖这个方法，而应该覆盖[`writable._destroy()`](https://nodejs.org/api/stream.html#writable_destroyerr-callback)。



##### **writable.destroyed**

v8.0.0新增

- <boolean>

[`writable.destroy()`](https://nodejs.org/api/stream.html#writabledestroyerror)调用后返回true。

```js
const { Writable } = require('stream');

const myStream = new Writable();

console.log(myStream.destroyed); // false
myStream.destroy();
console.log(myStream.destroyed); // true
```



##### **writable.end([chunk[, encoding]][, callback])**

- `chunk` [string]([](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#String_type)) | [Buffer](https://nodejs.org/api/buffer.html#class-buffer) | [Uint8Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array) | [any](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Data_types) 非对象模式只能写入string、Buffer、Uni8Array; 对象模式是除null的任意JavaScript值。
- `encoding` [string](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#String_type) 编码，chunk是string时有效。
- `callback` [Function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function) 流结束时回调此函数。
- Returns: [this](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)

调用`writable.end()`方法意味着后续将不会再有任何数据被写入。传入`chunk`和`encoding`参数来写入最后一块数据，之后流会被销毁。

在调用[`stream.end()`](https://nodejs.org/api/stream.html#writableendchunk-encoding-callback)方法后再调用[`stream.write()`](https://nodejs.org/api/stream.html#writablewritechunk-encoding-callback)会抛错。

```js
// Write 'hello, ' and then end with 'world!'.
const fs = require('fs');
const file = fs.createWriteStream('example.txt');
file.write('hello, ');
file.end('world!');
// Writing more now is not allowed!
```



##### **writable.setDefaultEncoding(encoding)**

- `encoding` [string](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#String_type) 默认编码
- Returns: [this](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)

设置写端的默认编码。



##### **writable.uncork()**

v0.11.2新增

 `writable.uncork()` 从 [`stream.cork()`](https://nodejs.org/api/stream.html#writablecork) 调用时缓存的数据，全部刷到目的地。

当使用[`writable.cork()`](https://nodejs.org/api/stream.html#writablecork)和`writable.uncork()`方法管理写流的缓冲数据时，最好是用`process.nextTick()`延时调用`writable.uncork()`，因为在事件循环内，允许批量的`writable.write`()操作。

```js
stream.cork();
stream.write('some ');
stream.write('data ');
process.nextTick(() => stream.uncork());
```

如果[`writable.cork()`](https://nodejs.org/api/stream.html#writablecork)方法调用了多次，那`writable.uncork()`方法也要调用相应的次数。

```js
stream.cork();
stream.write('some ');
stream.cork();
stream.write('data ');
process.nextTick(() => {
  stream.uncork();
  // The data will not be flushed until uncork() is called a second time.
  stream.uncork();
});
```



##### **writable.writable**

v11.4.0新增

- Returns: [boolean](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Boolean_type)

如果可以安全的[`writable.write()`](https://nodejs.org/api/stream.html#writablewritechunk-encoding-callback)方法，返回true。所谓的安全是指流还没被销毁、结束、抛错。



##### **writable.writableEnded**

v12.9.0新增

- Returns: [boolean](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Boolean_type)

调用了[`writable.end()`](https://nodejs.org/api/stream.html#writableendchunk-encoding-callback)后，返回true。返回true并不意味着数据被刷出到目的地，如果要判断数据是否被刷出，要用[`writable.writableFinished`](https://nodejs.org/api/stream.html#writablewritablefinished)。



##### **writable.writableCorked**

v13.2.0, v12.16.0新增

- Returns: [integer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Number_type)

[`writable.uncork()`](https://nodejs.org/api/stream.html#writableuncork)的调用次数。



##### **writable.writableFinished**

v12.6.0新增

- Returns: [boolean](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Boolean_type)

[`'finish'`](https://nodejs.org/api/stream.html#event-finish)事件被触发前设置为true。



##### **writable.writableHighWaterMark**

v9.3.0新增

- Returns: [number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Number_type)

返回`highWaterMark`值。



##### **writable.writableLength**

v9.4.0新增

- Returns: [number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Number_type)

可以被写入的字节数（或者对象数）。



##### **writable.writableNeedDrain**

v15.2.0, v14.17.0新增

- Returns: [boolean](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Boolean_type)

流buffer满的时候设置为true，且将要触发`drain`事件。



##### **writable.writableObjectMode**

v12.3.0新增

- Returns: [boolean](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Boolean_type)

是否对象模式。



##### **writable.write(chunk, [encoding], [callback])**

- `chunk` [string]([](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#String_type)) | [Buffer](https://nodejs.org/api/buffer.html#class-buffer) | [Uint8Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array) | [any](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Data_types) 非对象模式只能写入string、Buffer、Uni8Array; 对象模式是除null的任意JavaScript值。
- `encoding` [string](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#String_type) 编码，chunk是string时有效，默认是'utf8'。
- `callback` [Function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function) 这块数据被刷出时回调。
- Returns: [boolean](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Boolean_type)

`writable.write()`方法往流里写数据，如果发生错误，`callback`函数会被回调并传入`error`参数。`callback`是异步调用，并且在`error`事件触发之前调用。

返回true表示内部buffer < `highWaterMark`；返回false时，应该停止write，直到`drain`事件触发。

如果流没有drain，`write()`写入的数据将会一直被放在内部buffer，并且返回false，直到内存爆炸，进程意外终止。高内存使用会导致垃圾收集性能贼低和高RSS。对于TCP socket，如果对端没有读数据，那么这端就永远不会drain，这也算是一个远程可执行漏洞（remotely exploitable vulnerability）。

对于[`Transform`](https://nodejs.org/api/stream.html#class-streamtransform)流，这尤其是个问题。因为`Transform`不drain，会一直缓冲write进的数据，除非被导管了（piped）或者添加了`data`、`readable`任一的事件处理器。

```js
function write(data, cb) {
  if (!stream.write(data)) {
    stream.once('drain', cb);
  } else {
    process.nextTick(cb);
  }
}

// Wait for cb to be called before doing any other write.
write('hello', () => {
  console.log('Write completed, do more writes now.');
});

```

对象模式下总是会忽略`encoding`参数。



### Readable streams(读流)

数据来源的抽象。

- [HTTP responses, on the client](https://nodejs.org/api/http.html#class-httpincomingmessage)
- [HTTP requests, on the server](https://nodejs.org/api/http.html#class-httpincomingmessage)
- [fs read streams](https://nodejs.org/api/fs.html#class-fsreadstream)
- [zlib streams](https://nodejs.org/api/zlib.html)
- [crypto streams](https://nodejs.org/api/crypto.html)
- [TCP sockets](https://nodejs.org/api/net.html#class-netsocket)
- [child process stdout and stderr](https://nodejs.org/api/child_process.html#subprocessstdout)
- [process.stdin](https://nodejs.org/api/process.html#processstdin)