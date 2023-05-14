## Golang常用语法

```go
package basic

import (
    "bufio"
    "fmt"
    "io/ioutil"
    "math"
    "os"
    "reflect"
    "runtime"
    "strconv"
)

func consts() {
    // 如果对常量进行规定 那么这些常量有类型
    const fileName string = "../../main.go"
    // 也可以对常量不确定类型
    const a, b = 5, 9
    // 如果不确定类型那么可以随时是任何类型 这里就不需要转float类型
    math.Sqrt(a*a + b*b)
    // 可以对常量进行定义大量定义
    // const 定义的数值可以对各种类型使用
    const (
        fileNameString string = "java"
        aa, bb = 123, 444
    )
}


func Enums() {
    // golang的枚举类型 使用const进行定义
    const(
        cpp = 0
        java = 1
        python = 2
        golang = 3
    )

    // 简化const枚举
    // iota 为自增表达式
    const(
        cpp2 = iota
        _
        java2
        python2
        golang2
        javascript2
    )

    // b, kb, mb, gb , tb, pb
    // 只要能写出表达式就能自增
    const(
        b = 1 << (10 * iota)
        kb
        mb
        gb
        tb
        pb
    )

    fmt.Println(cpp, java, python, golang)
    fmt.Println(cpp2, java2, python2, golang2, javascript2)
    fmt.Println(b, kb, mb, gb, tb, pb)
}

func Branch() {
    const fileName2 string = "/home/yeenughu/Code/Go/GoogleTeacher/main.go"

    // 第一中写法
    contents, e := ioutil.ReadFile(fileName2)
    if e != nil {
        fmt.Println(e)
    } else {
        fmt.Printf("%s\n", contents)
    }

    // 第二种写法 简化写法
    if contentsTwo, i := ioutil.ReadFile(fileName2); i != nil {
        fmt.Println(i)
    } else {
        fmt.Printf("%s\n", contentsTwo)
    }
}

func Grade(score int) string {
    grades := ""
    switch {
    case score < 0 || score > 100 :
        panic(fmt.Sprintf("wrong score : %d", score))
    case score < 60 :
        grades = "F"
    case score < 80 :
        grades = "C"
    case score < 90:
        grades = "B"
    case score <= 100:
        grades = "A"
    default:
        panic(fmt.Sprintf("wrong score : %d", score))
    }
    return grades
}

func ConvertToBin(n int) string {
    result := ""
    for ; n > 0 ; n /= 2 {
        lsb := n % 2
        result = strconv.Itoa(lsb) + result
    }
    return result;
}

func PrintFile(filename string) {
    if file, e := os.Open(filename); e != nil {
        panic(e)
    } else {
        scanner := bufio.NewScanner(file)
        for scanner.Scan() {
            fmt.Println(scanner.Text())
        }
    }
}

func ForeverPrint(filename string) {
    // golang 对 for 进行高效优化
    for {
        fmt.Println(filename)
    }
}


func Eval(a, b int, op string) (int, error) {
    switch op {
    case "+":
        return a + b, nil
    case "-":
        return a - b, nil
    case "*":
        return a * b, nil
    case "/":
        q, _ := Div(a, b)
        return q, nil
    default:
        return 0, fmt.Errorf("unsupported operation : " + op)
    }
}

func Div(a, b int) (q, r int) {
    return a / b, a % b
}

// 使用nil 将错误和正确的返回值区分开来
func RunEval() {
    if i, e := Eval(10, 2, "/"); e != nil {
        fmt.Println(e)
    } else {
        fmt.Println(i)
    }
}


func Pow(a, b int) int {
    return int(math.Pow(float64(a), float64(b)))
}

// 函数一等公民的体现 函数内部调用函数
func Apply(op func(int, int) int, a, b int) int {
    pointer := reflect.ValueOf(op).Pointer()
    opName := runtime.FuncForPC(pointer).Name()
    fmt.Printf("Calling function %s with args" + "(%d, %d)", opName, a, b)
    return op(a, b)
}

// 传递可变参数
func Sum(numbers ...int) int {
    s := 0
    for i := range numbers {
        s += numbers[i]
    }
    return s
}

// 交换两个值 使用指针传递的方式进行
func Swap(a, b *int) {
    *b, *a = *a, *b
}

func Arrays() {
    // 定义数组的方法
    var arrx [5] int
    // 定义数组的第二种方法
    arry := [3] int{1, 2, 3}
    // 第三种定义方案
    arrz := [...] int {3, 4, 5, 6, 7}
    // 数组是值的类型
    var grid [4][5] int

    // [10] int 和 [20] int 是不同类型
    // 调用func f(arr[10] int) 会拷贝数组
    // 在go语言中一般不直接使用数组 而是使用切片
    fmt.Println(arrx, arry, arrz)

    fmt.Println(grid)

    sum := 0
    for _, v := range arrz {
        sum += v
    }
    fmt.Println(sum)
}

func Slices() {
    arr := [...] int {0, 1, 2, 3, 4, 5, 6, 7}
    s := arr[2:6]
    fmt.Println(s)
}
```