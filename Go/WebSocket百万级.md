## WebSocket连接


### http简单连接

// 使用http.HandleFunc("/", method) 创建http连接头

// http.ListenAndServe() 监听端口

// hello_world(w http.ResponseWriter, r *http.Request) ResponseWriter 响应的io流 写入对应数据

```go
package simple_web_server

import (
	"io"
	"net/http"
)

func RunSimpleServer() {
	http.HandleFunc("/", hello_world)
	http.ListenAndServe(":19999", nil)
}

func hello_world(w http.ResponseWriter, r *http.Request) {
	io.WriteString(w, "hello world!")
}
```

### websocket简单连接

#### server 端

// 下载gorilla 歌莉娅websocket包 这个包有效和简单化websocket操作

// http.HandleFunc Handle接收web_socket通信在 "/" 根目录

// websocket连接使用 websocket.Upgrader{} 结构体 

// Upgrader{} 结构体放入(http.ResponseWriter *http.Request http.Header) 三个参数

// conn 连接 ReadMessage() 读取信息.  如果没有断开错误 那么将会将从连接对象中获取到的信息打印.

```go
package simple_websocket_server

import (
	"github.com/gorilla/websocket"
	"log"
	"net/http"
)

func RunWebSocket() {
	http.HandleFunc("/", web_socket)
	if err := http.ListenAndServe(":8000", nil); err != nil {
		log.Fatal(err)
	}
}

func web_socket(w http.ResponseWriter, r *http.Request) {
	upGrader := websocket.Upgrader{}
	conn, err := upGrader.Upgrade(w, r, nil)
	if err != nil {
		return
	}

	// Read messages from socket
	for {
		_,msg, err := conn.ReadMessage()
		if err != nil {
			conn.Close()
			return
		}
		log.Printf("msg: %s", string(msg))
	}
}
```

#### client端

// ip 和 conn 两个连接参数获取 对应ip地址和连接数

// url.URL 结构体获取websocket的连接地址

// 将 *websocket.Conn的指针连接地址加入连接数组中

// 当客户端发起的连接关闭时, 在客户端结束时关闭连接.

// 最后 conn.WriteControl 连接控制 连接写入信息 conn.WriteMessage

```go
package web_socket_client

import (
	"flag"
	"fmt"
	"github.com/gorilla/websocket"
	"io"
	"log"
	"net/url"
	"os"
	"time"
)

var (
	ip 			= flag.String("ip", "127.0.0.1", "server IP")
	connections = flag.Int("conn", 1, "number of websocket connections")
)

func RunClient() {
	flag.Usage = func() {
		io.WriteString(os.Stderr, `WebSockets client generator
Example usage: ./client -ip=172.17.0.1 -conn=10
`)
		flag.PrintDefaults()
	}
	flag.Parse()

	u := url.URL{Scheme:"ws", Host: *ip + ":8000", Path: "/"}
	log.Printf("Connecting to %s", u.String())
	var conns[] *websocket.Conn
	for i := 0; i < *connections; i++ {
		c, _, err := websocket.DefaultDialer.Dial(u.String(), nil)
		if err != nil {
			fmt.Println("Failed to connect", i, err)
			break
		}
		conns = append(conns, c)
		// 最后执行
		defer func() {
			c.WriteControl(websocket.CloseMessage, websocket.FormatCloseMessage(websocket.CloseNormalClosure, ""), time.Now().Add(time.Second))
			time.Sleep(time.Second)
			c.Close()
		}()
	}

	log.Printf("Finished initializing %d connections", len(conns))
	tts := time.Second
	if *connections > 100 {
		tts = time.Millisecond * 5
	}

	for {
		for i := 0; i < len(conns); i++ {
			time.Sleep(tts)
			conn := conns[i]
			log.Printf("Conn %d sending message", i)
			if err := conn.WriteControl(websocket.PingMessage, nil, time.Now().Add(time.Second*5)); err != nil {
				fmt.Println("Failed to receive pong : %v", err)
			}
			conn.WriteMessage(websocket.TextMessage, []byte(fmt.Sprintf("Hello from conn %v", i)))
		}
	}
}
```

