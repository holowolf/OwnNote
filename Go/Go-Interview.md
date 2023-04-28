## Go-Interview 面试资料

1.go的调度

A: Go调度相关的四个基本单元是g、m、p、schedt.
g是协程任务信息单元, m实际执行体, p是本地资源池和g任务池, schedt是全局资源池和任务池.
具体调度进行为GMP方案.

2.go struct能不能比较
A: 存在两种情况
* 同一个struct的两个实例能不能比较?
* 两个不同的struct的实例能不能比较?

* 可比较类型: Integer, Floating-point, String, Boolean, Complex(复数型), Pointer, Channel, Interface, Array
* 不可比较类型: Slice, Map, Function

> 同一个struct的两个实例能不能比较

同一struct的两个实例可比较也不可比较, 当结构不包含不可直接比较的成员变量时可直接比较, 否则不可直接比较.


> 两个不同的struct的实例能不能比较

可以通过强制转换来进行比较, 如果成员变量中包含不可比较成员变量, 即使可以强制转换, 也不可以比较.

3.context包的用途

A: 主要的作用是控制Goroutine的声明周期. 当一个计算任务被Goroutine承接了之后, 由于某种原因(超时, 或者强制退出)我们希望中止这个goroutine的计算任务, 那么就用得到这个Context了.
通常翻译为上下文,上下层的信息传递功能.
Context有四种结构,CancelContext, TimeoutContext, DeadLineContext, ValueContext的使用决定了Context包的用途.

1.RPC调用, RPC函数中第一个参数是一个CancelContext, 用于判断ctx的done和当前的rpc调用哪个先结束.
2.PipeLine模式, 通过判断传递ctx.Done()函数进行控制.
3.超时请求, Context包也实现了这个需求:timeCtx. 具体实例化的方法是WithDeadLine和WithTimeout.
4.Http请求request互相传递数据, Context提供了valueCtx的数据结构


4.Go语言GC(垃圾回收)的工作原理

A: 最常见的垃圾回收算法有标记清楚(Mark-Sweep)和引用计数(Reference Count),Go语言采用的标记清除算法.
并在此基础上使用三色标记法和写屏障技术, 提高了效率.


5.主协程如何等其余协程完再操作

A: 使用channel进行通信, 或者 context, select, 或者sync.WaitGroup来进行并发安全阻塞.


6.slice，len，cap，共享，扩容

A: 切片的底层由数据, len, cap组成 通过append扩容添加连续内存块或者生成更大的数组并复制.

7.map如何顺序读取

A: map是哈希表结构无法顺序读取,可以通过将map中的key进行排序为slice结构顺序,通过遍历有序的slice来读取map.

8.什么是协程

A: Goroutine是与其他函数或方法同时运行的函数或方法.Goroutines可以被认为是轻量级的线程.
与线程相比,创建Goroutine的开销很小.Go应用程序同时运行数千个Goroutine是非常常见的做法.

9.高效拼接字符串

A: Go语言中,字符串是只读的,也就意味着每次修改操作都会创建一个新的字符串.
如果需要拼接多次应用使用 strings.Builder, 最小化拷贝次数.

10.Golang Rune类型

A: ASCII码只需要7bit就可以完整的表示, 但只能表示英文字母内的128个字符, 为了表示世界上大部分的文字系统,发明了Unicode是ASCII超集, 并分配标准码.
在Go语言中称之为rune,是int32类型的别名.Golang中字符串底层是byte(8 bit), 而非rune(32bit). 
```
fmt.Println(len("Go语言")) // 8
fmt.Println(len([]rune("Go语言"))) // 4
```

11.defer的执行顺序

A: defer是栈内存的数据结构, 遵循后进先出的规则.


12.init()函数是什么时候执行的

A: init()函数式Go程序初始化的一部分. Go程序初始化优先于main函数, 由runtime初始化每个导入的包, 初始化顺序不是按照从上到下的导入顺序,而是按照解析的依赖关系,没有依赖的包最先初始化.
每个包首先初始化作用于的常量和变量(常量优先于变量),然后执行包的init()函数.同一个包,甚至是同一源文件可以有多个init()函数.
init()函数没有入参和返回值,不能被其他函数调用,同一个包内多个init()函数的执行顺序不作保证.
import-->const-->var-->init()-->main()


