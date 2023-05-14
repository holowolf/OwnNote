### & 与 * 的区别

测试 * 与 &
```
func testSymbol() {
	var a int = 1
	var b *int = &a
	var c **int = &b
	var x int = *b
	fmt.Println("a = ",a)
	fmt.Println("&a = ",&a)
	fmt.Println("*&a = ",*&a)
	fmt.Println("b = ",b)
	fmt.Println("&b = ",&b)
	fmt.Println("*&b = ",*&b)
	fmt.Println("*b = ",*b)
	fmt.Println("c = ",c)
	fmt.Println("*c = ",*c)
	fmt.Println("&c = ",&c)
	fmt.Println("*&c = ",*&c)
	fmt.Println("**c = ",**c)
	fmt.Println("***&*&*&*&c = ",***&*&*&*&*&c)
	fmt.Println("x = ",x)
}
```
#### 解释

##### 理论 &符号的意思是对变量取地址, 如：变量a的地址是&a
* 符号的意思是对指针取值, 如:*&a, 就是a变量所在地址的值, 当然也就是a的值了

##### 简单的解释 *和 & 可以互相抵消,同时注意, *&可以抵消掉, 但&*是不可以抵消的
* a和*&a是一样的, 都是a的值, 值为1 (因为*&互相抵消掉了)
* 同理, a和*&*&*&*&a是一样的, 都是1 (因为4个*&互相抵消掉了)

##### 展开 因为有
* var b *int = &a 所以 a和*&a和*b是一样的, 都是a的值, 值为1 (把b当做&a看)

##### 再次展开 因为有
* var c **int = &b  
* **c和**&b是一样的，把*&约去后会发现**c和*b是一样的 (从这里也不难看出，*c和b也是一样的)
* 又因为上面得到的*&a和*b是一样的
* **c和*&a是一样的，再次把*&约去后
* **c和a是一样的，都是1