#### websocket 运行连接

// 逻辑增加服务器进程打开文件的最大数量的软限制

// 可以统计最大的连接数 pprof go tool 记录日志工具

// 多端口运行

```go
package websocket_ulimit

import (
	"github.com/gorilla/websocket"
	"log"
	"net/http"
	"sync/atomic"
	"syscall"
)

var count int64

func websocket_ulimit(w http.ResponseWriter, r *http.Request) {
	upGrade := websocket.Upgrader{}
	conn, err := upGrade.Upgrade(w, r, nil)
	if err != nil {
		return
	}

	n := atomic.AddInt64(&count, 1)
	if n%100 == 0 {
		log.Printf("Total number of connections: %v", n)
	}

	defer func() {
		n := atomic.AddInt64(&count, -1)
		if n%100 == 0 {
			log.Printf("Total number of connections: %v", n)
		}
		conn.Close()
	}()

	// Read messages from socket
	for {
		_, msg, err := conn.ReadMessage()
		if err != nil {
			return
		}
		log.Printf("msg : %s", string(msg))
	}
}


func RunWebsocketUlimit() {
	var rLimit syscall.Rlimit
	if err := syscall.Getrlimit(syscall.RLIMIT_NOFILE, &rLimit); err != nil {
		panic(err)
	}
	rLimit.Cur = rLimit.Max
	if err := syscall.Setrlimit(syscall.RLIMIT_NOFILE, &rLimit);err != nil {
		panic(err)
	}
	// Enable pprof hooks
	go func() {
		if err := http.ListenAndServe("127.0.0.1:6060", nil); err != nil {
			log.Fatal("Pprof failed: %v",err)
		}
	}()
	
	http.HandleFunc("/", websocket_ulimit)
	if err := http.ListenAndServe(":8000", nil);err != nil {
		log.Fatal(err)
	}
}
```

#### Goroutine WebSocket

// Goroutine Websocket

```go
package optimize_ws_goroutines

import (
	"github.com/gorilla/websocket"
	"golang.org/x/sys/unix"
	"log"
	"reflect"
	"sync"
	"syscall"
)

// 定义EventsCollector 主结构体
type eventsCollector struct {
	fd int
	connections map[int]*websocket.Conn
	lock *sync.RWMutex
}

// 创建Event工厂
func MkEventsCollector() (*eventsCollector, error) {
	fd, err := unix.EpollCreate1(0)
	if err != nil {
		return nil, err
	}
	return &eventsCollector{
		fd : fd,
		lock: &sync.RWMutex{},
		connections: make(map[int]*websocket.Conn),
	}, nil
}


// Event工厂 添加方法
func (e *eventsCollector) Add(conn *websocket.Conn) error {
	fd := websocketFd(conn)
	err := unix.EpollCtl(e.fd, syscall.EPOLL_CTL_ADD, fd, &unix.EpollEvent{Events: unix.POLLIN | unix.POLLHUP, Fd: int32(fd)})
	if err != nil {
		return err
	}
	e.lock.Lock()
	defer e.lock.Unlock()
	e.connections[fd] = conn
	if len(e.connections)%100 == 0 {
		log.Printf("Total number of connections: %v", len(e.connections))
	}
	return nil
}

// Event工厂 删除方法
func (e *eventsCollector) Remove(conn *websocket.Conn) error {
	fd := websocketFd(conn)
	err := unix.EpollCtl(e.fd, syscall.EPOLL_CTL_DEL, fd, nil)
	if err != nil {
		return err
	}
	e.lock.Lock()
	defer e.lock.Unlock()
	delete(e.connections, fd)
	if len(e.connections)%100 == 0 {
		log.Printf("Total number of connections: %v", len(e.connections))
	}
	return nil
}

// Event工厂 等待锁
func (e *eventsCollector) Wait() ([]*websocket.Conn, error) {
	events := make([]unix.EpollEvent, 100)
	n, err := unix.EpollWait(e.fd, events, 100)
	if err != nil {
		return nil,err
	}
	e.lock.RLock()
	defer e.lock.RUnlock()
	var connections[]*websocket.Conn
	for i := 0; i < n; i++ {
		conn := e.connections[int(events[i].Fd)]
		connections = append(connections, conn)
	}
	return connections, nil
}

// 返回连接资源句柄的系统fd
func websocketFd(conn *websocket.Conn) int {
	connVal := reflect.Indirect(reflect.ValueOf(conn)).FieldByName("conn").Elem()
	tcpConn := reflect.Indirect(connVal).FieldByName("conn")
	fdVal   := tcpConn.FieldByName("fd")
	pfdVal  := reflect.Indirect(fdVal).FieldByName("pfd")
	return int(pfdVal.FieldByName("Sysfd").Int())
}
```


