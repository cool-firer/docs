### 安装

npm i heapdump



#### demo

```js
const heapdump = require('heapdump');
const http = require('http');

global.leakArry = [];

var leak = function() {
  leakArry.push('leakxxx' + Math.random())
}


http.createServer(function(req, res) {
  leak();
  res.writeHead(200, {'Content-Type': 'text/plain'})
  res.end('hello World\n')
}).listen(9999);

console.log('pid:', process.pid)
```



生成heapdump文件：

kill -USR2 pid





## 内存相关

https://developer.chrome.com/docs/devtools/memory-problems/memory-101/#object-sizes



### Object size（定量对象大小）

- Shallow size（浅）

- Retained size（深）

  

### Object graph（对象图）

to understand...



### Dominator tree（支配树）

to understand...

gc root