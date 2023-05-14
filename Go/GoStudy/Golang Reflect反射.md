## Golang Reflect反射

### Reflect 读取 tag

gorm模型中参数验证, beego动态路由, json解析等
```
package reflect

import (
	"github.com/sirupsen/logrus"
	"reflect"
)

type User struct {
	Name string `mcl:"name" default:"testName"`
	Age int64 `mcl:"age" default:"2222"`
	Email string `mcl:"email" default:"test@qq.com"`
	Github string `mcl:"github" default:"a8m"`
}

// 获取结构体属性值
func GetReflectTag() {
	var u interface{} = User{}

	//TypeOf 返回 reflect.Type 表示 u的动态Type
	t := reflect.TypeOf(u)

	// 如果反射类型不是struct
	if t.Kind() != reflect.Struct {
		// 直接返回
		return
	}
	// 获取字段下的各种属性
	for i := 0; i < t.NumField(); i++ {
		f := t.Field(i)
		logrus.Info(f.Tag.Get("mcl"), f.Tag.Get("default"), f.Name)
	}
}

```


### SetStruct 属性值

```
package reflect

import (
	"github.com/sirupsen/logrus"
	"reflect"
)

func SetStruct() {
	u := &User{Name: "NameUser"}

	elem := reflect.ValueOf(u).Elem()
	// 获取对应所有结构体下的所有元素
	logrus.Info(elem)

	name := elem.FieldByName("Github")

	// 判断file是有值的状态
	if !name.IsValid() || !name.CanSet() {
		return
	}

	// 如果对象反射的不是string 或者不等于空字符串
	if name.Kind() != reflect.String || name.String() != "" {
		logrus.Info(name.Kind(), reflect.String, name.String())
		return
	}
	// 设置结构体数据
	name.SetString("yeenughu@github.com")
	logrus.Info(u.Name, u)
}

```

### SetSlice 添加字符 decoder

```
package reflect

import (
	"errors"
	"github.com/sirupsen/logrus"
	"io"
	"reflect"
)

func AddSliceData()  {
	var (
		stringSlice []string
		interfaceSlice []interface{}
		writerSlice []io.Writer
	)

	logrus.Info(decide(&stringSlice), stringSlice) // 通过
	logrus.Info(decide(&interfaceSlice), interfaceSlice) // 通过
	logrus.Info(decide(&writerSlice), writerSlice) // 失败
}

// 向不知名的类型slice中添加字符 使用decode
func decide(i interface{}) error {
	v := reflect.ValueOf(i)

	// 如果Ptr不是指针类型
	if v.Kind() != reflect.Ptr {
		logrus.Error("non-pointer %v", v.Type())
		return errors.New("non-pointer" + v.Type().String())
	}

	v = v.Elem()
	// 如果结构元素不是slice类型
	if v.Kind() != reflect.Slice {
		logrus.Error("can't decide non-slice value")
		return errors.New("can't decide non-slice value")
	}
	v.Set(reflect.MakeSlice(v.Type(), 3, 3))

	if !canAssign(v.Index(0)) {
		return errors.New("can't assign string to slice elements")
	}

	for i, w := range[]string {"foo", "bar", "baz"} {
		v.Index(i).Set(reflect.ValueOf(w))
	}
	return nil
}

// 是否能分配参数 判断其反射类型为string 或 其类型为interface 并且 方法的个数为0
func canAssign(v reflect.Value) bool {
	return v.Kind() == reflect.String || (v.Kind() == reflect.Interface && v.NumMethod() == 0)
}
```

### SetNumberValue Decoder

```
package reflect

import (
	"errors"
	"fmt"
	"github.com/sirupsen/logrus"
	"reflect"
)

const number = 255

func IntAdd() {
	var (
		a int8
		b int16
		c uint
		d float32
		e string
	)

	logrus.Info(decode(&a), a) // 失败
	logrus.Info(decode(&b), b) // 通过
	logrus.Info(decode(&c), c) // 通过
	logrus.Info(decode(&d), d) // 通过
	logrus.Info(decode(&e), e) // 失败
}


func decode(i interface{}) error {
	// 获取接口反射值
	v := reflect.ValueOf(i)

	// 如果类型不是指针
	if v.Kind() != reflect.Ptr {
		return errors.New("non-pointer" + v.Type().String())
	}
	//获取value在指针上
	v = v.Elem()
	switch v.Kind() {
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		if v.OverflowInt(number) {
			return errors.New(fmt.Sprintf("can't assign value %s", v.Kind().String()))
		}
		v.SetInt(number)

	case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
		if v.OverflowUint(number) {
			return errors.New(fmt.Sprintf("can't assign value %s", v.Kind().String()))
		}
		v.SetUint(number)

	case reflect.Float32, reflect.Float64:
		if v.OverflowFloat(number) {
			return errors.New(fmt.Sprintf("can't assign value %s", v.Kind().String()))
		}
		v.SetFloat(number)

	default:
		return errors.New(fmt.Sprintf("can't assign value to a non-number type"))
	}
	return nil
}
```

