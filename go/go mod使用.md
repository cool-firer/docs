### go mod使用



版本：go version go1.15.2 darwin/amd64



ikka/ 目录下

go mod init：生成go.mod、go.sum文件



文件结构：

ikka/

​	go.mod

​	go.sum

​	main.go

​	core/

​		application.go

​		loader/core_loader.go

​		logger/

​			console.go

​			logger.go



application.go内容：

```go
package core

import "ikka/core/logger"
import "ikka/core/loader"

type CoreApplication struct {
	options interface{}
	console logger.ConsoleLogger
	loader *loader.CoreLoader
}

func (app CoreApplication) Loader() (*loader.CoreLoader) {
	return app.loader;
}
```



main.go内容：

```go
package main

import "fmt"
// import _ "ikka/core"

// import "ikka/core/logger/transport"
import . "ikka/core/logger"
import . "ikka/core"

func main() {
	fmt.Println("I am coming")

	var cl ConsoleLogger = ConsoleLogger{}
	cl.Info("what the fuck")

	cl.Info("what the fuck2222222222")

	cl.Info("what the fuck3333333333")

	var app CoreApplication = CoreApplication{}
	app.Loader().LoadPlugin()
}
```



<br />

在与ikka同级目录下，引入ikka包。

room/

​	go.mod

​	go.sum

​	main.go

<br />

go.mod内容：

```go
module room

go 1.15


require ikka v0.0.0-00010101000000-000000000000
replace ikka => ../ikka
```



main.go内容：

```go
package main

import . "ikka/core"

func main()  {
	var app CoreApplication = CoreApplication{}
	app.Loader().LoadPlugin()
}
```



运行go run main.go 正常。



必要的时候开启 set GO111MODULE=on



参考：https://www.codenong.com/cs110395119/