```go
package optimize_ws_goroutines

import (
	"github.com/gorilla/websocket"
	"log"
	"net/http"
	"syscall"
)

// 通过声明主结果体获取数据对象
var EventsCollector *eventsCollector

// websocket的handle的函数
func websocketHandle(w http.ResponseWriter, r *http.Request) {
	// websocket包中的结构体
	upgrade := websocket.Upgrader{}
	conn, e := upgrade.Upgrade(w, r, nil)
	if e != nil {
		return
	}
	// 如果有新的连接就加入结构体
	if err := EventsCollector.Add(conn); err != nil {
		log.Printf("Failed to add connection")
		conn.Close()
	}
}

// syscall 系统底层调用指针
func RunWebSocketServer() {
	var rLimit syscall.Rlimit
	if err := syscall.Getrlimit(syscall.RLIMIT_NOFILE, &rLimit); err != nil {
		panic(err)
	}
	rLimit.Cur = rLimit.Max
	if err := syscall.Setrlimit(syscall.RLIMIT_NOFILE, &rLimit); err != nil {
		panic(err)
	}

	// enable pprof hooks
	go func() {
		if err := http.ListenAndServe("127.0.0.1:6060", nil); err != nil {
			log.Fatalf("pprof failed : %v", err)
		}
	}()

	// Start event
	var err error
	EventsCollector, err = MkEventsCollector()
	if err != nil {
		panic(err)
	}

    // 开始读取数据打印
	go Start()
    
	// http Handle读取对象
	http.HandleFunc("/", websocketHandle)
	if err := http.ListenAndServe(":8000", nil); err != nil {
		log.Fatal(err)
	}
}

// 运行等待和输入日志
func Start() {
	for {
		connections,err := EventsCollector.Wait()
		if err != nil {
			log.Printf("Failed to epoll wait %v", err)
			continue
		}

		for _, conn := range connections {
			if conn == nil {
				break
			}
			_, msg, err := conn.ReadMessage()
			if err != nil {
				if err := EventsCollector.Remove(conn); err != nil {
					log.Printf("Failed to remove %v", err)
				}
				conn.Close()
			} else {
				log.Printf("msg: %s", string(msg))
			}
		}
	}
}
```

#### gobwas/ws库

gobwas/ws库对 gorilla/websocket进行了显示改进   
这可以实现更高的性能和更低的内存占用，这主要是由于gobwas/ws库中的高性能设计允许在连接之间重用分配的缓冲区

功效一样 但是ws库不一致