### SetKeyValueToMap

设置keyValue转换至Map
```
package reflect

import (
	"errors"
	"fmt"
	"github.com/sirupsen/logrus"
	"reflect"
	"strconv"
	"strings"
)

func DecodeMap() {

	var (
		map0 = make(map[int]string)
		map1 = make(map[int64]string)
		map2 map[interface{}]string
		map3 map[interface{}]interface{}
		map4 map[bool]string
	)

	s := "1=apple;2=banana;3=pure"
	logrus.Info(decodeMap(s, &map0), map0) // 通过
	logrus.Info(decodeMap(s, &map1), map1) // 通过
	logrus.Info(decodeMap(s, &map2), map2) // 通过
	logrus.Info(decodeMap(s, &map3), map3) // 通过
	logrus.Info(decodeMap(s, &map4), map4) // 失败
}


func decodeMap(s string, i interface{}) error {
	v := reflect.ValueOf(i)
	// 如果interface 不是指针类型返回错误
	if v.Kind() != reflect.Ptr {
		return errors.New(fmt.Sprintf("non-pointer %s", v.Type()))
	}

	// 获取指针指向对象
	v = v.Elem()

	// 获取interface类型
	t := v.Type()

	// 开辟新的new map空间, 如果 v 是空指针, 看map2 map3 map4
	if v.IsNil() {
		// 设置map类型
		v.Set(reflect.MakeMap(t))
	}

	// split string
	for _,kv := range strings.Split(s, ";") {
		data := strings.Split(kv, "=")
		n, err := strconv.Atoi(data[0])
		if err != nil {
			return fmt.Errorf("failed to parse number: %v", err)
		}
		key, value := reflect.ValueOf(n), reflect.ValueOf(data[1])
		// 获取interface的类型
		keyType := t.Key()
		if !key.Type().ConvertibleTo(keyType) {
			return errors.New(fmt.Sprintf("can't conver key to Type %v", keyType.Kind()))
		}
		// 转换指定类型
		convert := key.Convert(keyType)
		elem := t.Elem()

		// 如果interface 类型不等于数值转换后的类型 或 值类型转换错误
		if elem.Kind() != v.Kind() && !value.Type().ConvertibleTo(elem) {
			return errors.New(fmt.Sprintf("can't assign value to type %v", keyType.Kind()))
		}
		v.SetMapIndex(convert, value.Convert(elem))
	}
	return nil
}
```

### DecdoeKeyValueToStruct

转换keyValue到struct
```
package reflect

import (
	"errors"
	"fmt"
	"github.com/sirupsen/logrus"
	"reflect"
	"strings"
)

type UserDecodeStruct struct {
	Name string
	Github string
	Email string
}

func DecodeStruct()  {
	var (
		user0 User
		user1 *User
		user2 = new(User)
		user3 struct{Name string}
		filedString = "Name=Yeenughu;Github=yeenughu@gmail.com"
	)

	logrus.Info(decodeStruct(filedString ,&user0), user0) // 通过
	logrus.Info(decodeStruct(filedString ,user1), user1) //
	logrus.Info(decodeStruct(filedString ,user2), user2)
	logrus.Info(decodeStruct(filedString ,user3), user3)
	logrus.Info(decodeStruct(filedString ,&user3), user3)
}

func decodeStruct(s string, i interface{}) error {
	v := reflect.ValueOf(i)
	if v.Kind() != reflect.Ptr || v.IsNil() {
		return errors.New(fmt.Sprintf("decode requires non-nil pointer"))
	}

	// get the value that the pointer v points to.
	v = v.Elem()
	// assume that the input valid
	for _, kv := range strings.Split(s, ";") {
		split := strings.Split(kv, "=")
		name := v.FieldByName(split[0])
		// 如果这个字段是有效的 并且是定义的值不可以被更改
		if !name.IsValid() || !name.CanSet() {
			continue
		}
		// assume all the filed are type string 假设所有字段都是string
		name.SetString(split[1])
	}
	return nil
}
```

### EncodeKeyValueToStruct

