go-ethereum笔记 -- 02 cli



 ../build/bin/geth --datadir "./chain_data" init genesis.json



cli App的结构, 只列出关键部分

```go
import ( "github.com/urfave/cli/v2" )
app := cli.NewApp()

app: &App {
  Action: getFunc,
  Commands: []*cli.Command [
    
    // 第一个command
    &cli.Command{ // 名为initCommand的指针
      Action: initGeneis,
      Name: "init",
      Usage: "xxx",
      ArgsUsage: "xxx",
      Flags: []cli.Flag [
        
        &DirectoryFlag{ // 不在cli里面, eth自定义的名为 DataDirFlag 的指针
          Name: "datadir",
          Usage: "Data directory for the databases and keystore",
          Value: "临时目录",
          Category: "ETHEREUM", // 类别
        },
        &DirectoryFlag{ // 不在cli里面, eth自定义的名为 AncientFlag 的指针
          Name: "datadir.ancient",
          Usage: "xxxx",
          Category: "ETHEREUM",
        },
        &cli.StringFlag{ // 在cli里面定义的struct
          Name: "remotedb",
          Usage: "xxxx",
          Category: "LOGGING AND DEBUGGING", // 这个类别不一样
        },
       ]
    },
    
    // 第二个command, 所有支持的command
    ...
  ], // end Command
  Flags: 所有Command的Flag 拍扁的[]flag
  Before: func,
  After: func,
 
}
```

目前为止, so far so good，直到func (a *App) Setup()，开始神经病写法了。

eth重新赋值了cli里的FlagStringer，恶心：

```go
func init() {
	cli.FlagStringer = FlagString
}
```

<br />

Setup()三件事：

一、为app的command填充了flagCategories，同类别的flag 聚在一起；

二、同类别的command聚在一起 command category；

三、同类别的flag聚在一起 flag category

```go
import ( "github.com/urfave/cli/v2" )
app := cli.NewApp()

app: &App {
  Commands: []*cli.Command [
    
    &cli.Command{
      Action: initGeneis,
      Name: "init",
      Usage: "xxx",
      ArgsUsage: "xxx",
 
      // 新赋值的 同flag category的聚在一起
      flagCategories: &defaultFlagCategories{
        m: {
          "ETHEREUM" -> &defaultVisibleFlagCategory{
            name: "ETHEREUM",
            m: {
              // 一大串字符串作key
              "--datadir value \t $usage " -> flag
              "datadir.ancient value \t usage" -> flag
            }
          },
          
          "LOGGING AND DEBUGGING" -> &defaultVisibleFlagCategory{
            name: "LOGGING AND DEBUGGING",
            m: {
              // 一样的大串
              "--remotedb value usgae" -> flag
            }
          },
             
        
      },
        
      }
    },

 // 同command category的 command聚在一起
 categories: []*commandCategory [ // 类型别名commandCategories
   
   &commandCategory{ // 第一个category
     name: "", commands: []*Command{ initCommand }})
   },
 ],

 // 同flag category的 flag聚在一起 跟command里的一样
 flagCategories: &defaultFlagCategories{
		m: {
      "ETHEREUM" -> &defaultVisibleFlagCategory{
        name: "ETHEREUM",
        m: {
          "--datadir value \t $usage " -> flag
          "datadir.ancient value \t usage" -> flag
        },
      },
      
      "LOGGING AND DEBUGGING" -> 
      "其他flag category",
    },
 },

 Metadata: make(map[string]interface{}),
    

}
```



 原生的flagSet， set := flag.NewFlagSet(name, flag.ContinueOnError)

遍历所有的flag，设置set： set.Var(&f.Value, f.Name, f.Usage)



  开始解析：err := set.Parse(args) // 调用的原生的, 结果保存在set里面。



new一个cli里面的Context { 

​	App: app, 

​	flagSet: set, 

​	parentContext: &Context{ 

​				Context: ctx(基) ,

​				flagSet = &flag.FlagSet{}

​	},

​	Context: 同parent指向,

​	Command: &Command{},

​	shellComplete: false

}



App.Before()在Action之前执行。







用到的包

path/filepath

```go
import (
	"path/filepath"
)

// "baz.js"
filepath.Base("/foo/bar/baz.js")

filepath.Abs("ba.js")
```



Strings:

```go
package main

import (
	"fmt"
	"strings"
)

// Hello, Gophers
func main() {
	fmt.Print(strings.Trim("¡¡¡Hello, Gophers!!!", "!¡"))
}
```

