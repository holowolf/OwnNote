## sync.Mutex互斥锁

> Mutex是一个互斥锁, 可以创建为其他结构体的字段; 零值为解锁状态. Mutex类型的锁和线程无关,
> 可以由不同线程加锁和解锁

#### Method

func (m *Mutex) Lock()

```go
func (m *Mutex) Lock()
```
Lock 方法锁住了m, 如果m已经加锁, 则阻塞直到m解锁.

func (*Mutex) Unlock

```go
func (m *Mutex) Unlock()
```
Unlock 方法解锁m, 如果m未加锁导致运行时错误.

### 注意
* 在一个goroutine获得Mutex后, 其他goroutine只能等到这个goroutine释放该Mutex
* 使用Lock()加锁后, 不能再继续对其加锁,知道利用Unlock()解锁后再加锁
* 在Lock()之前使用Unlock()会导致panic异常
* 已经锁定的Mutex并不与特定的goroutine相关联, 这样可以利用goroutine对其加锁, 再利用其他goroutine对其解锁.
* 在同一个goroutine中的Mutex解锁之前再次进行加锁, 会导致死锁
* 适用于读写不确定, 并且只有一个读或者写的场景

示例
```go
func UseMutex()  {
	var mutex sync.Mutex
	wait := sync.WaitGroup{}
	fmt.Println("Locked")
	mutex.Lock()
	for i:= 1; i <= 3; i++ {
		wait.Add(1)
		go func(i int) {
			fmt.Println("Not lock:", i)
			mutex.Lock()
			fmt.Println("Lock:", i)

			time.Sleep(time.Second)

			fmt.Println("Unlock:",i)
			mutex.Unlock()

			defer wait.Done()
		}(i)
	}
	time.Sleep(time.Second)
	fmt.Println("Unlocked")
	mutex.Unlock()
	wait.Wait()
}
```

输出
```
Locked
Not lock: 3
Not lock: 1
Unlocked
Lock: 3
Unlock: 3
Lock: 1
Unlock: 1
Lock: 2
Unlock: 2
```


## RWMutex (读写锁)

* RWMutex 是单写多读锁, 该锁可以加多个读锁和一个写锁
* 读锁占用的情况下会阻止写, 不会阻止读, 多个goroutine可以同时获取读锁
* 写锁会阻止其他goroutine(无论是读和写)进来, 整个锁由该goroutine独占
* 适用于读多写少的场景

#### Lock() 和 Unlock()

* Lock()加写锁, Unlock()解写锁
* 如果在加写锁之前已经有其他的读锁和写锁, 则Lock()会阻塞直到该锁可用,
为确保该锁可用,已经阻塞的Lock()调用会从获得的锁中排出新的读取器,即写锁权限高于读锁,
有写锁时优先进行写锁.
* 在Lock()之前使用Unlock()会导致panic异常


#### RLock() 和 RUnlock()

* RLock()加读锁, RUnlock解读锁
* RLock()加读锁时, 如果存在写锁, 则无法加读锁;
当只有读锁没有写锁时,可以加读锁,读锁可以加载多个.
* RUnlock()解读锁, RUnlock()撤销单词RLock()调用, 对于其他同时存在的读锁则没有效果
* 在没有读锁的情况下调用RUnlock()会导致panic错误
* RUnlock()的个数不得多余RLock(), 否则会导致panic错误

```go
func UseRLockAndUnRLock()  {
	var mutex *sync.RWMutex
	mutex = new(sync.RWMutex)
	fmt.Println("Lock the lock")
	mutex.Lock()
	fmt.Println("The lock is locked")

	channels := make([]chan int, 4)

	for i := 0; i < 4; i++ {
		channels[i] = make(chan int)
		go func(i int, c chan int) {
			fmt.Println("Not read lock: ", i)
			mutex.RLock()
			fmt.Println("Read Locked: ", i)
			fmt.Println("Unlock the read lock: ", i)
			time.Sleep(time.Second)
			mutex.RUnlock()
			c <- i
		}(i, channels[i])
	}
	time.Sleep(time.Second)
	fmt.Println("Unlock the lock")
	mutex.Unlock()
	time.Sleep(time.Second)
	for _, c := range channels {
		<-c
	}
}
```

输出

```
Lock the lock
The lock is locked
Not read lock:  3
Not read lock:  2
Not read lock:  0
Unlock the lock
Read Locked:  1
Unlock the read lock:  1
Read Locked:  3
Unlock the read lock:  3
Read Locked:  2
Unlock the read lock:  2
Read Locked:  0
Unlock the read lock:  0
```