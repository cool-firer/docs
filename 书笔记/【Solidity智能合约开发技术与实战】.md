- [简介](#简介)
- [第3章 Solidity编程基础](#第3章-solidity编程基础)
  - [3.2.1 常量](#321-常量)
  - [3.2.2 变量](#322-变量)
  - [3.3 基本数据类型](#33-基本数据类型)
    - [3.3.1 字符串弄](#331-字符串弄)
    - [3.3.3 定长浮点型](#333-定长浮点型)
    - [3.3.5 地址类型](#335-地址类型)
    - [3.4.4 数组](#344-数组)
- [第5章 智能合约与函数](#第5章-智能合约与函数)
  - [5.1 智能合约编程基础](#51-智能合约编程基础)
  - [5.2 函数编程基础](#52-函数编程基础)
    - [函数修饰符](#函数修饰符)
    - [Fallback函数](#fallback函数)
  - [5.5 抽象合约、接口和继承](#55-抽象合约接口和继承)
    - [5.5.1 抽象合约](#551-抽象合约)
    - [5.5.2 接口](#552-接口)
  - [5.6 异常外理函数](#56-异常外理函数)
- [第6章 以太坊JavaScript API --- Web3.js](#第6章-以太坊javascript-api-----web3js)
  - [6.1 Web3.js概述](#61-web3js概述)
  - [6.4 智能合约编程基础](#64-智能合约编程基础)
    - [6.4.1 合约ABI](#641-合约abi)
    - [6.4.2 合约字节码](#642-合约字节码)
      - [函数选择器](#函数选择器)
      - [参数编码](#参数编码)
    - [6.4.4 JSON-RPC](#644-json-rpc)
  - [6.5 在Web3.js中与智能合约交互](#65-在web3js中与智能合约交互)
- [第7章 事件与日志](#第7章-事件与日志)
  - [7.1 事件](#71-事件)
  - [7.2 日志](#72-日志)
- [第8章 以太坊DApp开发框架Truffle](#第8章-以太坊dapp开发框架truffle)
  - [8.1.2 安装Truffle](#812-安装truffle)
  - [8.1.3 选择以太坊客户端](#813-选择以太坊客户端)
  - [Truffle配合Ganache的一些用法](#truffle配合ganache的一些用法)
  - [如何测试用例](#如何测试用例)
  - [Truffle示例项目pet-shop](#truffle示例项目pet-shop)
- [第9章 以太坊测试网络](#第9章-以太坊测试网络)
  - [infura基本用法](#infura基本用法)
  - [跳过Solidity合约，用Web3.js完成交易](#跳过solidity合约用web3js完成交易)
- [第10章 编写安全的智能合约](#第10章-编写安全的智能合约)
  - [10.1.2 从软件工程技术角度规避风险](#1012-从软件工程技术角度规避风险)
    - [1、合约发布](#1合约发布)
    - [2、自动下线机制](#2自动下线机制)
    - [3、升级智能合约的两种方法](#3升级智能合约的两种方法)
    - [4、使用熔断机制](#4使用熔断机制)
  - [10.1.3 开发文档](#1013-开发文档)
  - [10.2 常见的针对智能合约攻击](#102-常见的针对智能合约攻击)
    - [10.2.1 重入(Reentrancy)](#1021-重入reentrancy)
    - [10.2.2 抢先交易](#1022-抢先交易)
  - [10.3.2 Solidity安全问题](#1032-solidity安全问题)
  - [10.4 智能合约的安全审计](#104-智能合约的安全审计)
    - [审计报告例子](#审计报告例子)
    - [Mythril工具分析智能合约安全漏洞](#mythril工具分析智能合约安全漏洞)






<br />

# 简介

作者：李晓黎

2022年12月第一版，但其实可能2019年就写好了。。。

里面的编译版本都是0.5.x开头，现在2023年都0.8.x了。

<br />

# 第3章 Solidity编程基础

## 3.2.1 常量

常量只能在合约中定义，在函数中定义会报错。

```javascript
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.0;

contract DataLocationTest {
  int constant x = 10;
  
  function foo() public {
    // 这里会报错
    // ParserError: Expected ';' but got 'constant'
    int constant y = 10;
  }
}
```

<br />

## 3.2.2 变量

状态变量：在智能合约中定义的变量，永久存储在区块链中。

变量修饰符：可见性修饰符、存储位置修饰符。

可见性修饰定义在合约里。

3种存储位置修饰符：

* storage: 指定变量永久存在区块中，写入要Gas。
* memory：指定变量存在内存。
* calldata：指定变量为只读，只有外部函数中的参数才可以指定为calldata。

```scheme
string public storage name;
```

如果定义变量时没有指定存储位置，将默认使用下面的位置修饰符：

| 变量用法           | 默认存储位置修饰符 |
| ------------------ | ------------------ |
| 函数的返回值       | memory             |
| 函数内部的局部变量 | memory             |
| 函数的参数         | memory             |
| 全局外部状态变量   | storage            |
| 外部函数的参数     | calldata           |

<br />

## 3.3 基本数据类型

### 3.3.1 字符串弄

定义: `string a;`

长度：`uint x = bytes(s).length;`

字符串比较：没有直接比较的方法。先比较长度、再比较hash值。

字符串拼接：不能用a+b，需要读者自己实现。

> 老书新出无疑了。。。
>
> 字符串拼接在0.8.20文档里是这样的：
>
> ```javascript
> contract C {
>   string s = "Storage";
>   function f(bytes calldata bc, string memory sm, bytes16 b) public view {
>     // 拼接
>     // bytes转string
>     string memory concatString = string.concat(s, string(bc), "Literal", sm);
>     }
> }
> ```

<br />

### 3.3.3 定长浮点型

有符号定长浮点：fixedMxN，无符号：ufixedMxN。

M表示整数部分的位数，必须是8的倍数，范围8~256；

N表示小数部分位数，范围0~80；

不指定MN，默认是fixed128x19；

> 0.8.20文档里是这样说的（只列不同的点）：
>
>  ufixedMxN 和 fixedMxN 中, M表示该类型占用的位数, N表示可用的小数位数。 
>
>  ufixed 和 fixed 分别是 ufixed128x18 、fixed128x18。
>
> 
>
> 类型占用的位数跟整数部分位数，有很大不同！
>
> 老书新出？还是作者理解错误？

<br />

### 3.3.5 地址类型

定义：`address _owner`，长度20个字节。

调用transfer()和send()从当前账户中转账到address变量：

```mariadb
address变量.transfer(转账金额);
address变量.send(转账金额);
```

也可以通过call()函数实现转账，但有可能发生重入攻击。

<br />

### 3.4.4 数组

定长数组：

```javascript
uint[5] arr;
uint[5] arr =  [0,1,2,3,4]
```

<br />

变长数组：

```javascript
uint[] arr;
arr.push(3);
arr.pop();
```

<br />

# 第5章 智能合约与函数

## 5.1 智能合约编程基础

状态变量的可见性：

* public：公有变量，可以在其他合约中访问；
* private：只能在本合约中访问，其他智能合约，包括派生合约都不能；
* internal：本合约 + 派生访问，外不能；

```c#
contract Animal {
  string public name;
  uint private owner;
  uint internal age;
}
```

<br />

## 5.2 函数编程基础

```javascript
function add(uint _a, uint _b) public pure returns(int) {}
```

### 函数修饰符

* 可见性修饰符：public、private、internal、external；

  > external：用于外部调用，用EVM消息传递调用，参数是calldata；
  >
  > public：也可以在外部调用，但参数是要拷贝到memory，以支持JUMP进行内部调用；
  >
  > 一般来说，external消耗更少的Gas。

* 状态性修饰符：

  | 状态性修饰符 | 说明                                         |
  | ------------ | -------------------------------------------- |
  | pure         | 不读写区块数据，只操作存储在memory上的变量。 |
  | view         | 不写区块数据，但可以从区块上读数据。         |
  | constant     | 与view相同。                                 |

  > 0.8.x叫可变性修饰符，constant早就废弃了。
  >
  > 又是老书新书的一大罪证！

* payable修饰符：允许函数在调用时同时接收以太币，以msg.value接收：

  ```javascript
  constract PayDemo {
    address payable Account1 = 0x1232323;
    
    // payble表明调用此函数时会可以发送以太币
    function pay() payable public {
      Account1.transfer(msg.value);
    }
  }
  ```

  EIP-55格式的账户地址：0x14736AC0832bca234CDF 有大写有小写形成检验和，让地址有自校验的能力。

* 自定义的modifier：

  ```javascript
  contract ModifierDemo {
    address seller;
    
    // 定义
    modifier OnlySeller() {
      require(msg.sender == seller, "必须是卖家");
      _;
    }
    
    function Abort() public view OnlySeller {
      ...
    }
  ```

<br />

### Fallback函数

没有函数名、参数、返回值。两种情况会调用Fallback函数：

* 调用合约中不存的函数时。

  ```javascript
  /**  前置知识
  
  用address.call(函数选择器, 参数列表) 来模拟调用不存的函数;
  函数选择器 = abi.encodeWithSignature("<函数名>(参数类型列表)");
  
  */
  contract ExecuteFallback {
    function callNonExist() public returns(bool) {
      // 函数选择器
      bytes memory func = abi.encodeWithSignature("functionNotExist()");
      bool success = address(this).call(func);
      return success;
    }
    
    event FallbackCalled(bytes data);
    
    // Fallback函数
    function() external {
      emit FallbackCalled(msg.data);
    }
  ```

* 合约在接收以太币时，且加上payable时。

  一个没有定义Fallback函数的合约，如果接收以太币，会触发异常，并返回以太币。

  所谓接收以太币，就是像合约A地址= 0x123abc，在某一个合约中调用：

  合约A地址.send(1 ether)，向合约A转以太币。

  ```javascript
  contract ReceivFallback {
    
    function sendEther public {
      
      // 如果msg.sender全额没有1wei, result返回false, 不调用fallback。
      bool result = address(this).send(1);
      
      emit SendEvent(address(this), result);
      
      // 如果没有加上payable, 上面的result也是返回false。
      function() external payable {
        
      }
      
   
  ```

  

## 5.5 抽象合约、接口和继承

### 5.5.1 抽象合约

抽象合约：包含抽象函数的合约，可以包含正常的函数。

抽象函数：没有方法体的函数。

```javascript
// 抽象合约
contract Pet {
  // 抽象函数
  function cry() public returns(string memory);
  
  // 正常函数
  function run() public returns(string memory) {
    return "run"
  }
}
```

抽象合约不能部署。

### 5.5.2 接口

接口只能有抽象函数。合约可以继承接口。

```javascript
interface Pet {
  function cry(string memory) public;
}
```

<br />

## 5.6 异常外理函数

如果生成异常，所有状态变化会回退。

**assert函数：**

会烧掉所有的Gas。

```javascript
assert(msg.sender == owner);
```

<br />

**require函数：**

会退回剩下的Gas，尽量使用require。

```
require(条件表达式，返回的错误信息);
```

<br />

**revert函数：**

退回剩下Gas，标识错误。

```javascript
contract VendingMachine {
  function buy(uint amount) public payable {
    if (amount > msg.value / 2 ether)
      revert("没有支付足够的以太币");
  }
}
```



# 第6章 以太坊JavaScript API --- Web3.js

## 6.1 Web3.js概述

Web3.js是一个函数库，通信模型如下：

```
                                              RPC通信
用户 -------->   Web应用 -----------> Web.js -------------> 以太坊区块链
              (前端HTML+JS)                 <-------------

```

Web3.js分几个模块：

* web3-eth：与以太坊和智能合约相关；
* web3-shh：Whisper协议相关。Whisper协议是以太坊分工布消息协议，可以实现智能合约间的消息互通；
* web3-bzz：Swarm协议相关。Swarm是以太坊分布式存储协议；
* web3-utils：helpe函数。

<br />

小例子：

```javascript
// connect.js
var Web3 = require('web3');
var web3 = new Web3(new Web3.providers.HttpProvider('http:///192.168.1.101:8545'))
var version = web3.node.version; // 返回节点版本号
console.log(version);
```

通过Http连接到私有链节点。

为了能接收Web3.js的连接，需要在启动私链时启用RPC：

```shell
geth --datadir ethchain --nodiscover console 2 --http.addr 192.168.1.101 --http.port 8545 --http.corsdomain "*" -http --http.api="db,eth,net,web3,personal"
```

<br />

## 6.4 智能合约编程基础

### 6.4.1 合约ABI

ABI传递的是二进制格式的信息。

智能合约的ABI是一个数组，表示合约的函数或事件。

其中，函数用以下7个参数表示：

* name：函数名。

* type：函数类型，function、constructor、fallback。

* inputs：输入参数，是一个数组。

  * inputs.name：参数名，字符串；

  * inputs.type：数据类型，字符串；

  * inputs.components：输入参数是struct才会有，用来描述struct中的数据，数组；

    

* outputs：返回值，跟inputs也是一个数组。

* payable：为true表示函数可以接收以太币。

* constant：为true表示函数不会修改合约字段的状态变量。

  

* stateMutability：状态类型，字符串，可取值为。

  * pure：函数不读取区块链；
  * view：函数只查看状态变量；
  * nonpayable：函数不可以接收以太币；
  * payable：函数可以接收以太币；

以下合约：

```javascript
pragma solidity ^0.5.1;
contract Demo {
  function sum_for(uint _max) public pure returns(unit) {
    return 2;
  }
}

// 对应的ABI:
[{
  "name": "sum_for",
  "type": "function",
  "inputs": [{
      "name": "_max",
      "type": "uint256"
    }],
    "outputs": [{
      "name": "",
      "type": "uint256"
    }],
    "payable": false,
    "constant": true,
    "stateMutability": "pure"
}]
```

 <br />

### 6.4.2 合约字节码

bytecode，有两个地方用到：

1、在部署合约时用到；

2、在调用合约时用到，由函数选择器和参数编码2个部分组成；

#### 函数选择器

SHA3（Keccak256）(函数名+参数类型) 取前四字节。

```javascript
const Web3 = require('web3');
const web3 = new Web3();

// 得到0x82412b......
console.log(web3.sha3('sum_for(uint256)'));

// 再取前四字节, 再取0x82412b就是sum_for(uint256)的函数选择器。
```

<br />

#### 参数编码

以32字节(256位)为单位，不同参数的编码方式不一样。

以整形为例，正数高位补0，负数高位补1 。

合约Foo：

```javascript
pragma solidity ^0.5.1;

contract Foo {
  function baz(uint32 x, bool y) pubilc pure returns(bool r) {
    r = x > 32 || y;
  }
}
```

如果要调用baz(69, true)，生成调用合约的字节码过程：

1. baz(uint32,bool)的函数选择器，得到 0xcdcd77c0。
2. 计算69的参数编码：   0x00...0045  (32字节)。
3. 计算true的参数编码： 0x00...0001  (32字节)。
4. 拼接起来得到字节码：0xcdcd77c00x00...00450x00...0001。

可以用这个字节码通过Web3.js调用baz()函数。

<br />

### 6.4.4 JSON-RPC

Web3.js与以太坊交互的数据格式。

格式长这样：

```javascript
{
  "jsonrpc": "2.0",
  "method": "eth_coinbase",
  "params": [...],
  "id": 1, // 命令编号, 返回的结果也有一个相同值的id匹配。
}
```

上面barz(69, true)的调用，就可以用curl这样来调用：

```bash
curl -H 'Content-Type: application/json'
     --data '{
       "jsonrpc": "2.0",
       "method": "eth_call",
       "params": [{
         "from": "0xddd...",
         "to": "0x9c...", // 要调用的合约地址
         "data": "0xcdcd77c00x00...00450x00...0001", // 字节码
         }, "latest"]
       "id": 1
     }'
     localhost:8545
```

<br />

## 6.5 在Web3.js中与智能合约交互

1、连接到以太坊节点，获得Web3对象。

```javascript
var Web3 = require('web3');
var web3 = new Web3(new Web3.providers.HttpProvider(节点URL));
```

<br />

2、根据ABI、合约地址创建合约对象。

```javascript
var myContract = web3.eth.Contract(abi, [address], [options]);
```

address：合约地址。

options：{ from、gasPrice、gas、data } 部署合约的时候用的。

这样就拿到的合约对象myContract，之后就可以通过合约对象的methods对象调用合约函数。

<br />

3、调用不修改状态的pure、view函数。

合约对象.methods.函数名(函数参数).call().then(function(result, error) {});

```javascript
myContract.methods.GetVal().call().then(function(result, error) {
  ...
});
```

<br />

4、调用修改状态的函数。

合约对象.methods.函数名(函数参数).send({ from, ... }).then(function(receipt) {});

```javascript
myContract.methods.SetVal(123).send({
  from: web3.eth.coinbase,
}).then(function(receipt) {
  ...
});
```

<br />

# 第7章 事件与日志

事件记录到日志并保存在区块链上。

## 7.1 事件

Solidity事件包括定义事件、触发事件、监听事件。

前两个在合约中实现，监听事件通过Web3.js实现。

**定义事件**: event 事件名(参数)

```c#
event PersonInfoUpdate(string name, uint age);
```

<br />

**触发事件:** emit 事件名(参数)

```c#
emit PersonInfoUpdate("sss", 12);
```

<br />

**在Web3.js中监听事件**

1、首先节点要开启WS-RPC连接服务：

```shell
geth --datadir ethchain --nodiscover console 2>>1.log --http.port 8545 --http.corsdomain "*" --http.addr=192.168.1.101 -http --http.api="db,eth,net,web3,personal" 
--ws # 启用ws
--ws.addr 192.168.1.101 # ws监听地址端口
--ws.port 7777 
--ws.origins '*' # 请允许连接的源
```

<br />

2、Web3.js要用ws的方式连接：

```javascript
var web3 = new Web3(new Web3.providers.WebsocketProvider("ws://192.168.1.101:7777"));
```

<br />

3、部署以下合约：

```javascript
pragma solidity ^0.5.1;
contract Person {
  string public name;
  
  function SetInfo(string memory _name) public {
    name = _name;
    emit PersonInfoUpdate(_name);
  }
  
  event PersonInfoUpdate(string name);
}
```

<br />

4、合约对象监听事件：

合约对象.events.事件名(回调函数)。

```javascript
myContract.events.PersonInfoUpdate(function(error, event) {
  if(error) {
    console.log(error);
  } else {
    console.log(event.returnValues.name);
  }
});
```

或者这样：

```javascript
myContract.events.PersonInfoUpdate()
  .on('data', function(event) {})    // 收到新事件时
  .on('changed', function(event) {}) // 事件从区块链上移除时
  .on('error', console.error);       // 发生错误时
```

<br />

## 7.2 日志

日志是事件存储在区块链上的数据。存1字节日志消耗8Gas，存储一个32字节的合约状态变量要20000Gas。

虽然存日志消耗Gas少，但合约不能访问日志。

Solidity中可以用log0()、log1()、log2()、... logi()、函数簇直接记录日志。

logi()函数接收 i + 1个参数。

i <= bytes32。

收据中包含一些日志信息，因此可以通过web3.eth.getTransactionReceipt(交易哈希)获取收据，收据对象就有log数据。

<br />



# 第8章 以太坊DApp开发框架Truffle

## 8.1.2 安装Truffle

```shell
npm config set strict-ssl false
npm install -g truffle --registry=https://registry.npm.taobao.org
```

安装完成后，本机：

```c#
➜  work truffle version
Truffle v5.8.2 (core: 5.8.2)
Ganache v7.7.7
Solidity v0.5.16 (solc-js)
Node v14.6.0
Web3.js v1.8.2
```

## 8.1.3 选择以太坊客户端

所谓客户端，就是以太坊节点。比如官方的geth。

**开发时可以选择的以太坊客户端**

1. EthereumJs TestRPC。
2. Ganache。Truffle强推的工具，有GUI、有CLI。GUI的区块链默认使用7545端口，CLI用8545。

<br />

**正式发布时可以选择了以太坊客户端**

1. Geth：官方的。
2. Hyperledger Besu：基于Java的以太坊客户端，兼容以太坊主网。
3. Parity：快速、轻量级的以太坊客户端。
4. Nethermind：.NET Core开发的以太坊客户端。



## Truffle配合Ganache的一些用法

## 如何测试用例

## Truffle示例项目pet-shop

<br />



# 第9章 以太坊测试网络

## infura基本用法

## 跳过Solidity合约，用Web3.js完成交易

<br />

# 第10章 编写安全的智能合约

## 10.1.2 从软件工程技术角度规避风险

### 1、合约发布

要做到：

* 覆盖率100%的整体测试。
* 部署到公网的测试网络。
* beta版本部署到主网测试，限制交易金额。

### 2、自动下线机制

禁止所有操作，强制让合约下线。

```javascript
modifier isActive() {
  require(block.number <= 100000);
  _;
}

function deposit() public isActive {
  //...
}
所有函数都加上isActive
```

### 3、升级智能合约的两种方法

方法一：定义一个注册合约，保存最新版本的合约地址。

```javascript
pragma solidity ^0.5.0;

contract SomeRegister {
  address backendContract;
  address[] previous;
  address owner;
  
  constructor() {
    owner = msg.sender;
  }
  
  modifer onlyOwner() {
    require(msg.sender == owner);
    _;
  }
  
  function changeBackend(address newBackend) public onlyOwner() returns(bool) {
    if(newBackend != address(0) && newBackend != backendContract) {
      previous.push(backendContract);
      backendContract = newBackend;
      return true;
    }
    return false;
  }
}
```

<br />

方法二：通过中继合约调用。

定义中继合约，配合fallback函数，再用delegatecall：

```javascript
pragma solidity ^0.5.0;

contract Relay {
  address public currentVersion;
  address public owner;
  
  modifier onlyOwner() {
    require(msg.sender == owner);
    _;
  }
  
  constructor(address initAddr) {
    require(initAddr != address(0));
    currentVersion = initAddr;
    owner = msg.sender;
  }
  
  function changeContract(address newVersoin) public onlyOwner() {
    require(newVersion != address(0));
    currentVersion = newVersion;
  }
  
  fallback() external payable {
    (bool success, ) = address(currentVersion).delegatecall(msg.data);
    require(success);
  }
}
```

<br />

### 4、使用熔断机制

熔断：当满足某些条件时自动停止执行程序。

示例：

```javascript
bool private stopped = false;
address private owner;

modifier isAdmin() {
  require(msg.sender == owner);
  _;
}

function toggleContractActive() isAdmin public {
  stopped = !stopped;
}

modifier stopInEmergency {
  if(!stopped) _;
}

function deposit() stopInEmergency public {
  //
}
```

<br />

## 10.1.3 开发文档

包括：

1. 说明书和发布计划；
2. 当前状态；
3. 已知的问题；
4. 历史记录；
5. 故障处理过程；
6. 联系人信息；



## 10.2 常见的针对智能合约攻击

### 10.2.1 重入(Reentrancy)

<br />

### 10.2.2 抢先交易

<br />

## 10.3.2 Solidity安全问题

1. **使用未检查的外部调用。**

   Solidity的低级调用（如address.call()）并不抛出异常，而是返回false，所以需检查返回值。

   ```javascript
   if(!addr.send(1)) { revert(); }
   ```

2. **循环次数太多。**耗尽Gas。

3. **合约的创建者拥有过多的权限。**创建者私钥被窃取就GG。

4. **运算精度。**

5. **依赖tx.origin。**

   tx.origin代表调用链上的第一个账户，而msg.sender代表当前的调用者。

   

6. **溢出和下溢。**

7. **不安全的类型引用。**

8. **不正当的转账。**

<br />

## 10.4 智能合约的安全审计

### 审计报告例子

<br />

### Mythril工具分析智能合约安全漏洞

mythril是以太坊官方推荐的安全漏洞分析工具。