13.Go 语言的局部变量分配在栈上还是堆上？

A: 由编译器决定. Go语言编译器会自动把一个变量放在栈还是堆上,编译器会做逃逸分析(escape analysis),当发现变量的作用域没有超出函数范围,就可以在栈上,反之则必须分配在堆上.

```
func foo() *int {
	v := 11
	return &v
}

func main() {
	m := foo()
	println(*m) // 11
}
```

foo()函数中,如果v分配在栈上,foo函数返回时,&v就不存在了,但在这段函数能执行.Go编译器发现v的引用脱离了foo的作用域,会将分配在堆上.因此,main函数中能够正常访问该值.

14.无缓冲的channel和有缓冲的channel的区别

对于无缓冲的channel,发送方将阻塞该信道,知道接收方从该信道接收到数据为止,而接收方也将阻塞该信道,直到发送方将数据发送到该信道中为止.
对于有缓存的channel,发送方在没有空插槽(缓冲区使用完)的情况下阻塞,而接收方在信道为空的情况下阻塞.

eg：

```
func main() {
	st := time.Now()
	ch := make(chan bool)
	go func ()  {
		time.Sleep(time.Second * 2)
		<-ch
	}()
	ch <- true  // 无缓冲，发送方阻塞直到接收方接收到数据。
	fmt.Printf("cost %.1f s\n", time.Now().Sub(st).Seconds())
	time.Sleep(time.Second * 5)
}
```

```
func main() {
	st := time.Now()
	ch := make(chan bool, 2)
	go func ()  {
		time.Sleep(time.Second * 2)
		<-ch
	}()
	ch <- true
	ch <- true // 缓冲区为 2，发送方不阻塞，继续往下执行
	fmt.Printf("cost %.1f s\n", time.Now().Sub(st).Seconds()) // cost 0.0 s
	ch <- true // 缓冲区使用完，发送方阻塞，2s 后接收方接收到数据，释放一个插槽，继续往下执行
	fmt.Printf("cost %.1f s\n", time.Now().Sub(st).Seconds()) // cost 2.0 s
	time.Sleep(time.Second * 5)
}
```

15.什么是协程泄露(Goroutine Leak)

A:泄露的原因大多数集中在:
* Goroutine 内正在进行 channel/mutex 等读写操作, 但由于逻辑问题, 某些情况下会被一直阻塞.

channel使用不当
Goroutine+Channel 是最经典的组合,因此不少泄露都出于此.

发送不接收
eg:
```
func main() {
    for i := 0; i < 4; i++ {
        queryAll()
        fmt.Printf("goroutines: %d\n", runtime.NumGoroutine())
    }
}

func queryAll() int {
    ch := make(chan int)
    for i := 0; i < 3; i++ {
        go func() { ch <- query() }()
     }
    return <-ch
}

func query() int {
    n := rand.Intn(100)
    time.Sleep(time.Duration(n) * time.Millisecond)
    return n
}
```

输出结果

```
goroutines: 3
goroutines: 5
goroutines: 7
goroutines: 9
```

在这个例子中,我们调用多次queryAll方法, 并在for循环中利用Goroutine调用了query方法.
其重点在于调用query方法后的结果会写入ch变量中,接收成功后再返回ch变量.
最后可以看到输出的goroutines的数量是不断增加的,每次多2个.也就是每调用一次就会泄露Goroutine.
原因在于channel均已发送(每次发送3个),但是在接收端并没有接收完全(只返回1个ch),所诱发的Goroutine泄露.

接收不发送
eg:
```
func main() {
    defer func() {
        fmt.Println("goroutines: ", runtime.NumGoroutine())
    }()

    var ch chan struct{}
    go func() {
        ch <- struct{}{}
    }()
    
    time.Sleep(time.Second)
}
```

输出结果

```
goroutines:  2
```

在这个例子中,与"发送不接收"两者是相对的,channel接收了值,但是不发送的话,同样会造成阻塞.
但是在实际的业务场景中,一般更复杂.基本一大堆业务逻辑里,有一个channel的读写操作出现了问题,自然就阻塞了.

