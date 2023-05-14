## Select基本用法

select 只能用于channel 而且随机选取一个

```go
func SelectTodo()  {
	todo := make(chan int, 1)
	for i := 0; i < 100; i++ {
		todo <- i
		select {
		case <- todo:
			fmt.Println("random 1")
		case <- todo:
			fmt.Println("random 2")
		default:
			fmt.Println("exit")
		}
	}
}
```

select 使用channle超时用法

```go
func SelectTimeOut() {
	todo := make(chan int, 1)
	select {
	case <-todo:
		fmt.Println("todo")
	case <-time.After(time.Second * 1):
		fmt.Println("time out 02")
	}
}
```

select 检查channel是否已满

```go
func main() {
    ch := make(chan int, 2)
    ch <- 1
    select {
    case ch <- 2:
        fmt.Println("channel value is", <-ch)
        fmt.Println("channel value is", <-ch)
    default:
        fmt.Println("channel blocking")
    }
}
```

select for Loop 用法

```go
func SelectForLoop()  {
	i := 0
	message := make(chan string, 1)
	defer func() {
		close(message)
	}()

	go func() {
		LOOP:
			for {
				time.Sleep(time.Second)
				fmt.Println(time.Now().Unix())
				i++

				if i == 1 {
					message <- "我来了"
				}

				select {
				case m := <-message:
					fmt.Println(m)
					break LOOP
				default:
				}
			}
	}()

	time.Sleep(time.Second * 4)
	message <- "stop"
}
```








