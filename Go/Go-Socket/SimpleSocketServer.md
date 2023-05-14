## 简单socket 服务端

```go

import (
	"bufio"
	"fmt"
	"net"
	"strings"
)

func Server() {
	fmt.Println("Launching server...")

	// listen on all interfaces
	listener, _ := net.Listen("tcp", ":39999")

	// accept connection on port
	conn, _ := listener.Accept()

	// run loop forever(or until ctrl-c)
	for {
		// will listen for message to process ending in new line(\n)
		message, _ := bufio.NewReader(conn).ReadString('\n')
		// output message received
		fmt.Print("Message Received:", string(message))
		//simple process for string received
		newMessage := strings.ToUpper(message)
		// send new string back to client
		_, _ = conn.Write([] byte(newMessage + "\n"))
	}
}
```