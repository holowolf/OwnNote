## reflect反射的实际应用

### 1. 实际解决日志方案

```
HOST;000012000629948340196501;ipv4;3; ips: user_id=2;user_name=172.21.1.102;policy_id=1;src_mac=52:54:00:62:7f:4a;dst_mac=58:69:6c:7b:fa:e7;src_ip=172.21.1.102;dst_ip=172.22.2.3;src_port=48612;dst_port=80;app_name=网页浏览(HTTP);protocol=TCP;app_protocol=HTTP;event_id=1310909;event_name=Microsoft_IIS_5.1_Frontpage扩展路径信息漏洞;event_type=安全漏洞;level=info;ctime=2019-12-26 11:17:17;action=pass
```

规范化结构体

```go
type IpsItem struct {
	UserId int `json:"user_id"`
	UserName string `json:"user_name"`
	SrcIp string `json:"src_ip"`
	DstIp string `json:"dst_ip"`
	SrcPort int `json:"src_port"`
	DstPort int `json:"dst_port"`
	AppName string `json:"app_name"`
	Protocol string `json:"protocol"`
	AppProtocol string `json:"app_protocol"`
	EventId int `json:"event_id"`
	EventName string `json:"event_name"`
	EventType string `json:"event_type"`
	Level string `json:"level"`
	Ctime string `json:"ctime"`
	Action string `json:"action"`
}
```

如果上面日志文件是json就非常容易解决了. 因为golang 标准库使用的就是 reflect反射生成 `struct`.

所以我的思路也是使用`reflect`反射实现字符串转换成结构化的数据,你也可以大致了解标准库json.Unmarshal的原理.

### 2. 代码实验

```go
package reflect_example

import (
	"fmt"
	"reflect"
	"strings"
)

type IpsItem struct {
	UserId int `json:"user_id"`
	UserName string `json:"user_name"`
	SrcIp string `json:"src_ip"`
	DstIp string `json:"dst_ip"`
	SrcPort int `json:"src_port"`
	DstPort int `json:"dst_port"`
	AppName string `json:"app_name"`
	Protocol string `json:"protocol"`
	AppProtocol string `json:"app_protocol"`
	EventId int `json:"event_id"`
	EventName string `json:"event_name"`
	EventType string `json:"event_type"`
	Level string `json:"level"`
	Ctime string `json:"ctime"`
	Action string `json:"action"`
}


func ReflectExample(raw string) *IpsItem {
	// 清除非法字符
	raw = strings.ReplaceAll(raw, ":", ";")
	ins := IpsItem{}
	typeOf := reflect.TypeOf(ins)

	// 遍历结构体属性
	for i := 0; i < typeOf.NumField(); i++ {
		// 获取属性structField
		structField := typeOf.Field(i)
		// 属性名称
		fieldName := structField.Name
		// tag json的值
		tagName := structField.Tag.Get("json")
		// 获取field值
		value := reflect.ValueOf(&ins).Elem().FieldByName(fieldName)

		// 属性的值 type
		switch structField.Type.Name() {
		case "int":
			var someInt int64
			scanValueFromString(raw, tagName, tagName+"=%d", &someInt)
			// 给属性赋值
			value.SetInt(someInt)
			// todo:: 支持更多类型
			break
		default:
			var someString string
			scanValueFromString(raw, tagName, tagName+"=%s", &someString)
			// 给属性赋值
			value.SetString(someString)
			break
		}
	}
	return &ins
}

func scanValueFromString(raw string, tagJsonValue, format string, someV interface{})  {
	for _, ss := range strings.Split(raw, ";") {
		element := strings.TrimSpace(ss)
		if strings.HasPrefix(element, tagJsonValue) {
			_, _ = fmt.Sscanf(element, format, someV)
			return
		}
	}
}
```

### 3. 使用常用场景

* 使用反射开发`gorm.model`的自动文档工具
* 开发自己的`json/ini/yml/toml`等格式的序列化库
* 开发自己`nginx`日志收集库