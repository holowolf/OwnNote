RPC基础 异步GOB连接

```go
package socket

import (
	"bufio"
	"encoding/gob"
	"flag"
	"github.com/pkg/errors"
	"io"
	"log"
	"net"
	"strconv"
	"strings"
	"sync"
)

type complexData struct {
	N int
	S string
	M map[string] int
	P [] byte
	C *complexData
}

const (
	Port = ":63999"
)

// Outcoing connections  (发射连接)
func Open(addr string) (*bufio.ReadWriter, error) {
	log.Println("Dial" + addr)
	conn, e := net.Dial("tcp", addr)
	if e != nil {
		return nil, errors.Wrap(e, "Dialing " + addr + "failed")
	}
	return bufio.NewReadWriter(bufio.NewReader(conn), bufio.NewWriter(conn)), nil
}

// 声明一个HandleFunc 该类型接收一个bufio.ReadWrite指针值的函数类型
type HandleFunc func(*bufio.ReadWriter)


// Endpoint 结构体 三个属性
// listener:net.Listen() 返回的Listener对象
// handler:用于保存已注册的处理器函数的映射
// m 互斥锁 解决map多个goroutine不安全的问题
type Endpoint struct {
	listener net.Listener
	handler map[string]HandleFunc
	m sync.RWMutex
}


func NewEndpoint() *Endpoint {
	// 创建一个新的Endpoint 和 一个空的 HandleFunc list集合
	return &Endpoint{
		handler : map[string] HandleFunc{},
	}
}

// AddHandleFunc adds a new function for handling incoming data
func (e *Endpoint) AddHandleFunc(name string, f HandleFunc) {
	e.m.Lock()
	e.handler[name] = f
	e.m.Unlock()
}

// 监听端口添加处理函数
func (e *Endpoint) listen() error {
	var err error
	e.listener, err = net.Listen("tcp", Port)
	if err != nil {
		return errors.Wrapf(err, "Unable to listen on port %s\n", Port)
	}
	log.Println("Listen on", e.listener.Addr().String())

	for {
		log.Println("Accept a connection request.")
		conn, err := e.listener.Accept()
		if err != nil {
			log.Println("Failed accepting a connection request : ", err)
			continue
		}
		log.Println("Handle incoming message.")
		go e.handleMessage(conn)
	}
}

// handleMessage 读取换行符前第一个字符串的连接方式
// 根据这个字符串使用适当的连接
func (e *Endpoint) handleMessage(conn net.Conn) {
	// 包装缓冲区 以便读写
	rw:= bufio.NewReadWriter(bufio.NewReader(conn), bufio.NewWriter(conn))
	defer conn.Close()

	// 从连接读取到EOF 期望命令称呼
	// 下一个输入调用此命令注册的处理程序
	for {
		log.Println("Receive command '")
		s, i := rw.ReadString('\n')
		switch {
		case i == io.EOF:
			log.Println("Reached EOF - close this connection.\n ---")
			return
		case i != i:
			log.Println("\nError reading command. Got: '"+s+"'\n", i)
			return
		}
		s = strings.Trim(s, "\n ")
		log.Println(s + "'")

		// 从handle 映射中获取对应的函数处理器并调用
		e.m.RLock()
		handleFunc, ok := e.handler[s]
		e.m.RUnlock()
		if !ok {
			log.Println("Command '" + s + "' is not registered.")
			return
		}
		handleFunc(rw)
	}
}


// 处理 STRING 的前置 请求
func handleStrings(rw *bufio.ReadWriter) {
	//Receive a string
	log.Print("Receive STRING message:")
	s, e := rw.ReadString('\n')
	if e != nil {
		log.Println("Cannot read from connection.\n", e)
	}
	s = strings.Trim(s, "\n ")
	log.Println(s)
	_, e = rw.WriteString("Thank you.\n")
	if e != nil {
		log.Println("Cannot write to connection.\n", e)
	}
	e = rw.Flush()
	if e != nil {
		log.Println("Flush failed.", e)
	}
}

func handleGob(rw *bufio.ReadWriter) {
	log.Println("Receive GOB data:")
	var data complexData
	// Create a decoder that decodes directly into a struct variable.
	dec := gob.NewDecoder(rw)
	err := dec.Decode(&data)
	if err != nil {
		log.Println("Error decoding GOB data:", err)
		return
	}
	// 打印结构体日志
	log.Printf("Outer complexData struct: \n%#v\n", data)
	log.Printf("Inner complexData struct: \n%#v\n", data.C)
}

// 使用client 连接
func client(ip string) error {
	// 一些测试数据 注意GOB 处理map和切片 没有问题的地柜数据结构
	testStruct := complexData{
		N: 23,
		S: "string data",
		M: map[string]int{"one": 1, "two": 2, "three": 3},
		P: []byte("abc"),
		C: &complexData{
			N: 256,
			S: "Recursive structs? Piece of cake!",
			M: map[string]int{"01": 1, "10": 2, "11": 3},
		},
	}

	rw, e := Open(ip + Port)
	if e != nil {
		return errors.Wrap(e, "Client: Failed to open connection to " + ip + Port)
	}

	// Send a STRING request.
	// Send the request name.
	// Send the data.
	log.Println("Send the string request.")
	n, err := rw.WriteString("STRING\n")
	if err != nil {
		return errors.Wrap(err, "Could not send the STRING request (" + strconv.Itoa(n) + " bytes written)")
	}
	n, err = rw.WriteString("Additional data.\n")
	if err != nil {
		return errors.Wrap(err, "Could not send additional STRING data(" + strconv.Itoa(n) + " bytes written")
	}
	log.Println("Flush the buffer.")
	e = rw.Flush()
	if e != nil {
		return errors.Wrap(err, "Flush failed.")
	}

	// Read the reply.
	log.Println("Read the reply.")
	response, err := rw.ReadString('\n')
	if err != nil {
		return errors.Wrap(err, "Client : Failed to read the reply: '"+response+"'")
	}
	log.Println("STRING request: got a response:", response)

	// Send a GOB request.
	// Create an encoder that directly transmits to 'rw'.
	// Send the request name
	// Send the GOB

	log.Println("Send a struct as GOB:")
	log.Printf("Outter complexData struct: \n%#v\n", testStruct)
	log.Printf("Inner complexData struct: \n%#v\n", testStruct.C)

	enc := gob.NewEncoder(rw)
	n, err = rw.WriteString("GOB\n")
	if err != nil {
		return errors.Wrap(err, "Could not write GOB data (" + strconv.Itoa(n)+ " bytes written)")
	}
	err = enc.Encode(testStruct)
	if err != nil {
		return errors.Wrapf(err, "Encode failed for struct: %#v", testStruct)
	}
	err = rw.Flush()
	if err != nil {
		return errors.Wrap(err, "Flush failed.")
	}
	return nil
}

// 监听端口请求
// 注册处理映射函数
func server() error {
	endpoint := NewEndpoint()
	// Add the handle funcs
	endpoint.AddHandleFunc("STRING", handleStrings)
	endpoint.AddHandleFunc("GOB", handleGob)
	return endpoint.listen()
}

// main 方法
func Main() {
	connect := flag.String("connect", "", "IP address of process to join. If empty, go into listen mode.")
	flag.Parse()

	if *connect != "" {
		err := client(*connect)
		if err != nil {
			log.Println("Error:", errors.WithStack(err))
		}
		log.Println("Client done.")
		return
	}

	// Else go into server mode.
	err := server()
	if err != nil {
		log.Println("Error:", errors.WithStack(err))
	}
	log.Println("Server done.")
}

func init() {
	log.SetFlags(log.Lshortfile)
}
```