作者：Amit Saha（澳）

译：贾玉彬、刘光磊

2022年11月第一版



# 第4章 高级HTTP客户端

**http.Client基本使用**

1) 发请求

```go
c := http.Client{Timeout: 100 * time.Second}
r, err := c.Get(url)
```



2) 重定向。默认会有重定向，可以控制重定向的行为：

```go
// via是重定向经过的ulr
func redirectPolicy(req *http.Request, via []*http.Request) {}

c := http.Client{Timeout: t, checkRedirect: redirectPolicy)
```



3) 定制请求。 用http.NewRequestWithContext：

```go
ctx, cancel := context.WithTimeout(context.Background(), 15 * time.Second)
req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
defer cancel()

resp, err := client.Do(req)
```



4) 中间件功能。用RoundTipper接口做。net/http包里抽象为Transport：

```go
type LogginClient struct {
  log *log.Logger
}

func (c LogginClient) RoundTrip(r *http.Request) (*http.Response, error) {
  c.log.Printf("请求前")
  resp, err := http.DefaultTransport.RoundTrip(r)
  c.log.Printf("请求后")
  return resp, err
}

// 注册
myTransport := LoggingClient{}
client := http.Client{
  Timeout,
  Transport: &myTransport
}
```



5) 单元测试。Go提供了专门的包方便做单测：net/http/httptest。



**连接池**

Go内部会自己维护一个tcp连接池，复用来发http请求。可以用net/http/httptrace包跟踪。



# 第5章 构建HTTP服务器

基本用法

```go
import "net/http"

func apiHandler(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, "Hello World")
}

mux := http.NewServeMux()
mux.HandleFunc("/api", apiHandler)
mux.HandleFunc("/check", checkHandler)

http.ListenAndServe(":8080", mux)
```



json.NewDecoder()函数使用场景

```go
假设客户端发送这样的数据：
'{ a: 1, b:2 } { a: 3, b: 4 }' 非正常的json数组。

这时候可以用json.Decoder一个个解析出来。

func decodeHandler(w http.ResponseWriter, r *http.Request) {
  dec := json.NewDecoder(r.Body)
  for {
    var d Line
    err := dec.Decode(&d)
    if err == io.EOF {
      break
    }
    if err != nil { // 解析出错
      return
    }
  }
}
```



# 第6章高级HTTP服务器应用程序

http.ListenAndServe(addr string, handler Handler)本质

```go
type Handler interface {
  ServeHTTP(ResponseWriter, *Request)
}

实现了Handler接口就可以。

```

