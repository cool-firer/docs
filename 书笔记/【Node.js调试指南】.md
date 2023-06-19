# 第三章 代码

## 3.1 Promise

promise的出现是为了解决回调地狱，最终ES6采用了Promise/A+规范。

Promise/A+规范：

1. 是个状态机，三个状态：pending、fulfilled、rejected。状态只能从pending --> fulfilled或pending --> rejected，状态转换不可逆转。
2. then方法可以被同一个promise调用多次。
3. then方法必须返回一个promise，从而实现链式调用。
4. 值穿透。



### 值穿透

传入then/catche的参数如果不是函数，则忽略该值，返回上一个promise的结果：

```js
Promise.prototype.then = function(onFulfilled, onRejected) {
  if( (!isFunction(onFulfilled) && this.state === FULFILLED) ||
      (!isFunction(onRejected) && this.state === REJECTED) ) {
   	return this; 
  }
}
```



示例代码：

```js
// 参数不是函数的情况
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve('haha');
  }, 1000);
});
promise
.then('hehe')
.then(console.log); // haha

// promis已经Fulfilled/Rejected的情况
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve('haha');
  }, 1000);
});
promise.then(() => {
  promise.then().then((res) => {
    console.log(res);  // haha
  });

  promise.catch().then((res) => {
    console.log(res);  // haha
  })
})

```



### promise代码例子

一、输出 1 2 3 4 

Promise的构造函数是同步执行的，then中的函数是异步。

```js

const promise = new Promise((resolve, reject) => {
  console.log(1);
  resolve();
  console.log(2);
});

promise.then(() => {
  console.log(3);
})

console.log(4);
```

<br />

二、报错。then/catch返回的值不能是promise本身，否则会造成死循环。

```js
const promise = Promise.resolve().then(() => {
  return promise;
});
promise.catch(console.err);

(node:41000) UnhandledPromiseRejectionWarning: TypeError: Chaining cycle detected for promise #<Promise>
    at <anonymous>
    at process._tickCallback (internal/process/next_tick.js:118:7)
    at Function.Module.runMain (module.js:692:11)
    at startup (bootstrap_node.js:194:16)
    at bootstrap_node.js:666:3
```

<br />

三、输出 end、nextTick、then、setImmediate

```js
'use strict';

process.nextTick(() => {
  console.log('nextTick');
})

Promise.resolve().then(() => {
  console.log('then');
})

setImmediate(() => {
  console.log('setImmediate');
})

console.log('end');
```

Process.nextTick和promise.then都属于microtask，而setImmdiate属于macrotask，在事件循环的check阶段执行。



## 3.6 Event Loop

概括一下：

Node底层由Libuv维护着一个I/O线程池，用来执行异步任务。主线程在CPU上运行，读取timer、i/o callbacks、poll、check各个阶段上的回调队列，一旦有已经完成的异步任务，就取出来运行。

各个阶段如下：

```js
       |             // tick
----------------
|   timer      |     // 执行setTimeout()和setInterval()中到期的callback.
----------------
       |              // 各阶段之间是tick
-------------------
|   I/O callbacks  |  // 上一轮少数的I/O callbacks被延迟到这一轮这一阶段执行.
-------------------
       |
-------------------
|   idle, prepare  |
-------------------
       |
-------------------
|      poll        |   // 1. timers的定时器到期，执行setTimeout和setInterval的callback.
-------------------    // 2. 执行poll里的I/O callback.
       |
-------------------
|      check       |   // 执行setImmediate()的callback.
-------------------
         |
------------------------
|      close callbacks  |  // 执行close事件的callback, 例如 socket.on('close', func).
------------------------
  
  
```

<br />

### 示例代码

一、输出setImmediate、setTimeout

```js
const fs = require('fs');
fs.readFile(__filename, () => {
	setTimeout(() => {
		console.log('setTimeout');
	}, 0);
	
	setImmediate(() => {
		console.log('setImmediate');
	});
});
```

setTimeout的回调注册到timer阶段，setImmediate注册到check阶段，从poll阶段出来正好是check，所以先跑immdiate。

总结：在I/O callbacks中注册的setTimeout和setImmediate，永远都是setImmediate先执行。

<br />

二、无限循环，将event loop阻塞在microtask阶段。解决方法是用setImmediate代替nextTick。

```js
setInterval(() => {
	console.log('setInterval');
), 100);

process.nextTick(function tick() {   ==>   setImmediate(function tick() {
	process.nextTick(tick);                    setImmediate(tick);
});                                        });
```

immediate内执行setImmediate时将tick注册到下一次event loop的check阶段，而不是当前的check，所以不会阻塞。







