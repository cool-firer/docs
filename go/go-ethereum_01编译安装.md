# go-ethereum笔记 -- 01编译安装



# 环境

Go version: go version go1.20.3 darwin/amd64

版本：https://github.com/ethereum/go-ethereum/tree/release/1.10

直接下载zip包下来, 解压就行。



# 编译

cd  go-ethereum1.10

make geth

编译成功，生成 ./build/bin/geth





# 跑起来

新建一个目录：

mkdir XXX_DATA

cd XXX_DATA

**➜** **XXX_DATA** pwd

/Users/luke/Desktop/work/go-ethereum1.10/XXX_DATA



新建一个genesis.json

XXX_DATA/genesis.json

```json
{
  "config": {
        "chainId": 10,
        "homesteadBlock": 0,
        "eip150Block": 0,
        "eip155Block": 0,
        "eip158Block": 0
  },
  "alloc"      : {},
  "coinbase"   : "0x0000000000000000000000000000000000000000",
  "difficulty" : "0x0200000",
  "extraData"  : "",
  "gasLimit"   : "0x2fefd8",
  "nonce"      : "0x0000000000000042",
  "mixhash"    : "0x0000000000000000000000000000000000000000000000000000000000000000",
  "parentHash" : "0x0000000000000000000000000000000000000000000000000000000000000000",
  "timestamp"  : "0x00"
}

```



生成创世区块：

XXX_DATA/

 ../build/bin/geth --datadir "./chain_data" init genesis.json

会生成chai_data目录， geth 和 keystore，两个文件夹分别用于保存如下内容：

| 目录     | 作用                           |
| -------- | ------------------------------ |
| geth     | 保存区块链相关数据，如：数据库 |
| keystore | 保存账户信息                   |



创建私有链，启动区块链：

../build/bin/geth --datadir "./chain_data" --nodiscover console 2>>eth_output.log


参数说明:

- –nodiscover 表示该链不可被发现，即非公开的
- console 进入JavaScript 命令行模式
- 2>>eth_output.log 指定日志文件



控制台操作

`eth.accounts` 命令查看已有账户情况



创建账户

personal.newAccount("123456") 指定密码



开始挖矿

`miner.start()` 开始挖矿，挖矿奖励的币会默认保存到第一个创建的账户中。



`miner.stop()` 停止挖矿



V4:

"enode://d860a01f9722d78051619d1e2351aba3f43f943f6f00718d1b9baa4101932a1f5011f16bb2b1bb35db20d6fe28fa0bf09636d26a87d31de9ec6203eeedb1f666@18.138.108.67:30303"



V5:

"enr:-KG4QOtcP9X1FbIMOe17QNMKqDxCpm14jcX5tiOE4_TyMrFqbmhPZHK_ZPG2Gxb1GE2xdtodOfx9-cgvNtxnRyHEmC0ghGV0aDKQ9aX9QgAAAAD__________4JpZIJ2NIJpcIQDE8KdiXNlY3AyNTZrMaEDhpehBDbZjM_L9ek699Y7vhUJ-eAdMyQW_Fil522Y0fODdGNwgiMog3VkcIIjKA"

##  



RLP编码

uint

uint8、uint16、uint32、uint64都按照uint64来处理。对于uint(x)的值：

1. x == 0，直接写入 0x80;

2. x < 128，写入自身 x;

   ```js
   这里为什么是 < 128? 128 = 2^7, 二进制是
   0 000 0000
   1 000 0000
   
   因为是要用多字节存储，所以需要一个办法知道，后面还有没有连续的字节，所以牺牲了部分数据范围，用最高位来表示后续是否还有字节。
   ```

   

3. 否则，写入 0x80 + 字节长度  高位  低位;

   

![rpl_uint](images/rpl_uint.png)

[简单的验证步骤。](./rlp验证#kk)



big.Int

位数 > 64位，不支持负大数。x为bit.Int， 二进制位数为bitlen，bitlen的字节长度为length。

1. length < 56，写入0x80 + length；
2. 否则，写入0xB7 + length的字节长度  length值高位  length值低位；
3. 写入x的高位字节  x的低字节；



例1：x = 0x102030405060708090A0B0C0D0E0F2

二进制位数bitlen = 120，字节长度length = 15，

最终写入 0x80+15  10  20  30  ....  F2



例2：x = 0x102030405060708090A0B0C0D0E0F2

​				102030405060708090A0B0C0D0E0F2

​				102030405060708090A0B0C0D0E0F2

​				102030405060708090A0B0C0D0E0F2

方便查看, 中间用空格隔开。

二进制位数bitlen = 480，字节长度length = 60，

最终写入 0xB7+1(存60只要一个字节)   0x3C(60)  10  20  30  ....  F2   10  20  30 ...  F2



List

首先对一个扁平的list编码：

list = [ X~1~, X~2~, X~3~, ....   X~n~ ]，假设X~i~ 是unit64，list的长度是n，

1. n < 56，写入0xC0 + n 、X~1~、 X~2~  ...  X~n~ ；
2. n >= 56，写入 0xF7 + n的字节长度 、n的高位、n的低位、X~1~、X~2~、... X~n~；

举例：list = [ 1, 2, 3, 4, 5, 6 ]，编码就是：0xC0+6、1、2、3、4、5、6。

list = [ 1, 2, 3, 4, ... , 257 ]，257二进制表示为：1 00000001 需要两个字节存下257，那么257的字节长度就是2，编码就是：0xF7+2、0x01(257的高位)、0x01(257的低位)、1、2、3、... 257。



可以看到，一个list固定需标志（0xC0、0xF7这些），然后是长度，对于长度<56的，长度内嵌到标志里，而对于长度>=56的，需要额外的长度字节。即 len(list) = 标志 + 长度字节(数据>=56才有) + 数据。

嵌套的list相当于把里面的子list当做数据，只不过这些数据的长度 = len(子list)。

举例 list = [ 1, 2, 3, [ 4, 5, 6, 7, 8, 9 ] ]：

先对子list编码，[ 4, 5, 6] = 0xC0+6、4、5、6、7、8、9，len(子list) = 1 + 6 = 7；

再外层list编码，[ 1, 2, 3, 长度7的一坨 ]：走第一条规则，

就是：0xC0+10、1、2、3、0xC0+6、4、5、6、7、8、9。

[对应到代码的应用是这样](./rlp验证#List)。



结构体

没怎么看懂，先放过