struct转换至keyValue
```
package reflect

import (
	"fmt"
	"github.com/sirupsen/logrus"
	"reflect"
	"strings"
)

type UserStructDecode struct {
	Email string `kv:"email, omitempty"`
	Name string `kv:"name, omitempty"`
	Github string `kv:"github, omitempty"`
	private string
}

// struct 转换 Map或者Slice
func StructEncodeMapOrSlice() {
	var (
		user = UserStructDecode{Name: "NewName", Github: "newGithub"}
		vxt = struct {
			A, B, C string
		}{
			"A-Name",
			"B-Name",
			"C-Name",
		}
		w = &User{}
	)

	logrus.Info(structEncodeMap(user))
	logrus.Info(structEncodeMap(vxt))
	logrus.Info(structEncodeMap(w))
}

// 结构体DecodeMap
func structEncodeMap(i interface{}) (string, error) {
	valueOf := reflect.ValueOf(i)
	valueType := valueOf.Type()

	if valueType.Kind() != reflect.Struct {
		return "", fmt.Errorf("type %s is not supported", valueType.Kind())
	}

	var s []string
	// 遍历结构体字段
	for i := 0; i < valueType.NumField(); i++ {
		valueField := valueType.Field(i)
		//logrus.Info(valueField)
		// 字段程序包的路径 ??? ...
		if valueField.PkgPath != "" {
			continue
		}
		fieldValue := valueOf.Field(i)
		tag, b := readStructTag(valueField)
		if b && fieldValue.String() == "" {
			continue
		}
		s = append(s, fmt.Sprintf("%s=%s", tag, fieldValue.String()))
	}
	return strings.Join(s, ";"), nil
}

// 查看结构体Tag
func readStructTag(f reflect.StructField) (string, bool) {
	lookup, ok := f.Tag.Lookup("kv")
	if !ok {
		return f.Name, false
	}
	opts := strings.Split(lookup, ",")
	omit := false

	if len(opts) == 1 {
		omit = opts[1] == "omitempty"
	}
	return opts[0], omit
}
```

### Check Implement Interface

检验是否实现Interface
```
package reflect

import (
	"fmt"
	"github.com/sirupsen/logrus"
	"reflect"
)

type Marshaler interface {
	MarshalerKV() (string, error)
	MarshalerHello() string
}

type UserTestMarshaler struct {
	Email   string `kv:"email, omitempty"`
	Name    string `kv:"name, omitempty"`
	Github  string `kv:"github, omitempty"`
	private string
}

func (u UserTestMarshaler) MarshalerHello() string {
	return fmt.Sprintf("Hello interface Marshaler!")
}

func (u UserTestMarshaler) MarshalerKV() (string, error) {
	return fmt.Sprintf("name=%s,email=%s,github=%s", u.Name, u.Email, u.Github), nil
}

var marshalerType = reflect.TypeOf(new(Marshaler)).Elem()

func InterfaceMarshaler() {
	// interface结构体数据转换
	logrus.Info(encodeInterface(UserTestMarshaler{"EncodeMarshaler", "Marshaler", "Marshaler", ""}))
	// 结构体指针进行进行对象方法调用
	logrus.Info(encodeInterface(&UserTestMarshaler{Email: "EmailPoint", Name: "NamePoint", Github: "GithubPoint"}))
	// 调用对象的方法
	logrus.Info(reflect.ValueOf(UserTestMarshaler{}).MethodByName("MarshalerHello").Call(nil))
}

func encodeInterface(i interface{}) (string, error) {
	interfaceType := reflect.TypeOf(i)
	if !interfaceType.Implements(marshalerType) {
		return "", fmt.Errorf("encode only supports structs than implement the Marshaler interface")
	}
	marshaler := reflect.ValueOf(i).Interface().(Marshaler)
	return marshaler.MarshalerKV()
}
```

### 调用函数

调用有参函数
```
package reflect

import (
	"github.com/sirupsen/logrus"
	"reflect"
)

func Add(a, b int) int {
	return a * b
}

// 方法外部调用
func MethodCall(test [2]int)  {
	method := reflect.ValueOf(Add)
	if method.Kind() != reflect.Func {
		return
	}
	methodType := method.Type()
	argv := make([]reflect.Value, methodType.NumIn())
	// 获取对应参数
	for i := range argv {
		// validate the type of parameter "i" 判断参数类型
		if methodType.In(i).Kind() != reflect.Int {
			return
		}
		argv[i] = reflect.ValueOf(test[i])
	}
	// note than, len(result) == t.NumberOut()
	result := method.Call(argv)
	if len(result) != 1 || result[0].Kind() != reflect.Int {
		return
	}
	logrus.Info(result[0].Int())
}
```

