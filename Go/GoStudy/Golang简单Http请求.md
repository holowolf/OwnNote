## Golang简单Http请求

Golang Http请求

```go
package httpUtil

import (
    "fmt"
    "net/http"
    "net/http/httputil"
)

func Client() {
    request, i := http.NewRequest(http.MethodGet, "http://www.imooc.com", nil)
    if i != nil {
        panic(i)
    }
    request.Header.Add("User-Agent", "Mozilla/5.0 (iPhone; CPU iPhone OS 11_0 like Mac OS X) AppleWebKit/604.1.38 (KHTML, like Gecko) Version/11.0 Mobile/15A372 Safari/604.1")
    client := http.Client {
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            fmt.Println("Redirect:", request)
            return nil
        },
    }
    resp, err := client.Do(request)
    if err != nil {
        panic(err)
    }
    defer resp.Body.Close()

    bytes, e := httputil.DumpResponse(resp, true)
    if e != nil {
        panic(e)
    }
    fmt.Printf("%s", bytes)
}
```