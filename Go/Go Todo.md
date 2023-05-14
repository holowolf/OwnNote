Golang 相对于传统的服务端编程语言已经具备非常大的性能优势，写网络服务的api，标准库其实都已经封装好了，主要是注意下面这些使用场景就能达到高性能：

GC 不要忽略gc的影响，应尽量避免频繁创建小块内存碎片
锁 能用channel就用channel，能用读写锁就用读写锁
对象复用 大块内存buffer可以考虑下对象池 内存复用
cgo cgo是go提供的在go代码里面执行c/c++的功能，cgo只能在线程上执行，会有频繁的线程切换，能不用就不用
反射 反射性能较低，能不用就不用，比如官方的json解析库性能很差 可以替换成jsoniter
监控 监控一定要做好 保持对自己服务的敏感 出现问题立马分析 如profile性能分析cpu mem gc， 耗时分析

最简单的 Cgo 运行

package main

/*
#include <stdio.h>
#include <stdlib.h>

void myprint(char *s) {
printf("%s\n", s);
}
*/
import "C"

import "unsafe"



func main() {
cs := C.CString("hello world\n")
C.myprint(cs)
C.free(unsafe.Pointer(cs))
}
Notice : 注意import "C" 必须和 C代码注释连接在一起 不允许空格


# 启用 Go Modules 功能
go env -w GO111MODULE=on

# 配置 GOPROXY 环境变量，以下三选一

# 1. 七牛 CDN
go env -w  GOPROXY=https://goproxy.cn,direct

# 2. 阿里云
go env -w GOPROXY=https://mirrors.aliyun.com/goproxy/,direct

# 3. 官方
go env -w  GOPROXY=https://goproxy.io,direct
go mod download 下载成功

go mod download