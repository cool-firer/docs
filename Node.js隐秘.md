







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