nil channel
eg：
```
func main() {
    defer func() {
        fmt.Println("goroutines: ", runtime.NumGoroutine())
    }()

    var ch chan int
    go func() {
        <-ch
    }()
    
    time.Sleep(time.Second)
}
```

输出结果

```
goroutines:  2
```

在这个例子中, 可以得知channel如果忘记初始化, 那么无论你是读, 还是写操作, 都会造成阻塞.

正常初始化代码:

```
ch := make(chan int)
go func() {
    <-ch
}()
ch <- 0
time.Sleep(time.Second)
```

调用make函数进行初始化


* Goroutine 内的业务逻辑进入长时间等待, 有不断新增的 Goroutine 进入等待或进入死循环, 资源一直无法释放

慢等待
eg:
```
func main() {
    for {
        go func() {
            _, err := http.Get("https://www.xxx.com/")
            if err != nil {
                fmt.Printf("http.Get err: %v\n", err)
            }
            // do something...
    }()

    time.Sleep(time.Second * 1)
    fmt.Println("goroutines: ", runtime.NumGoroutine())
  }
}
```

输出结果:

```
goroutines:  5
goroutines:  9
goroutines:  13
goroutines:  17
goroutines:  21
goroutines:  25
...
```

这个例子中,展现了一个Go语言中景点的事故场景. 也就是一般我们会在应用程序中去调用第三方服务的接口.
但是三方接口很慢,响应很慢.恰好,Go语言中默认的`http.Client`是没有设置超时时间的.
因此就会导致一直阻塞,Goroutine自然也就持续暴涨,不断泄露,最终占满资源导致事故.

在Go工程中,我们一般建议至少对`http.Client`设置超时时间:
```
httpClient := http.Client{
    Timeout: time.Second * 15,
}
```

并且要做限流,熔断等措施,以防突发的流量造成的依赖坍塌.

* 死锁(dead lock)

两个或两个以上的协程执行任务过程中由于竞争资源或彼此通信造成阻塞,这种情况下会导致协程被阻塞,不能退出.
sync.Mutex加锁忘记解锁或使用不当.

互斥锁忘记解锁
eg:
```
func main() {
    total := 0
    defer func() {
        time.Sleep(time.Second)
        fmt.Println("total: ", total)
        fmt.Println("goroutines: ", runtime.NumGoroutine())
 }()

    var mutex sync.Mutex
    for i := 0; i < 10; i++ {
        go func() {
            mutex.Lock()
            total += 1
        }()
    }
}
```

输出结果:

```
total:  1
goroutines:  10
```

在这个例子中, 第一个互斥锁`sync.Mutex`加锁了,但它可能在处理业务,或者忘记Unlock了.
因此带催产后面的所有`sync.Mutex`想加锁,却因未释放又都阻塞住了.一般在Go工程中建议

```
var mutex sync.Mutex
for i := 0; i < 10; i++ {
    go func() {
        mutex.Lock()
        defer mutex.Unlock()
        total += 1
    }()
}
```

同步锁使用不当

```
func handle(v int) {
    var wg sync.WaitGroup
    wg.Add(5)
    for i := 0; i < v; i++ {
        fmt.Println("脑子进煎鱼了")
        wg.Done()
    }
    wg.Wait()
}

func main() {
    defer func() {
        fmt.Println("goroutines: ", runtime.NumGoroutine())
    }()

    go handle(3)
    time.Sleep(time.Second)
}
```

在这个例子中,我们调用了同步编排`sync.WaitGroup`,模拟了一遍我们会从外部传入循环遍历的控制变量.
但由于`wg.Add`的数量与`wg.Done`数量并不匹配.因此调用`wg.Wait`方法后一直阻塞等待.

建议写法:

```
var wg sync.WaitGroup
for i := 0; i < v; i++ {
    wg.Add(1)
    defer wg.Done()
    fmt.Println("脑子进煎鱼了")
}
wg.Wait()
```

排查方案:

我们可以调用`runtime.NumGoroutine`方法来获取Goroutine的运行数量,进行前后一比较,就能知道有没有泄露了.
但在业务服务的运行场景中,Goroutine内导致的泄露,大多数处于生产,测试环境,因此更多的是使用 PProf：
```
import (
    "net/http"
     _ "net/http/pprof"
)

http.ListenAndServe("localhost:6060", nil))
```

