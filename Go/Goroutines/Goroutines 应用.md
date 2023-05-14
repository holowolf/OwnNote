## Goroutines

在Go语言中,每一个并发执行单元叫作一个goroutines.当程序启动时.  
其主函数会在一个单独的goroutine运行.  
一般称为main goroutine.新的goroutine会用go语句创建.  
go 语句会使其语句中的函数在新创建的goroutine中运行.


goroutine 运行时管理的轻量级线程

开启并行
```
go f(x, y, z)
```

正常运行
```
f(x, y, z)
```

示例代码

```go
package goroutine

import (
	"fmt"
	"time"
)

func say(s string) {
	for i := 0; i < 5; i++ {
		time.Sleep(100 * time.Millisecond)
		fmt.Println(s)
	}
}

func Run() {
	go say("world")
	say("hello")
}
```


