ctx首次出现在

/Users/demon/Desktop/work/gowork/src/v2ray.com/core/v2ray.go

```go
func New(config *Config) (*Instance, error) {
  
	var server = &Instance{ctx: context.Background()}
  ...
}
```



Background()定义

```go
// /usr/local/go/src/context/context.go
package context

type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}

type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}

func (e *emptyCtx) String() string {
	switch e {
	case background:
		return "context.Background"
	case todo:
		return "context.TODO"
	}
	return "unknown empty Context"
}

var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

func Background() Context {
	return background
}

```

返回包级变量background，用Context接收，emptyCtx实现了Context接口。



遍历config.App时：

```go
for _, appSettings := range config.App {
  obj, err := CreateObject(server, settings)
  ...
}


func CreateObject(v *Instance, config interface{}) (interface{}, error) {
	ctx := v.ctx
	if v != nil {
		// v2rayKey: 1
		ctx = context.WithValue(ctx, v2rayKey, v)
	}
	return common.CreateObject(ctx, config)
}
```

调用了context.WithValue方法。

```go
// /usr/local/go/src/context/context.go
func WithValue(parent Context, key, val interface{}) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}

type valueCtx struct {
	Context
	key, val interface{}
}
```



此时ctx指向改变了

再调用common.CreateObject(ctx, config)、接下就是到各个typeRegister里了。

v2ray.core.app.log.Config里没用到ctx。



v2ray.core.app.dispatcher.Config里有用到。

```go
func init() {
	common.Must(common.RegisterConfig((*Config)(nil), func(ctx context.Context, config interface{}) (interface{}, error) {
		
		d := new(DefaultDispatcher)

		if err := core.RequireFeatures(ctx, func(om outbound.Manager, router routing.Router, pm policy.Manager, sm stats.Manager) error {
		
			fmt.Println("Fuck Ass Code")
			return d.Init(config.(*Config), om, router, pm, sm)
		}); err != nil {
			return nil, err
		}
		return d, nil
	}))
}
```

