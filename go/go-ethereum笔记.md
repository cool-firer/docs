go-ethereum笔记



版本：https://github.com/ethereum/go-ethereum/tree/release/1.10

直接下载zip包下来, 解压就行。



RLP编码

uint

uint8、uint16、uint32、uint64都按照uint64来处理。对于uint(x)的值：

1. x == 0，直接写入 0x80;
2. x < 128，写入自身 x;
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





