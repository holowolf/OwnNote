## Golang 简单正则

```go
package regex

import (
	"fmt"
	"regexp"
)

const text = `
My email is yeenughu@gmail.com
email is Gooonughu@gmail.com
email is Kuuunughu@gmail.com
`

func RunHello() {
	re := regexp.MustCompile(`(\w+)@(\w+)\.(\w+)`)
	s := re.FindAllStringSubmatch(text, -1)
	fmt.Println(s)
}
```

输出子匹配内容
```txt
[[yeenughu@gmail.com yeenughu gmail com] [Gooonughu@gmail.com Gooonughu gmail com] [Kuuunughu@gmail.com Kuuunughu gmail com]]
```