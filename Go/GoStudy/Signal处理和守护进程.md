## Signal处理和守护进程

### 1.Golang信号处理

我们在生产环境下运行的系统要求优雅退出，即程序接收退出通知后，会有机会先执行一段清理代码，将收尾工作做完后再真正退出.
我们采用系统Signal来 通知系统退出，即kill pragram-pid.我们在程序中针对一些系统信号设置了处理函数，当收到信号后，会执行相关清理程序或通知各个子进程做自清理.kill -9强制杀掉程序是不能被接受的，那样会导致某些处理过程被强制中断，留下无法恢复的现场，导致消息被破坏，影响下次系统启动运行.

最近用Golang实现的一个代理程序也需要优雅退出，因此我尝试了解了一下Golang中对系统Signal的处理方式，这里和大家分享.Golang 的系统信号处理主要涉及os包、os.signal包以及syscall包.其中最主要的函数是signal包中的Notify函数：

```go
func Notify(c chan<- os.Signal, sig …os.Signal)
```

该函数会将进程收到的系统Signal转发给channel c.转发哪些信号由该函数的可变参数决定，如果你没有传入sig参数，那么Notify会将系统收到的所有信号转发给c.如果你像下面这样调用Notify：

```go
signal.Notify(c, syscall.SIGINT, syscall.SIGUSR1, syscall.SIGUSR2)
```

则Go只会关注你传入的Signal类型，其他Signal将会按照默认方式处理，大多都是进程退出.因此你需要在Notify中传入你要关注和处理的Signal类型，也就是拦截它们，提供自定义处理函数来改变它们的行为.

#### 信号类型

个平台的信号定义或许有些不同.下面列出了POSIX中定义的信号. Linux 使用34-64信号用作实时系统中. 命令 man signal 提供了官方的信号介绍. 在POSIX.1-1990标准中定义的信号列表

信号 | 值 | 动作 | 说明 |
---|---|---|---|
SIGHUP | 1 | Term | 终端控制进程结束(终端连接断开) |
SIGINT | 2 | term | 用户发送INTR字符(Ctrl+C)触发 |
SIGQUIT| 3 | Core | 用户发送QUIT字符(Ctrl+/)触发 |
SIGILL | 4 | Core | 非法指令(程序错误、试图执行数据段、栈溢出等) |
SIGABRT| 6 | Core | 调用abort函数触发 |
SIGFPE | 8 | Core | 算术运行错误(浮点运算错误、除数为零等) |
SIGKILL| 9 | Term | 无条件结束程序(不能被捕获、阻塞或忽略) |
SIGSEGV| 11| Core | 无效内存引用(试图访问不属于自己的内存空间、对只读内存空间进行写操作) |
SIGPIPE| 13| Term | 消息管道损坏(FIFO/Socket通信时，管道未打开而进行写操作) |
SIGALRM| 14| Term | 时钟定时信号 |
SIGTERM| 15| Term | 结束程序(可以被捕获、阻塞或忽略) |
SIGUSR1| 30,10,16 | Term | 用户保留 |
SIGUSR2| 31,12,17 | Term | 用户保留 |
SIGCHLD| 20,17,18 | Ign | 子进程结束(由父进程接收) |
SIGCONT| 19,18,25 | Cont | 继续执行已经停止的进程(不能被阻塞) |
SIGSTOP| 17,19,23 | Stop | 停止进程(不能被捕获、阻塞或忽略) |
SIGTSTP| 18,20,24 | Stop | 停止进程(可以被捕获、阻塞或忽略) |
SIGTTIN| 21,21,26 | Stop | 后台程序从终端中读取数据时触发 |
SIGTTOU| 22,22,27 | Stop | 后台程序向终端中写数据时触发 |

#### 在SUSv2和POSIX.1-2001标准中的信号列表