只要我们调用 `http://localhost:6060/debug/pprof/goroutine?debug=1，PProf` 会返回所有带有堆栈跟踪的Goroutine列表.


16.Go可以限制运行时操作系统线程的数量吗

可以使用环境变量`GOMAXPROCS`或`runtime.GOMAXPROCS(num int)`设置, 例如:
```
runtime.GOMAXPROCS(1) // 限制同时执行Go代码的操作系统线程数为 1
```
从官方文档的解释可以看出,`GOMAXPROCS`限制的同时执行用户态Go代码的操作系统线程的数量,但是对于被系统调用阻塞的线程数量是没有限制的.
`GOMAXPROCS`的默认值等于CPU的逻辑核数,同一时间,一个核只能绑定一个线程,然后运行被调度的协程.因此对于CPU密集型的任务,

17.Go进程,线程和Go协程之间的区别

A:
* 进程包含计算机指令,用户数据和系统数据的程序执行环境,以及包含其运行时获得的其他类型资源.
* 线程相对于进程是更加小巧而轻量的实体,线程由进程创建且包含自己的控制流和栈,线程和进程的区别在于:进程是正在运行的二进制文件,而线程是进程的子集.
* Goroutine是Go程序并发执行的最小单元,因为Goroutine不是像Unix进程那样是自治的实体,Goroutine生活在Unix进程的线程中.Goroutine的主要优点是非常轻巧,运行成千上万都没有问题.

总结: Goroutine比线程更轻量,而线程比进程更轻量. 实际上,一个进程可以有多个线程已经许多Goroutine,而Goroutine需要一个进程的环境才能存在.
因此为了创建一个Goroutine，你需要一个进程且这个进程至少有一个线程--Unix负责进程和线程管理,而Go工程师只需要处理Goroutine,这极大降低了开发成本.



17.new 和 make关键字区别

Go语言中new和make都是用来内存分配的原语(allocation primitives).简单的来说,new只分配内存,make用于slice,map,和channel的初始化.

new(T)函数式一个分配内存的内建函数. 我们都知道对于一个已经存在的变量,可对其指针进行赋值.

```
var p int
var v *int
v = &p
*v = 11
fmt.println(*v)
```

那么已经存在的变量能直接赋值吗?

```
var v *int
*v = 8
fmt.Println(*v)
```

结果报错

```
panic: runtime error: invalid memory address or nil pointer dereference
[signal 0xc0000005 code=0x1 addr=0x0 pc=0x48df66]
```

解决Go提供new来初始化一个地址解决

```
var v *int
v = new(int)
*v = 8
fmt.Println(*v)
```

```
var v *int
fmt.Println(v) //<nil>
v = new(int)
fmt.Println(*v) //
fmt.Println(v) // 0xx0044488
```

我们可以看出初始化一个指针变量,其值为nil,nil的值不能直接赋值.通过new其返回一个指向新分配的类型为int的指针,指针值为0xx0044488,这个指针指向的内容值为零(zero value).

```
type Name struct {
    p string
}
var av *[5]int
var iv *int
var sv *string
var tv *Name

av = new([5]int)
fmt.println(*av) // [0,0,0,0,0]
iv = new(int)
fmt.println(*iv) // 0
sv = new(string)
fmt.Println(sv) // 
tv = new(Name)
fmt.println(tv) // {}
```
上面是普通类型new()处理后如何赋值的,这里再讲一下复合类型(数组,slice,map,channel等),new()处理过后,如何赋值.

数组
```
var a [5]int
fmt.Printf("a: %p %#v \n", &a, a)//a: 0xc04200a180 [5]int{0, 0, 0, 0, 0} 
av := new([5]int)
fmt.Printf("av: %p %#v \n", &av, av)//av: 0xc000074018 &[5]int{0, 0, 0, 0, 0}
(*av)[1] = 8
fmt.Printf("av: %p %#v \n", &av, av)//av: 0xc000006028 &[5]int{0, 8, 0, 0, 0}
```

