## Golang 操作文件或目录

#### 获取GOPATH

用法:得到`GOPATH`环境变量,就可以使用绝对路径访问一些目录和文件了
```
package os

import (
	"fmt"
	"go/build"
	"os"
)

func GetGoPath()  {
	goPath := os.Getenv("GOPATH")
	if goPath == "" {
		goPath = build.Default.GOPATH
	}
	fmt.Println(goPath)
}
```


### 文件创建/写/读

文件创建读与写
```
package os

import (
	"bytes"
	"fmt"
	"github.com/sirupsen/logrus"
	"io"
	systemOs "os"
	"os/exec"
	"syscall"
)

func SystemFunc()  {
	pwd, err := systemOs.Getwd()
	if err != nil {
		panic("system getPwd error")
	}
	path := pwd + "/test.py"
	createFile(path)
	writeFile(path)
	readFile(path)
	execFile(path)
	deleteFile(path)
}

// 创建文件
func createFile(path string)  {
	_, err := systemOs.Stat(path)
	if systemOs.IsNotExist(err) {
		var file, err = systemOs.Create(path)
		if err != nil {
			return
		}
		defer file.Close()
	}
	logrus.Info("file create success!")
}

// 写入文件
func writeFile(path string) {
	file, err := systemOs.OpenFile(path, systemOs.O_RDWR, 0755)
	if err != nil {
		return
	}
	defer file.Close()
	_, err = file.WriteString("#!/usr/bin/env python3\r\n")
	if err != nil {
		return
	}
	_, err = file.WriteString("import time\r\n")
	if err != nil {
		return
	}
	_, err = file.WriteString("print(time.time())\r\n")

	// 内存拷贝到磁盘?
	err = file.Sync()
	if err != nil {
		return
	}
	logrus.Info("write file success!")
}

// 读取文件
func readFile(path string) {
	file, err := systemOs.OpenFile(path, systemOs.O_RDWR, 0644)
	if err != nil {
		logrus.Info(err)
		return
	}
	defer file.Close()
	text := make([]byte, 1024)
	for {
		_, err := file.Read(text)

		// 如果到了文件结尾弹出
		if err == io.EOF {
			break
		}

		if err != nil && err != io.EOF{
			logrus.Info(err)
			return
		}
	}
	logrus.Info(string(text))
}

// 删除文件
func deleteFile(path string) {
	err := systemOs.Remove(path)
	if err != nil {
		logrus.Info(err)
		return
	}
	logrus.Info(fmt.Sprintf("%s remove success!", path))
}

// 执行文件
func execFile(path string)  {
	command := exec.Command("python3", path)
	outInfo := bytes.Buffer{}
	command.Stdout = &outInfo
	err := command.Start()
	if err != nil {
		logrus.Info(err.Error())
	}
	if err = command.Wait(); err != nil {
		logrus.Info(err)
	} else {
		logrus.Info(command.ProcessState.Pid())
		logrus.Info(command.ProcessState.Sys().(syscall.WaitStatus).ExitStatus())
		logrus.Info(outInfo.String())
	}
}
```