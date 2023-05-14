## Goland Tips 提高效率


### 1. 快速实现Interface

操作步骤:

1. 光标移动到struct 名称上
2. Alt/Option + Enter
3. 选择Implement Interface … Control+I
4. 搜索你需要实现的interface

![image](https://note.youdao.com/yws/res/4103/848963B5B8264FAE9AB1E35EBA3B6FC4)


### 2. 快速抽象Interface

操作步骤:

1. 右键 struct 名称
2. 选择 Refactor->Extract->Interface
3. 选择要抽象的方法,填写interface名称

![image](https://note.youdao.com/yws/res/4107/F64F3D5B5EEB4B81A70967BA39D45D6E)


### 3. 快速填充Struct

操作步骤: 
1. 把你的光标放在{}中间 
2. Alt/Option + Enter 
3. 选择Fill Struct 或者 Fill Struct Recursively(递归填充)

![image](https://note.youdao.com/yws/res/4108/B539CBCAF03B46F480875ED86EB51181)

### 4.快速Struct工厂方法

操作步骤:

1. 光标移动到struct 名称上
2. Alt/Option + Enter
3. Generate Constructor
4. 选择属性

![image](https://note.youdao.com/yws/res/4105/5142777A3490427296E45C7CD26001E0)

### 5.快速生成TestCase文件

需要`go get golang.org/x/tools/imports go get github.com/cweill/gotests`

操作步骤:

1. 光标移动到Method/Function上
2. Command/Control+Shift+T

![image](https://note.youdao.com/yws/res/4104/0213AAB6CB04488DA4911D18C69A430E)


### 6.Live Template Code Fly

实时代码模板只是为了让我们更加高效的写一些固定模式的代码，以提高编码效率，同时也可以增加个性化. 
调用常规的实时代码模板主要是通过两个快捷键：Tab 和 Ctrl + J.
虽然 IntelliJ IDEA 支持修改此对应的快捷键，但是默认大家都是这样使用的，所以没有特别原因就不要去改 
该两个快捷键的使用方法：在输入模板的缩写名称后按 Tab 键，即立即生成预设语句.
如果按 Ctrl + J 则会先提示与之匹配的实时代码模板介绍，然后还需按 Enter 才可完成预设语句的生成

![image](https://note.youdao.com/yws/res/4106/73799D03E7C942D095F055F8AB5990CE)