slice
```
var a *[]int
fmt.Printf("a: %p %#v \n", &a, a) //a: 0xc042004028 (*[]int)(nil)
av := new([]int)
fmt.Printf("av: %p %#v \n", &av, av) //av: 0xc000074018 &[]int(nil)
(*av)[0] = 8
fmt.Printf("av: %p %#v \n", &av, av) //panic: runtime error: index out of range
```

map
```
var m map[string]string
fmt.Printf("m: %p %#v \n", &m, m)//m: 0xc042068018 map[string]string(nil) 
mv := new(map[string]string)
fmt.Printf("mv: %p %#v \n", &mv, mv)//mv: 0xc000006028 &map[string]string(nil)
(*mv)["a"] = "a"
fmt.Printf("mv: %p %#v \n", &mv, mv)//这里会报错panic: assignment to entry in nil map
```

channel
```
cv := new(chan string)
fmt.Printf("cv: %p %#v \n", &cv, cv)//cv: 0xc000074018 (*chan string)(0xc000074020) 
//cv <- "good" //会报 invalid operation: cv <- "good" (send to non-chan type *chan string)
```

通过上面示例可以看到数组通过new处理,数组av初始化零值,数组虽然是复合类型,但不是引用类型,其他slice、map、channel类型也属于引用类型,
go会给引用类型初始化为nil,nil是不能直接赋值的.并且不能new分配内存.无法直接赋值.那么用make函数处理会怎么样呢?

make

```
av := make([]int, 5) 
fmt.Printf("av: %p %#v \n", &av, av) // av: 0xc0000044e0 []int{0, 0, 0, 0, 0}
av[0] = 1
fmt.Printf("av : %p %#v \n", &av, av) // av : 0xc0000044e0 []int{1, 0, 0, 0, 0}
mv := make(map[string] string)
fmt.Printf("mv %p %#v \n", &mv, mv) // mv 0xc000006030 map[string]string{}
mv["m"] = "m"
fmt.Printf("mv: %p %#v \n", &mv, mv) // mv: 0xc000006030 map[string]string{"m":"m"}
chv := make(chan string)
fmt.Printf("chv : %p %#v \n", &chv, chv) // chv : 0xc000006038 (chan string)(0xc000040120)
go func(message string) {
    chv <- message
}("ping")
fmt.Println(<-chv) // ping
close(chv)
```

make 不仅可以开辟一个内存,还能给这个内存类型初始化零值

它和new还能配合使用
eg:
```
var mv *map[string]string
fmt.Printf("mv:%p%#v \n", &mv, mv)  // mv:0xc000006028(*map[string]string)(nil)
mv = new(map[string]string)
fmt.Printf("mv:%p%#v \n", &mv, mv)  // mv:0xc000006028&map[string]string(nil)
*mv = make(map[string]string)
fmt.Printf("mv:%p%#v \n", &mv, mv)  // mv:0xc000006028&map[string]string{}
(*mv)["a"] = "a"
fmt.Printf("mv %p %#v \n", &mv, mv) // mv 0xc000006028 &map[string]string{"a":"a"}
```

通过new给指针变量mv分配一个内存,并赋值给其内存地址.Map是引用类型,其零值为nil, 使用make初始化map, 然后变量就能使用*给指针变量mv赋值了.

* make和new都是golang来分配内存的内建函数,且在堆上分配内存,make即分配内存,也初始化内存.new只是将内存清零,并没有初始化内存.
* make返回的还是引用类型本身,而new返回的是指向类型的指针.
* make只能用来分配初始化slice, map, channel的数据;new可以分配任意类型的数据.


18.Golang中的锁机制

Golang 中有两种锁机制 `Mutex`(互斥锁)和`RWMutex`(读写锁).


* 知识点1 : 信号量  
信号量是Edsger Dijkstra(最短路劲算法作者)发明的数据结构, 在解决多种同步问题时很有用.本质是一个整数,并关联两个操作:

申请acquire(也称为wait, decrement 或 P 操作)
申请release(也称为signal, increment 或 V 操作)

acquire操作将信号量减1,结果结果值为负则线程阻塞,且直到其他线程进行了信号量累加为正数才能恢复.如果结果为正数,线程则继续执行.

release操作将信号量加1,如存在被阻塞的线程,此时他们中一个线程将解除阻塞.

