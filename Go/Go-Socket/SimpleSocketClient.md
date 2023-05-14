## 简单socket 客户端

```go
package simple_socket

import (
	"bufio"
	"fmt"
	"net"
	"os"
)

func Client() {
	// connect to this socket
	conn, _ := net.Dial("tcp", "127.0.0.1:39999")

	for {
		// read input form stdin
		reader := bufio.NewReader(os.Stdin)
		fmt.Print("Text to send: ")
		s, _ := reader.ReadString('\n')
		// send to socket
		_, _ = fmt.Fprintf(conn, s+"\n")
		// listen for reply
		readString, _ := bufio.NewReader(conn).ReadString('\n')
		fmt.Print("Message from server : " + readString)
	}
}
```