v2ray换网站了，文档也换了，现在的地址：

git: https://github.com/v2fly

文档: https://www.v2fly.org/



# 环境

主机：macOS Catalina 10.15.7 (19H15)

go版本：go version go1.15.2 darwin/amd64

v2ray版本：4.31.0



# 服务端

## 编译&运行

```shell
cd $(go env GOPATH)/src/v2ray.com/core/main
env CGO_ENABLED=0 go build -o ./v2ray -ldflags "-s -w"

cd $(go env GOPATH)/src/v2ray.com/core/infra/control/main
env CGO_ENABLED=0 go build -o $(go env GOPATH)/src/v2ray.com/core/main/v2ctl -tags confonly -ldflags "-s -w"

# 增加配置文件 config.json
{
  "inbounds": [
    {
      "port": 10086,
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "b831381d-6324-4d53-ad4f-8cda48b30811"
          }
        ]
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "settings": {}
    }
  ]
}

# 运行:
./v2ray --config=./config.json
```







# 用到的库

## flag

```go
import "flag"

type Arg []string
func (c *Arg) String() string {
	return strings.Join([]string(*c), " ")
}
func (c *Arg) Set(value string) error {
	*c = append(*c, value)
	return nil
}

var (
  // 返回*bool指针
	version = flag.Bool("version", false, "Show current version of V2Ray.")
  configFiles cmdarg.Arg
)

// 自定义 需要实现 String()、Set(xx)方法
flag.Var(&configFiles, "config", "xxx")

// 最后调用 Parse 解析
flag.Parse()
```



## strings

```go
import "strings"

const name = "v2ray.location.confdir"

// 去掉空格
strings.TrimSpace(name)

// 转成大写
strings.ToUpper(strings.TrimSpace(name))

// 替换成 v2ray_location_confdir  
// 最后的参数 < 0 表示全替换
strings.Replace(strings.ToUpper(strings.TrimSpace(name)), ".", "_", -1)
```



# TCP的Conn

Accept返回的Conn

- [func (l *TCPListener) Accept() (Conn, error)](https://pkg.go.dev/net#TCPListener.Accept)

Conn是个接口，同属net模块：

```go
type Conn interface {
	Read(b []byte) (n int, err error)
	Write(b []byte) (n int, err error)
	Close() error
	LocalAddr() Addr
	RemoteAddr() Addr
	SetDeadline(t time.Time) error
	SetReadDeadline(t time.Time) error
	SetWriteDeadline(t time.Time) error
}
```



嵌入net.Conn，就能接收

```go
import (
	"net"
)

type Connection interface {
	net.Conn
}

v.addConn(Connection(conn)) // 调用callback

func (w *tcpWorker) callback(conn internet.Connection) {}
```



转成只读的

```go
type readerOnly struct {
	io.Reader
}

func (s *Server) Process(conn internet.Connection) error {
	reader := bufio.NewReaderSize(readerOnly{conn}, buf.Size)
}
```



用bufio包装

func NewReaderSize(rd io.Reader, size int) *Reader

> NewReaderSize returns a new Reader whose buffer has at least the specified size. If the argument io.Reader is already a Reader with large enough size, it returns the underlying Reader.

```go
func (s *Server) Process(conn internet.Connection) error {
	reader := bufio.NewReaderSize(readerOnly{conn}, buf.Size)
}

// 返回的Reader是一个struct, 内嵌了Reader接口
type Reader struct {
	buf          []byte
	rd           io.Reader // reader provided by the client
	r, w         int       // buf read and write positions
	err          error
	lastByte     int // last byte read for UnreadByte; -1 means invalid
	lastRuneSize int // size of last rune read for UnreadRune; -1 means invalid
}
```



阻塞等待一个http包

```go
// http.ReadRequest正好接收一个*bufio.Reader结构体
request, err := http.ReadRequest(reader)

// unc ReadRequest(b *bufio.Reader) (*Request, error) { ... }
```





http代理测试程序

```js
'use strict';

const net = require('net');

const ip = '192.168.2.166';

function test() {
  const client = net.createConnection({
    host: '127.0.0.1',
    port: 1080,
  }, () => {
    console.log('connected to server!');
    client.write('CONNECT 192.168.2.166:8080 HTTP/1.1\r\n' +
    'Host: 192.168.2.166:8080\r\n' +
    'Proxy-Connection: Keep-Alive\r\n' +
    'User-Agent: Go-http-client/1.1\r\n' +
    '\r\n');
  });

  client.on('data', async data => {
    console.log(data.toString() + 'kkk');
    if (data.toString().indexOf('200 Connection established') >= 0) {
      await new Promise((r, j) => {
        setTimeout(() => {
          r(true);
        }, 5000);
      });
      client.write('GET /hello HTTP/1.1\r\n' +
      'Host: 192.168.2.202:8080\r\n' +
      'Content-Length: 0\r\n' +
      '\r\n');

      // client.write('GET /hel');


      await new Promise((r, j) => {
        setTimeout(() => {
          r(true);
        }, 10000);
      });

      // client.write('lo HTTP/1.1\r\n' +
      // 'Host: 192.168.2.166:8080\r\n' +
      // 'Content-Length: 0\r\n' +
      // '\r\n');

      client.end();

    }
  });

  client.on('end', () => {
    console.log('disconnected from server');
  });
}


function testHTTP() {

  const axios = require('axios');
  axios.get('http://192.168.2.166:8080/hello', {
    proxy: {
      host: '127.0.0.1',
      port: '1080',
    },
  })
    .then(function(response) {
      console.log(response.data);
    })
    .catch(function(error) {
      console.log('err:', error.msg);
    });
}

test();

```



修改源码后生效

go build -a hello.go -a表示所有import的包都从源码编译，而不是引用已经编译好的静态库。