* 知识点2 : 锁的意义
```
type Locker interface {
    Lock()
    Unlock()
}
```

在golang中如果实现了Lock()和Unlock(),那么它就可以被称为锁.

* 知识点3 : 锁的自旋

* 知识点4 : cas算法


读写锁RWMutex

```
type RWMutex struct {
    w Mutext
    writeSem uint32
    readerSem uint32
    readerCount uint32
    readerWait uint32
}
```

结构中发现读写锁内部包含一个`w Mutex`互斥锁  
注释页很明确,这个锁的目的就是控制多个写入操作和并发执行.

RWMutex属性:

`writeSem`是写入操作的信号量  
`readerSem`是读操作的信号量  
`readerCount`是当前读操作的个数  
`readerWait`当前写入操作需要等待读操作解锁的个数  

RWMutex方法:

##### RLocker()

```
func (rw *RWmutex) RLocker() {
    return (*rlocker)(rw)
}
```

此方法返回locker对象.


##### RLock()

每当协程需要读锁的时候,atomic就将readerCount+1  
但需要注意的是当readerCount + 1之后的值 < 0的时候, 那么将会调用runtime_Semacquire方法.

```
func runtime_Semacquire(s *uint32)
```

此方法是一个runtime方法,会一直等待出现s > 0的时候,当先reader_count + 1 为负数时那么就会被等待,所以当有写操作出现的时候,那么读操作就被等待.


##### Lock()

操作前面属性中的互斥锁,目的就是处理并发情况,写锁只有一把,因为互斥锁只有一个人能拿到所以写锁只有一个人能拿到.


##### RUnlock()

操作为将之前的readerCount - 1, 没有读锁而去释放读锁就会导致异常.


##### Unlock()

写锁的释放需要恢复readerCount, runtime_Semrelease 通过for循环通知那些持有写锁等待的协程,通过信号量runtime_Semrelease可以懂了.最后还需要释放互斥锁,让别的协程可以开始抢锁.


互斥锁 Mutex

互斥锁是公平锁

互斥锁有两种模式: 正常模式 和 饥饿模式

在正常模式下等待获取所的goroutine会以一个先进先出的队列进行,但是被唤醒的等待着并不能代表它已经拥有这个mutex锁,它需要与新到达的goroutine争夺mutex锁.
新来的goroutine有一个优势,它们在cpu上运行了,所以抢到的可能性要大一些,所以一个被唤醒的等待者有很大可能抢不过.在这样的情况下,被唤醒的等待者在队列的头部,如果一个等待者抢锁超过1ms失败了,就会切换为饥饿模式.

在饥饿模式下,mutex锁会直接由解锁的goroutine交给队列头部的等待者.
新来的goroutine不能尝试去获取锁,即使可能根本就没goroutine在持有锁,并且不能尝试自旋.取而代之的是它们只能排到队伍尾巴上等着.

如果一个等待者获取了锁,并且遇到了下面两种情况之一,就恢复成正常工作模式.
情况1: 它是最后一个队列中的等待者
情况2: 它的等待时间小于1ms

正常模式下,即使有很多阻塞的等待者,有更好的表现,因为一轮能多次获得锁的机会.饥饿模式是为了避免哪些一直在队尾的倒霉蛋.

互斥锁的结构

```
type Mutex struct {
    state int32
    sema uint32
}
```

state 将一个32位的整数拆分为:  
当前阻塞的goroutine数(29位)  
饥饿状态(1位, 0为正常模式; 1为饥饿模式)  
唤醒状态(1位, 0未唤醒; 1已唤醒)  
锁状态(1位,0可用; 1占用)

sema: 信号量

##### Lock()获取锁

Lock()方法的核心目的就是抢锁, 如果不是处于饥饿模式,那么就会进行自旋操作,然后不断循环, 如果是正常模式如欧冠拿到了就会立即上锁并往下执行,如果在状态标记位表示处于饥饿模式,并且锁被占用情况,那么就需要多队列排队.

##### Unlock()释放锁

如果是正常模式,如果没有等待的goroutine或goroutine已经解锁完成的情况就直接返回了.如果有等待的goroutine那就通过信号量去唤醒runtime_Semrelease(这里是false),同时操作队列-1.

