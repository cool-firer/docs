







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



## 取余与取模

取余Remainder：商向0方向舍入。

取模modulo：商向负无穷远舍入。

如：-7 mod 4

取余：-7 mod 4 = -1 ...... -3 (node用的取余)

取模：-7 mod 4 = -2 ...... 1



再如：7 mod -4

取余：7 mod -4 = -1 ...... 3 (node用的取余)

取模：7 mod -4 = -2 ...... -1

