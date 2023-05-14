## 协程池任务调度

```go
package pool_test

import (
	"fmt"
	"time"
)

// 定义一个Task类型
type Task struct {
	f func() error // 一个Task里面应该有一个具体的业务
}

// 创建一个Task任务
func NewTask(arg func() error) *Task {
	t := Task {
		f: arg,
	}
	return &t
}

// Task也需要一个执行业务的方法
func (t *Task) Execute() {
	// 调用任务中已经绑定号的业务方法
	t.f()
}


// -------------------- 有关协程池pool 的功能 --------------------------

// 定义一个pool的协程池类型
type Pool struct {
	// 对外的Task入口 EntryChannel
	EntryChannel chan *Task

	// 内部Task队列 JobsChannel
	JobsChannel chan *Task

	// 最大的Work数量
	workerNum int
}

// 创建Pool函数
func NewPool(cap int) *Pool {
	// 创建一个pool
	p := Pool {
		EntryChannel:make(chan *Task),
		JobsChannel:make(chan *Task),
		workerNum:cap,
	}
	// 返回一个pool
	return &p
}

// 协程池创建一个Worker 并且让这个Worker去工作
func (p *Pool) worker(workerId int) {
	// 一个work具体的工作

	// 永久从JobsChannel去取任务
	for task := range p.JobsChannel {
		// task 就是当前worker从JobsChannel中拿到的任务
		// 一旦取到任务, 执行这个任务
		task.Execute()
		fmt.Println("workerId ... -> ", workerId, "执行完毕!")
	}

}


// 让协程池 开始真正的工作 协程池一个启动方法
func (p *Pool) run() {
	// 根据worker_num 创建worker工作
	for i := 0; i < p.workerNum; i++ {
		// 每个worker都应该是一个goroutine
		go p.worker(i)
	}

	// 从EntryChannel中去取任务, 将取到的任务, 发送给JobsChannel
	for task := range p.EntryChannel {
		// 一旦有task读到
		p.JobsChannel <- task
	}
}

func business() error {
	// 当前任务的业务, 打印出当前的系统时间
	fmt.Println(time.Now())
	return nil
}

// 主函数 来测试协程池的工作
func ThreadTaskMain() {
	// 创建一些任务
	t := NewTask(business)

	// 创建一个Pool 设置最大的Worker
	p := NewPool(4)

	taskRumNum := 0

	// 将这些任务交给协程池修改
	go func() {
		for {
			// 不断向p中写入任务t, 每个任务就是打印当前时间
			taskRumNum++
			p.EntryChannel <- t
			fmt.Println("当前执行任务数: ", taskRumNum)
		}
	}()

	p.run()
}
```