如果是饥饿模式就直接唤醒.

atomic 操作 LOCK 和 CAS 操作都是利用底层汇编实现的.

互斥锁总结

在无冲突的时候,锁能拿就会拿到,出现冲突就会进行自旋解决(自旋一般会解决),如果无法解决通, 就通过信号量解决,同时如果正常模式就是我们所说的抢占式,非公平,如果是饥饿模式,就是我们说的排队,公平,公平防止有些G一直抢不到.

锁的设计并不复杂,主要涉及cas和处理读写状态的信号量通知,对于那些位操作,是巧妙的设计维护了一个完整体系的艺术.


19.Golang slice

##### slice结构

切片是数组的封装

```
type slice struct {
    array unsafe.Pointer
    len int
    cap int
}
```

* 其中array是一个指针指向数组底层
* len代表slice的长度
* cap代表slice的容量


##### slice的长度和容量

数组在创建的时候只能创建固定大小的数组, 而当slice不断往其中添加元素的时候,势必会遇到大小不够的情况,如果每次添加都不够,那么每次都要创建新的数组,那会相当浪费时间和资源,所以当不够的时候会创建大一些,cap代表整体的一个容量,len代表用到了第几个.

##### slice的扩容

大概机制:当len<1024的时候每次2倍,当len>1024的时候每次1.25倍.

在此基础上,使用`内存对齐`的方式保证cpu平台的兼容性和对齐后的内存访问的性能提高(对齐后的内存仅访问一次,未对齐的内存需要访问两次)


##### 创建slice

使用make进行初始化

```
make([]int, 10, 32)
```

##### slice删除一个元素

```
sli = append(sli[:3], sli[4:]...)
```
由于底层是数组结构所以删除较为麻烦

##### slice中的坑

首先golang中只有值传递,没有引用传递.

* reslice后续操作会对原来的slice造成影响
* 如果经过append之后,那么由于扩容的时候重新分配内存,如果涉及扩容之后,那么就不会对原有slice造成影响
* 如果作为函数的参数传递的是数组,因为是值传递,所以函数内部的修改,不会对外汇的变量产生影响,如果是slice传递,那么传递的是指针,所以会修改外部的变量.
* 同时因为是值传递,形参的重新赋值是不会对外部的变量造成影响.

总结: 创建slice的时候最好初始化好需要用到的容量,避免扩容.注意slice作为形参进行传递的时候,还有slice进行append的时候注意.


20.golang Context

golang中的context类似于spring中 ApplicationContext

context的定义: context是用于在多个goroutines之间传递信息的媒介.

##### 传递信息

```
func main() {
    ctx := context.Background()
    ctx = context.WithValue(ctx, "xxx", "123")
    value := ctx.Value("xxx")
    fmt.Println(value)
}
```

通过`context.WithValue`方法设置,key-value然后通过`ctx.Value`方法取值即可.


##### 关闭goroutine

在程序运行时,经常需要开一个goroutine处理对应的任务,特别是一些循环一直处理的情况,这些goroutine需要知道自己什么时候需要停止.
常见的方式是通过channel接收一个关闭的信号,如果通过context如果实现.

```
func main() {
    ctx, _ := context.WithTimeout(context.Background(), time.Second * 3)
    go func() {
        go1(ctx)
    }()
    go func() {
        go2(ctx)
    }()
    time.Sleep(time.Second * 5)
}

func go1(ctx context.Context) {
    for {
        fmt.Println("1 正在工作")
        select {
        case <-ctx.Done():
            fmt.Println("1 停止工作")
            return
        case <-time.After(time.Second):
        }
    }
}

func go2(ctx context.Context) {
    for {
        fmt.Println("2 正在工作")
        select {
        case <-ctx.Done():
            fmt.Println("2 停止工作")
            return
        case <-time.After(time.Second):
        }
    }
}
```

通过context.WithTimeout创建一个3秒后自动取消的context;
所有的工作goroutine监听ctx.Done()的信号;
收到信号就证明需要取消任务;

##### context源码解析

context.TODO()

创建占位的context,可能在写程序的过程中还未确定后期context的作用,暂时占位.

context.Background()