### 动态调用方法

动态调用参数
```
package reflect

import (
	"fmt"
	"github.com/sirupsen/logrus"
	"html/template"
	"reflect"
	"strconv"
	"strings"
)

func EasyRoute()  {
	funcs := template.FuncMap{
		"trim": strings.Trim,
		"lower": strings.ToLower,
		"repeat": strings.Repeat,
		"replace": strings.Replace,
	}

	// 使用字符串执行对应函数
	logrus.Info(eval(`{{ "hello" 4 | repeat }}`, funcs))
	logrus.Info(eval(`{{ "Hello-World" | lower }}`, funcs))
	logrus.Info(eval(`{{ "foobarfoo" "foo" "bar" -1 | replace }}`, funcs))
	logrus.Info(eval(`{{ "-^-Hello-^-" "-^" | trim }}`, funcs))
}


// 执行
func eval(s string, funcs template.FuncMap) (string, error) {
	args, name := parseArgs(s)
	fn, ok := funcs[name]
	if !ok {
		return "", fmt.Errorf("function %s is not defined", name)
	}
	valueOf := reflect.ValueOf(fn)
	valueType := valueOf.Type()
	if len(args) != valueType.NumIn() {
		return "", fmt.Errorf("invalid number of arguments got: %v, want: %v", len(args), valueType.NumIn())
	}

	argv := make([]reflect.Value, len(args))
	for i := range argv {
		var argType reflect.Kind
		// 如果是字符串
		if strings.HasPrefix(args[i], "\"") {
			argType = reflect.String
			argv[i] = reflect.ValueOf(strings.Trim(args[i], "\""))
		} else {
			argType = reflect.Int
			num, _ := strconv.Atoi(args[i])
			argv[i] = reflect.ValueOf(num)
		}
		if valueType.In(i).Kind() != argType {
			return "", fmt.Errorf("Invalid argument. got: %v, want: %v ", argType, valueType.In(i).Kind())
		}
	}
	result := valueOf.Call(argv)
	if len(result) != 1 || result[0].Kind() != reflect.String {
		return "", fmt.Errorf("function %s must return a one string value", name)
	}
	return result[0].String(), nil
}

// 解析参数
func parseArgs(s string) ([]string, string) {
	args := strings.Split(strings.Trim(s, "{ }"), "|")
	return strings.Split(strings.Trim(args[0], " "), " "), strings.Trim(args[len(args) - 1], " ")
}
```

### 调用可变参数

通过argv调用多个参进行参与
```
package reflect

import (
	"fmt"
	"github.com/sirupsen/logrus"
	"os"
	"reflect"
	"strconv"
)

// 未知参数数量
func UnknownParams() {
	value := reflect.ValueOf(sum)
	if value.Kind() != reflect.Func {
		return
	}
	valueType := value.Type()
	argc := valueType.NumIn()
	// IsVariadic 报告函数类型的最终输入参数
	var sysArgs [] string
	if valueType.IsVariadic() {
		// 随机法
		//rand.Seed(time.Now().UnixNano())
		//argc += rand.Intn(10)
		// 系统参数法
		sysArgs = os.Args[1:]
		argc = len(sysArgs)
	} else {
		panic(fmt.Errorf("panic error parse IsVariadic Error"))
	}
	argv := make([]reflect.Value, argc)
	for i := range argv {
		atoi, err := strconv.Atoi(sysArgs[i])
		if err != nil {
			panic(fmt.Sprintf("args %d not trans int", i+1))
		}
		argv[i] = reflect.ValueOf(atoi)
	}

	result := value.Call(argv)
	logrus.Info(result[0].Int())
}


func sum(x, y int, xNumber ...int) int {
	sum := x + y
	for _, xNum := range xNumber {
		sum += xNum
	}
	return sum
}
```

### Runtime中运行方法

在runtime方法中运行
```
package reflect

import (
	"fmt"
	"reflect"
)

type AddSome func(int64, int64) int64

// 运行时函数修改
func RuntimeFunc()  {
	funcType := reflect.TypeOf(AddSome(nil))
	makeFunc := reflect.MakeFunc(funcType, func(args []reflect.Value) []reflect.Value {
		a := args[0].Int()
		b := args[1].Int()

		return []reflect.Value{reflect.ValueOf(a + b)}
	})

	addSome, ok := makeFunc.Interface().(AddSome)
	if !ok {
		return
	}
	fmt.Println(addSome(2, 3))
}
```
