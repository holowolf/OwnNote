## math/rand && crypto/rand 对比

### 1. 前言

`math/rand`软件包无法用于真正的随机数

* `math/rand`: 伪随机生成器
* `crypto/rand`: 加密安全随机数生成器

RobPike代码

```go
package math_rand_and_crypto_rand

import (
	"fmt"
	"github.com/sirupsen/logrus"
	"math/rand"
	"time"
)

func RobPike()  {
	chanString := fanIn(boring("Joe"), boring("Ann"))
	for i := 0; i < 10; i++ {
		logrus.Info(<-chanString)
	}
	fmt.Println("You're both boring; I'm leaving.")
}

func boring(message string) <-chan string {
	chanString := make(chan string)
	go func() {
		for i := 0; ; i++ {
			chanString <- fmt.Sprintf("%s %d", message, i)
			time.Sleep(time.Duration(rand.Intn(1e3)) * time.Millisecond)
		}
	}()
	return chanString
}


func fanIn(input1, input2 <- chan string) <-chan string {
	stringChan := make(chan string)
	go func() {
		for {
			stringChan <- <- input1
		}
	}()
	go func() {
		for {
			stringChan <- <- input2
		}
	}()
	return stringChan
}
```


### 2. Math/rand 伪随机生成器

实现伪随机数生成器

随机数由源生成顶级函数（例如Float64和Int）使用默认的共享源,该源在每次运行程序时都会产生确定的值序列. 如果每次运行需要不同的行为,请使用种子函数初始化默认的源.
默认的Source可安全地供多个goroutine并发使用,但不是由NewSource创建的Source.

```go
package math_rand_and_crypto_rand

import (
	"fmt"
	"github.com/sirupsen/logrus"
	"math/rand"
	"time"
)

func init() {
	rand.Seed(time.Now().UTC().UnixNano())
}

func MathRandom()  {
	chanString := mathRandomFanIn(genrt(), genrt())
	for i := 0; i < 10000; i++ {
		logrus.Info(<-chanString)
	}
}

func mathRandomFanIn(a <-chan int, b <-chan int) <-chan string {
	chanString := make(chan string)
	go func() {
		var count int
		for {
			count += <-a
			chanString <- fmt.Sprintf("Tally of A is: %d", count)
		}
	}()

	go func() {
		var count int
		for {
			count += <-b
			chanString <- fmt.Sprintf("Tally of B is: %d", count)
		}
	}()
	return chanString
}

func genrt() <-chan int {
	chanInt := make(chan int)
	go func() {
		for i := 0; ; i++ {
			chanInt <- rand.Intn(6) + 1
			time.Sleep(500 * time.Millisecond)
		}
	}()
	return chanInt
}
```

Console
```
time="2020-07-13T14:53:49+08:00" level=info msg="Tally of B is: 2"
time="2020-07-13T14:53:49+08:00" level=info msg="Tally of A is: 2"
time="2020-07-13T14:53:49+08:00" level=info msg="Tally of B is: 5"
time="2020-07-13T14:53:50+08:00" level=info msg="Tally of A is: 6"
time="2020-07-13T14:53:50+08:00" level=info msg="Tally of B is: 10"
time="2020-07-13T14:53:50+08:00" level=info msg="Tally of A is: 8"
```

#### 3. crypto/rand 加密安全的随机数生成

```go
package math_rand_and_crypto_rand

import (
	"crypto/rand"
	"fmt"
	"github.com/sirupsen/logrus"
	"math/big"
	"time"
)

func CryptoRandom()  {
	chanString := cryptoFanIn(cryptoGenrt(), cryptoGenrt())
	for i := 0; i < 10000; i++ {
		logrus.Info(<-chanString)
	}
}

func cryptoFanIn(a <-chan int, b <-chan int) <-chan string {
	chanString := make(chan string)
	go func() {
		var count int
		for {
			count += <-a
			chanString <-fmt.Sprintf("Tally of A is: %d", count)
		}
	}()

	go func() {
		var count int
		for {
			count += <-b
			chanString <-fmt.Sprintf("Tally of B is: %d", count)
		}
	}()
	return chanString
}

func cryptoGenrt() <-chan int {
	chanInt := make(chan int)
	go func() {
		for i := 0; ; i++ {
			dice, err := rand.Int(rand.Reader, big.NewInt(6))
			if err != nil {
				logrus.Info(err)
			}
			chanInt <- int(dice.Int64()) + 1
			time.Sleep(1 * time.Millisecond)
		}
	}()
	return chanInt
}
```

Console
```
time="2020-07-13T15:17:14+08:00" level=info msg="Tally of A is: 6"
time="2020-07-13T15:17:14+08:00" level=info msg="Tally of B is: 2"
time="2020-07-13T15:17:14+08:00" level=info msg="Tally of A is: 11"
```