信号 | 值 | 动作 | 说明 |
---|---|---|---|
SIGTRAP | 5 | Core | Trap指令触发(如断点，在调试器中使用) |
SIGBUS | 0,7,10 | Core | 非法地址(内存地址对齐错误) |
SIGPOLL |  | Term | Pollable event (Sys V). Synonym for SIGIO |
SIGPROF | 27,27,29 | Term | 性能时钟信号(包含系统调用时间和进程占用CPU的时间) |
SIGSYS | 12,31,12 | Core | 无效的系统调用(SVr4) |
SIGURG | 16,23,21 | Ign | 有紧急数据到达Socket(4.2BSD) |
SIGVTALRM | 26,26,28 | Term | 虚拟时钟信号(进程占用CPU的时间)(4.2BSD) |
SIGXCPU | 24,24,30 | Core | 超过CPU时间资源限制(4.2BSD) |
SIGXFSZ | 25,25,31 | Core | 超过文件大小资源限制(4.2BSD) |

```
第1列为信号名；
第2列为对应的信号值，需要注意的是，有些信号名对应着3个信号值，这是因为这些信号值与平台相关，将man手册中对3个信号值的说明摘出如下，the first one is usually valid for alpha and sparc, the middle one for i386, ppc and sh, and the last one for mips.
第3列为操作系统收到信号后的动作，Term表明默认动作为终止进程，Ign表明默认动作为忽略该信号，Core表明默认动作为终止进程同时输出core dump，Stop表明默认动作为停止进程.
第4列为对信号作用的注释性说明，浅显易懂，这里不再赘述.
需要特别说明的是，SIGKILL和SIGSTOP这两个信号既不能被应用程序捕获，也不能被操作系统阻塞或忽略.
```

### 2.kill pid与kill -9 pid的区别

kill pid的作用是向进程号为pid的进程发送SIGTERM（这是kill默认发送的信号）,
该信号是一个结束进程的信号且可以被应用程序捕获.若应用程序没有捕获并响应该信号的逻辑代码，则该信号的默认动作是kill掉进程.这是终止指定进程的推荐做法.

kill -9 pid则是向进程号为pid的进程发送SIGKILL（该信号的编号为9）,从本文上面的说明可知,SIGKILL既不能被应用程序捕获,也不能被阻塞或忽略,
其动作是立即结束指定进程.通俗地说，应用程序根本无法“感知”SIGKILL信号，它在完全无准备的情况下，就被收到SIGKILL信号的操作系统给干掉了，显然,
在这种“暴力”情况下，应用程序完全没有释放当前占用资源的机会.事实上，SIGKILL信号是直接发给init进程的，它收到该信号后，负责终止pid指定的进程.
在某些情况下（如进程已经hang死，无法响应正常信号），就可以使用kill -9来结束进程.

### 3. 应用程序如何优雅退出

Linux Server端的应用程序经常会长时间运行，在运行过程中，可能申请了很多系统资源，也可能保存了很多状态，在这些场景下，我们希望进程在退出前,
可以释放资源或将当前状态dump到磁盘上或打印一些重要的日志，也就是希望进程优雅退出（exit gracefully）.

从上面的介绍不难看出，优雅退出可以通过捕获SIGTERM来实现.具体来讲，通常只需要两步动作：

* 注册SIGTERM信号的处理函数并在处理函数中做一些进程退出的准备.信号处理函数的注册可以通过signal()或sigaction()来实现，其中，
推荐使用后者来实现信号响应函数的设置.信号处理函数的逻辑越简单越好，通常的做法是在该函数中设置一个bool型的flag变量以表明进程收到了SIGTERM信号，准备退出.

* 在主进程的main()中，通过类似于while(!bQuit)的逻辑来检测那个flag变量，一旦bQuit在signal handler function中被置为true，
则主进程退出while()循环，接下来就是一些释放资源或dump进程当前状态或记录日志的动作，完成这些后，主进程退出.


### 4. Go中的Signal发送和处理

* golang中对信号的处理主要使用os/signal包中的两个方法：
* notify方法用来监听收到的信号
* stop方法用来取消监听

##### 1.监听全部信号

```go
package signal

import (
	"fmt"
	systemOs "os"
	"os/signal"
)

func ListenAllSignal()  {
	// 创建channel
	signalChan := make(chan systemOs.Signal)

	// 监听所有信号
	signal.Notify(signalChan)

	// 阻塞直到有信号传入
	fmt.Print("启动")

	signals := <- signalChan
	fmt.Println("停止:",signals)
}
```

