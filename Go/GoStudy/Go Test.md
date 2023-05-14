go test -run=TestHelloworld
go test -bench=BenchmarkHelloworld

go test -timeout 30s cmap -run ^(TestMap)$

test -benchmem -run=^$ cmap -bench ^(BenchmarkMapGet)$

如果合并大量重复的字符串请使用strings.Repeat, 如果要合并不同的字符串,且图方便建议使用string.Builder + Write bytes/string.

1. 使用strings.Repeat效率最高,从strings.Repeat源码它是提前分配内存,使用strings.Builder.所以他的效率更高 18379 ns/op,大约是其他的1/10.
2. 其次使用strings.Buffer 提前分配内存 120803 ns/op
3. 使用 + 连接字符串效率最低 87599885 ns/op
4. Buffer Write bytes 和 Buffer Write string 几乎没有差别,因为在Go语言中 string 就是 []byte