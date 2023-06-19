## async/await

async返回一个promise。

[await](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/await)后面接的是promise，如果不是promise，则自动包装成Promise.resolve(xx)的值，本质也是一个promise，并且需要提交到nextTick执行。

```js
'use strict';

async function func() {
  console.log('in func');
  const y = 20;
  console.log(y);

  const x = await 10;
  console.log(x);
  return y;
}

console.log(func());
console.log('ww');

```

输出：

```shell
in func
20
Promise { <pending> }
ww
10
```

<br />

## 取余与取模

取余Remainder：商向0方向舍入。

取模modulo：商向负无穷远舍入。

如：-7 mod 4

取余：-7 mod 4 = -1 ...... -3 (node用的取余)

取模：-7 mod 4 = -2 ...... 1



再如：7 mod -4

取余：7 mod -4 = -1 ...... 3 (node用的取余)

取模：7 mod -4 = -2 ...... -1

<br />

## Array.fill()的注意

初始化一个matrix，为了省代码，用这样的：

```js
const record = new Array(3).fill(new Array(4).fill(0));
```

打印出来，也确实显示了3行4列：

```js
> record = new Array(3).fill(new Array(4).fill(0));
[ [ 0, 0, 0, 0 ], [ 0, 0, 0, 0 ], [ 0, 0, 0, 0 ] ]
> record
[ [ 0, 0, 0, 0 ], [ 0, 0, 0, 0 ], [ 0, 0, 0, 0 ] ]
> 
```

但是，赋值的时候出问题了：

```js
> record[1][1] = 1
1
> record
[ [ 0, 1, 0, 0 ], [ 0, 1, 0, 0 ], [ 0, 1, 0, 0 ] ]
```

所有行的元素都变了。

看[Array.fill()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/fill)的介绍，如果fill的参数是个对象，那每个数组元素都是这个对象的引用，所以其实都是同一个对象。

还是老实用它推荐的方法吧：

```js
const arr = new Array(3);
for (let i = 0; i < arr.length; i++) {
  arr[i] = new Array(4).fill(1); // Creating an array of size 4 and filled of 1
}
```



## void 0

this.value = void 0;

void后接任何表达式都返回undef，因为在一些低版本浏览器或者局部环境内，undef值可以被改写，不是一个只读的全局关键字，所以就可能使得 a === undef 判断变量是否是undef失效，用void就可以避免这样的问题。



## Promise报错没有catch的情况

```js
'use strict';

setInterval(() => {
  console.log(Date.now());
}, 1000);

const promise = new Promise((resolve, reject) => {
  console.log(aaa);
});
```

控制台抛出这样的错误，但程序还在运行。

```shell
(node:40906) UnhandledPromiseRejectionWarning: Unhandled promise rejection. This error originated either by throwing inside of an async function without a catch block, or by rejecting a promise which was not handled with .catch(). (rejection id: 1)
(node:40906) [DEP0018] DeprecationWarning: Unhandled promise rejections are deprecated. In the future, promise rejections that are not handled will terminate the Node.js process with a non-zero exit code.
```

暂时来看，Promise未处理错误不会有事，但最好不要这样。