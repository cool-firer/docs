# RLP uint验证

1. 进入xxx/go-ethereum1.10/rlp/ 目录, 新建一个my_encode_test.go文件，内容如下：

   ```go
   package rlp
   
   import (
   	"fmt"
   	"testing"
   	"bytes"
   	"encoding/hex"
   )
   
   func TestEcodeUint(t *testing.T) {
   	b := new(bytes.Buffer)
   	var val interface{}  = uint32(256)
   	Encode(b, val)
   	fmt.Println(hex.Dump(b.Bytes()))
   }
   ```

2. cd到go-ethereum1.10/rlp/，执行 go test -v -run ^TestEcodeUint$：

   ```shell
   ➜  go test -v -run ^TestEcodeUint$
   === RUN   TestEcodeUint
   00000000  82 01 00                                          |...|
   
   --- PASS: TestEcodeUint (0.00s)
   PASS
   ok  	github.com/ethereum/go-ethereum/rlp	0.515s
   ```





# RLP bitInt验证

```go
package rlp

import (
	"fmt"
	"testing"
	"bytes"
	"encoding/hex"
	"math/big"
)

func TestEncodeBigInt(t *testing.T) {
	s, _ := hex.DecodeString("102030405060708090A0B0C0D0E0F2")
	var val interface{}  = new(big.Int).SetBytes(s)

	b := new(bytes.Buffer)
	Encode(b, val)
	fmt.Println(hex.Dump(b.Bytes()))


	b.Reset()
	s, _ = hex.DecodeString("102030405060708090A0B0C0D0E0F2102030405060708090A0B0C0D0E0F2102030405060708090A0B0C0D0E0F2102030405060708090A0B0C0D0E0F2")
	val = new(big.Int).SetBytes(s)
	Encode(b, val)
	fmt.Println(hex.Dump(b.Bytes()))
}
```

