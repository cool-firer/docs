# go中的mysql



网上千篇一律的，一上来就直接

```go
import (
	"fmt"
	_ "github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)
```

之后就是各种用法了，也不管人受得了受不了。我觉得有很有必要理清一下database/sql与sqlx、driver的关系。





原生database/sql

官网写的很清楚，只提供通用的泛型接口，要使用这个包呢，必须跟一个驱动库一起使用。

一个是顶层设计，只提供你蓝图，另一个是包工头，负责搬砖实现。驱动库就是搬砖者。

[官方列出的驱动库](https://github.com/golang/go/wiki/SQLDrivers)有很多，其中之一就是go-sql-driver。



