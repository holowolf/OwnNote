## Goroutine 调度



```go
package goroutine

import (
	"fmt"
	"runtime"
	"time"
)

func RunGoroutine() {
	var a [10]int
	for i := 0; i < 10; i++ {
		go func(i int) {
			for {
				a[i]++
				// 通过Gosched释放cpu权指
				runtime.Gosched()
			}
		}(i)
	}
	time.Sleep(time.Millisecond)
	fmt.Println(a)
}
```