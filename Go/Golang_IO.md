## Golang Basic

### IO Write

#### write方法写入缓冲

```go
var write bytes.Buffer
write.Write([]byte(p))
```

通过Write方法写入到write缓冲区

#### write写入channel

```go
type NewChanWriteStruct struct {
	// ch 就是目标资源
	ch chan byte
}

func NewChanWrite() *NewChanWriteStruct {
	return &NewChanWriteStruct{make(chan byte, 1024)}
}

func (w *NewChanWriteStruct)Chan() <-chan byte {
	return w.ch
}

func (w *NewChanWriteStruct) Write(p []byte) (int, error) {
	n := 0
	// 遍历输入数据, 按字节写入资源
	for _,b := range p {
		w.ch <- b
		n++
	}
	return n, nil
}

func (w *NewChanWriteStruct) Close() error {
	close(w.ch)
	return nil
}

func ChanIoRun() {
	write := NewChanWrite()
	go func() {
		defer write.Close()
		_, _ = write.Write([]byte("Stream "))
		_, _ = write.Write([]byte("me!"))
	}()
	for c := range write.Chan() {
		fmt.Printf("%c", c)
	}
	fmt.Println()
}
```

重写结构对象的结构对象的 Write方法 和 Close方法


#### Write写入File

类型os.File表示本地系统上的文件.它实现了io.Reader和io.Writer,因此可以在任何io上下文中使用.

```go
func WriteIOFile() {
	proverbs := []string {
		"Channels orchestrate mutexes serialize\n",
		"Cgo is not Go\n",
		"Errors are values\n",
		"Don't panic\n",
	}

	file, e := os.Create("./proverbs")
	if e != nil {
		fmt.Println(e)
		os.Exit(1)
	}
	defer file.Close()

	for _,b := range proverbs {
		n, e := file.Write([]byte(b))
		if e != nil {
			fmt.Println(e)
			os.Exit(1)
		}
		if n != len(b) {
			fmt.Println("write error")
			os.Exit(1)
		}
	}

	fmt.Println("write end!")
}
```

os.File将byte写入到文件中



### IO Read

#### read 写入缓冲区
```go
func BufferIORead() {
	reader := strings.NewReader("Clear is better than clever")
	p := make([]byte, 4)
	for {
		n, err := reader.Read(p)
		if err != nil {
			if err == io.EOF {
				fmt.Println("EOF:",n)
				break
			}
			fmt.Println(err)
			os.Exit(1)
		}
		fmt.Println(n, string(p[:n]))
	}
}
```

使用读取器实现的Reader
讲读取器内容读入缓冲区中, 指定缓冲区大小, 读入.
最后在err = io.EOF时进行判断并跳出


#### 实现自定义Reader类

```go
type alphaReader struct {
	// 资源
	src string
	// 当前读到的位置
	cur int
}

// 创建一个实例
func newAlphaReader(src string) *alphaReader {
	return &alphaReader{src : src}
}

// 过滤函数
func alpha(r byte) byte {
	if (r >= 'A' && r <= 'Z') || (r >= 'a' && r <= 'z') {
		return r
	}
	return 0
}

// Read 方法
func (a *alphaReader) Read(p []byte) (int, error) {
	// 当前位置 >= 字符串长度 说明已经读取到结尾 返回EOF
	if a.cur >= len(a.src) {
		return 0, io.EOF
	}

	// x 式剩余魏都区长度
	x := len(a.src) - a.cur
	n, bound := 0,0
	if x >= len(p) {
		// 剩余长度超过缓冲区大小, 本次可完全填满缓冲区
		bound = len(p)
	} else if x < len(p) {
		// 剩余长度小于缓冲区大小, 使用剩余长度输出, 缓冲区不补满
		bound = x
	}

	buf := make([]byte, bound)
	for n < bound {
		// 每次读取一个字节, 执行过滤函数
		if char := alpha(a.src[a.cur]); char != 0 {
			buf[n] = char
		}
		n++
		a.cur++
	}
	// 将处理后得到的buf 内容复制到p中
	copy(p, buf)
	return n, nil
}

func RunOwnReader() {
	reader := newAlphaReader("Hello! It's 9am, where is the sun?")
	p := make([]byte, 4)
	for {
		_, e := reader.Read(p)
		if e == io.EOF {
			break
		}
		fmt.Print(string(p))
	}
	fmt.Println()
}
```

#### 组合多个Reader

```go
type multiAlphaReader struct {
	reader io.Reader
}

func multiNewAlphaReader(reader io.Reader) *multiAlphaReader {
	return &multiAlphaReader{reader : reader}
}

func multiAlpha(r byte) byte {
	if (r >= 'A' && r <= 'Z') || (r >= 'a' && r <= 'z') {
		return r
	}
	return 0
}

func (a *multiAlphaReader) Read(p []byte) (int, error) {
	// 这行代码调用的就是 io.Reader
	n,err := a.reader.Read(p)
	if err != nil {
		return n, err
	}
	buf := make([]byte, n)
	for i := 0; i < n; i++ {
		if char := multiAlpha(p[i]); char != 0 {
			buf[i] = char
		}
	}
	copy(p,buf)
	return n, nil
}

func RunMultiRead() {
	reader := multiNewAlphaReader(strings.NewReader("Hello! It's 9am, where is the sun?"))
	p := make([]byte,4)
	for {
		i, e := reader.Read(p)
		if e == io.EOF {
			break
		}
		fmt.Print(string(p[:i]))
	}
}
```