```go
package optimize_ws_gobwas

import (
	"golang.org/x/sys/unix"
	"log"
	"net"
	"reflect"
	"sync"
	"syscall"
)

type eventCollector struct {
	fd 			int
	connections map[int]net.Conn
	lock 		*sync.RWMutex
}


func MkEventsCollector() (*eventCollector, error) {
	fd, err := unix.EpollCreate1(0)
	if err != nil {
		return nil, err
	}
	return &eventCollector{
		fd: 		 fd,
		lock: 		 &sync.RWMutex{},
		connections: make(map[int]net.Conn),
	}, nil
}

func (e *eventCollector) Add(conn net.Conn) error {
	// 提取连接文件描述符
	fd := websocketFD(conn)
	err := unix.EpollCtl(e.fd, syscall.EPOLL_CTL_ADD, fd, &unix.EpollEvent{Events:unix.POLLIN | unix.POLLHUP, Fd: int32(fd)})
	if err != nil {
		return err
	}

	e.lock.Lock()
	defer e.lock.Unlock()
	e.connections[fd] = conn
	if len(e.connections)%100 == 0 {
		log.Printf("Total number of connections : %v", len(e.connections))
	}
	return nil
}

func (e *eventCollector) Remove(conn net.Conn) error {
	fd := websocketFD(conn)
	err := unix.EpollCtl(e.fd, syscall.EPOLL_CTL_DEL, fd, nil)
	if err != nil {
		return err
	}

	e.lock.Lock()
	defer e.lock.Unlock()
	delete(e.connections, fd)
	if len(e.connections)%100 == 0 {
		log.Printf("Total number of connections : %v", len(e.connections))
	}
	return nil
}

func (e *eventCollector) Wait() ([]net.Conn, error) {
	events := make([]unix.EpollEvent, 100)
	n, err := unix.EpollWait(e.fd, events, 100)
	if err != nil {
		return nil,err
	}
	e.lock.RLock()
	defer e.lock.RUnlock()
	var connections []net.Conn
	for i := 0; i < n; i++ {
		conn := e.connections[int(events[i].Fd)]
		connections = append(connections, conn)
	}
	return connections, nil
}

func websocketFD(conn net.Conn) int {
	tcpConn := reflect.Indirect(reflect.ValueOf(conn)).FieldByName("conn")
	fdVal := tcpConn.FieldByName("fd")
	pfdVal := reflect.Indirect(fdVal).FieldByName("pfd")
	return int(pfdVal.FieldByName("Sysfd").Int())
}
```

使用gobwas的服务端

```go
package optimize_ws_gobwas

import (
	"github.com/gobwas/ws"
	"github.com/gobwas/ws/wsutil"
	"log"
	"net/http"
	"syscall"
)

var EventsCollector *eventCollector

func websocketHandler(w http.ResponseWriter, r *http.Request) {
	// Upgrade 连接
	conn, _, _, err := ws.UpgradeHTTP(r, w)
	if err != nil {
		return
	}
	if err := EventsCollector.Add(conn); err != nil {
		log.Printf("Failed to add connection %v", err)
		conn.Close()
	}
}

func RunWebSocketGobWas() {
	// 增加资源限制
	var rLimit syscall.Rlimit
	if err := syscall.Getrlimit(syscall.RLIMIT_NOFILE, &rLimit); err != nil {
		panic(err)
	}
	rLimit.Cur = rLimit.Max
	if err := syscall.Setrlimit(syscall.RLIMIT_NOFILE, &rLimit); err != nil {
		panic(err)
	}
	// enable pprof hook
	go func() {
		if err := http.ListenAndServe("127.0.0.1:6060", nil); err != nil {
			log.Fatalf("pprof failed : %v", err)
		}
	}()
	// Start event
	var err error
	EventsCollector, err = MkEventsCollector()
	if err != nil {
		panic(err)
	}
	go Start()
	http.HandleFunc("/", websocketHandler)
	if err := http.ListenAndServe("0.0.0.0:8000", nil);err != nil {
		log.Fatal(err)
	}
}

func Start() {
	for {
		connections, err := EventsCollector.Wait()
		if err != nil {
			log.Printf("Failed to epoll wait %v", err)
			continue
		}
		for _, conn := range connections {
			if conn == nil {
				break
			}
			if msg, _, err := wsutil.ReadClientData(conn); err != nil {
				if err := EventsCollector.Remove(conn); err != nil {
					log.Printf("Failed to remove %v", err)
				}
				conn.Close()
			} else {
				// 如果连接速率 > 1M 那么stdout会不停发送消息(???)
				log.Printf("msg:%s", string(msg))
			}
		}
	}
}
```



