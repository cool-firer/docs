# vscode配置跳转

moby没用go mod模式，用的vendor，需要配置一下。

步骤：

1. moby代码路径：`/Users/demon/Desktop/source/docker/moby`；

2. 新建一个路径：`mkdir -p /Users/demon/Desktop/source/docker/mymoby/src/github.com/docker`；

3. 建立软链接：

```shell
ln -s /Users/demon/Desktop/source/docker/moby /Users/demon/Desktop/source/docker/mymoby/src/github.com/docker/docker
```

4. 用vscode打开: `code /Users/demon/Desktop/source/docker/mymoby/src/github.com/docker/docker`；

5. 在vscode新加个配置：`.vscode/settings.json` 内容如下：

   ```json
   {
     "go.gopath": "/Users/demon/Desktop/source/docker/mymoby",
     "go.toolsEnvVars": {
       "GO111MODULE": "auto"
     },
     "editor.links": false,
   }
   ```

6. vscode重新打开，即可跳转了。





# dockerd

入口main: cmd/dockerd/docker.go











## 用到的包

reexec

 `github.com/docker/docker/pkg/reexec`

用来执行自身，在clone前设置初始化、配置。用了linux里的namespace隔离各个容器的hostname、network、mount等。



[unix](https://pkg.go.dev/golang.org/x/sys@v0.0.0-20220513210249-45d2b4557a2a/unix#Umask)

`golang.org/x/sys/unix`

设置屏蔽码

```go
	desiredUmask := 0022
	unix.Umask(desiredUmask) // user保留系统设定, group、other去掉x权限
```





参考资料

namespace、reexec用法说明：https://here2say.com/39/

文件名区分平台：https://blog.csdn.net/l7l1l0l/article/details/104631228

go mod vendor使用：https://zhuanlan.zhihu.com/p/116410261

vscode配置moby跳转：https://blog.csdn.net/qq_17004327/article/details/116358363