实现自定义Reader方法 所有可以使用Read的方法 都可以用

#### File Read

```go
func FileRun() {
	file, e := os.Open("./proverbs")
	if e != nil {
		fmt.Println(e)
		os.Exit(1)
	}
	defer file.Close()

	// 任何实现了 io.Reader的类型都可以传入multiNewAlphaReader
	// 至于具体如何读取文件, 那是标准已经实现了的,我们不用再做一遍,达到了重用的目的

	reader := multiNewAlphaReader(file)
	p := make([]byte, 4)
	for {
		n,err := reader.Read(p)
		if err == io.EOF {
			break
		}
		fmt.Print(string(p[:n]))
	}
}
```

文件读取字节内容

### 标准输入、输出和错误

os包有三个变量 os.Stdout, os.Stdin, os.Stderr, 它们的类型为 *os.File, 分别代表
系统标准输入, 系统标准输出, 系统标准错误 的文件句柄

#### 标准输出写入


```go
func RunOsStdout() {
	proverbs := []string{
		"Channels orchestrate mutexes serialize\n",
		"Cgo is not Go\n",
		"Errors are values\n",
		"Don't panic\n",
	}

	for _,p := range proverbs {
		time.Sleep(1000*time.Millisecond)
		// 因为os.Stdout 也实现了 io.Write
		n, err := os.Stdout.Write([]byte(p))
		if err != nil {
			fmt.Println(err)
			os.Exit(1)
		}
		if n != len(p) {
			fmt.Println("failed to write data")
			os.Exit(1)
		}
	}
}
```

#### io.Copy()

io.Copy()可以轻松的讲数据从一个reader拷贝到另一个Writer
它抽象出for循环模式 并且正确处理io.EOF和字节计数.

```go
func RunIOCopy() {
	proverbs := new(bytes.Buffer)
	proverbs.WriteString("Channels orchestrate mutexes serialize\n")
	proverbs.WriteString("Cgo is not Go\n")
	proverbs.WriteString("Errors are values\n")
	proverbs.WriteString("Don't panic\n")

	file, err := os.Create("./proverbs")
	if err != nil {
		fmt.Println(err)
		os.Exit(1)
	}

	defer file.Close()

	if _,err := io.Copy(file, proverbs); err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
	fmt.Println("file created")
}
```

```go
func RunIoCopyStdout() {
	file, e := os.Open("./proverbs")
	if e != nil {
		fmt.Println(e)
		os.Exit(1)
	}
	defer file.Close()

	if _, e := io.Copy(os.Stdout, file); e != nil {
		fmt.Println(e)
		os.Exit(1)
	}
}
```

#### io.WriteString()

```go
func WriteString() {
	file,err := os.Create("./Magic_message")
	if err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
	defer file.Close()

	if _, err := io.WriteString(file, "Go is fun!"); err != nil {
		fmt.Println(err)
		os.Exit(1)
	}
}
```

将字符串类型写入一个Write 任何一个write, file stdin 等等

#### 使用管道的Writer 和 Reader
类型 io.PipeWriter 和 io.PipeReader 在内存管道中模拟io操作
数据被写入管道的一端, 并使用单独的goroutine在管道的另一端读取.
下面使用io.Pipe()创建管道reader和write,然后将数据从proverbs缓冲区复制到io.Stdout

```go
func RunIoPipe() {
	proverbs := new(bytes.Buffer)
	proverbs.WriteString("Channels orchestrate mutexes serialize\n")
	proverbs.WriteString("Cgo is not Go\n")
	proverbs.WriteString("Errors are values\n")
	proverbs.WriteString("Don't panic\n")

	pipeRead, pipeWrite:= io.Pipe()
	// 将 proverbs 写入pipew 这一端

	go func() {
		defer pipeWrite.Close()
		_, _ = io.Copy(pipeWrite, proverbs)
	}()

	// 从另一端piper中读取数据并拷贝到标准输出
	_, _ = io.Copy(os.Stdout, pipeRead)
	_ = pipeRead.Close()
}
```

读取双端队列中的数据


#### 缓冲区 io

标准库中 bufio 包支持 缓冲区操作, 可以轻松处理文本内容
程序逐行读取内容, 并以'\n‘分割

```go
func RunBufferIo() {
	file, e := os.Open("proverbs")
	if e != nil {
		fmt.Println(e)
		os.Exit(1)
	}

	defer file.Close()
	reader := bufio.NewReader(file)

	for {
		line,err := reader.ReadString('\n')
		if err != nil {
			if err == io.EOF {
				break
			} else {
				fmt.Println(err)
				os.Exit(1)
			}
		}
		fmt.Print(line)
	}
}
```

分割读取到io缓冲区


#### ioutil 包

io 包下面的一个子包 utilio 封装了一些非常方便的功能
例如,下面使用函数 ReadFile 将文件内容加载到 []byte 中

```go
func RunUtil() {
	bytes, e := ioutil.ReadFile("./proverbs")
	if e != nil {
		fmt.Println(e)
		os.Exit(1)
	}
	fmt.Print(string(bytes))
}
```

通过ioutil来读取文件 以及工具类的一批方法