启动 go run example-1.go 启动

ctrl+c退出,输出 退出信号 interrupt

kill pid 输出 退出信号 terminated

##### 2.监听指定信号

```go
package signal

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"
)

func Accept()  {
	signals := make(chan os.Signal)

	//Signals 最新信号调用方式
	//
	//const (
	//    SIGABRT   = Signal(0x6)
	//    SIGALRM   = Signal(0xe)
	//    SIGBUS    = Signal(0x7)
	//    SIGCHLD   = Signal(0x11)
	//    SIGCLD    = Signal(0x11)
	//    SIGCONT   = Signal(0x12)
	//    SIGFPE    = Signal(0x8)
	//    SIGHUP    = Signal(0x1)
	//    SIGILL    = Signal(0x4)
	//    SIGINT    = Signal(0x2)
	//    SIGIO     = Signal(0x1d)
	//    SIGIOT    = Signal(0x6)
	//    SIGKILL   = Signal(0x9)
	//    SIGPIPE   = Signal(0xd)
	//    SIGPOLL   = Signal(0x1d)
	//    SIGPROF   = Signal(0x1b)
	//    SIGPWR    = Signal(0x1e)
	//    SIGQUIT   = Signal(0x3)
	//    SIGSEGV   = Signal(0xb)
	//    SIGSTKFLT = Signal(0x10)
	//    SIGSTOP   = Signal(0x13)
	//    SIGSYS    = Signal(0x1f)
	//    SIGTERM   = Signal(0xf)
	//    SIGTRAP   = Signal(0x5)
	//    SIGTSTP   = Signal(0x14)
	//    SIGTTIN   = Signal(0x15)
	//    SIGTTOU   = Signal(0x16)
	//    SIGUNUSED = Signal(0x1f)
	//    SIGURG    = Signal(0x17)
	//    SIGUSR1   = Signal(0xa)
	//    SIGUSR2   = Signal(0xc)
	//    SIGVTALRM = Signal(0x1a)
	//    SIGWINCH  = Signal(0x1c)
	//    SIGXCPU   = Signal(0x18)
	//    SIGXFSZ   = Signal(0x19)
	//)
	signal.Notify(signals, os.Interrupt, os.Kill, syscall.Signal(0xa), syscall.Signal(0xc))

	// 阻塞知道有信号传入
	fmt.Println("启动")

	// 阻塞直到信号传入
	s := <-signals

	fmt.Println("退出信号", s)
}
```

启动 go run example-2.go 启动

ctrl+c退出,输出 退出信号 interrupt

kill pid 输出 退出信号 terminated

kill -USR1 pid 输出 退出信号 user defined signal 1

kill -USR2 pid 输出 退出信号 user defined signal 2

##### 3.优雅退出go守护进程

```go
package signal

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func ExitSignal()  {
	// 创建监听退出chan
	signals := make(chan os.Signal)

	signal.Notify(signals, syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT, syscall.Signal(0xa), syscall.Signal(0xc))

	go func() {
		for s := range signals {
			switch s {
			case syscall.SIGHUP, syscall.SIGINT, syscall.SIGTERM, syscall.SIGQUIT:
				fmt.Println("退出", s)
				exiFunc()
			case syscall.Signal(0xa):
				fmt.Println("usr1", s)
			case syscall.Signal(0xc):
				fmt.Println("usr2", s)
			default:
				fmt.Println("other", s)
			}
		}
	}()

	fmt.Println("进程启动...")
	sum := 0
	for {
		sum++
		fmt.Println("sum:", sum)
		time.Sleep(time.Second)
	}

}

func exiFunc() {
	fmt.Println("开始退出...")
	fmt.Println("执行清理...")
	fmt.Println("结束退出...")
	os.Exit(0)
}
```

```
sum: 1
sum: 2
sum: 3
sum: 4
退出 interrupt
开始退出...
执行清理...
结束退出...
```












