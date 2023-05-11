# C++中的四种智能指针

智能指针的作用是管理一个指针, 避免申请空间在函数结束时忘记释放, 造成内存泄漏这中情况发生.

使用智能指针可以避免此种问题的发生, 因为智能指针就是一个类, 当超出一个作用域的时候, 类会调用析构函数, 析构函数会自动释放资源. 

> todo 类的作用域有哪些

所以智能指针的作用原理就是在函数结束时,自动释放内存空间, 不需要手动释放内存空间.

常用接口

```C++

T* get();
T& operator*();
T* operator->();
T& operator=(const T& val);
T* release();
void reset(T* ptr = nullptr);
```

* T 是模板参数，也就是传入的类型;
* get()用来获取auto_ptr封装在内部的指针, 也就是获取原生的指针;
* operator()重载, operator->()重载了->,operator=()重载了=;
* release()将auto_ptr封装在内部的指针置为nullptr,但是不会破坏指针所指向的内容，函数会返回的内部指针空之前的值;
* 直接释放方庄的内部指针所指向的内存, 如果指定了ptr的值, 则将内部指针初始化为该值 (否则设置为nullptr);

1. auto_ptr (C++98方案, C11放弃) 采用所有权模式.
    ```C++
    auto_ptr<std::string> p1 (new string("hello"));
    auto_ptr<std::string> p2;
    p2 = p1; //auto_ptr 不会报错
    ```
    此时不会报错， p2剥夺了p1的所有权, 但程序运行时访问p1将会报错. 所以auto_ptr的缺点是: 存在潜在内存崩溃问题.

2. unique_ptr(替换auto_ptr)
    unique_ptr 实现独占式拥有或严格拥有概念, 保证同一时间只有一个智能指针可以指向该对象. 避免资源泄漏有用.
    
    // todo
   
    因此, unique_ptr 比 auto_ptr 更安全.

3. shared_ptr (共享型,强引用)
   shared_ptr 实现共享式拥有概念, 多个智能指针可以指向相同对象,该对象和其他相关资源会在`最后一个引用被销毁`时释放.
   // todo
   shared_ptr为了解决auto_ptr在对象所有权上的局限性(auto_ptr是独占的),在使用引用计数的机制上提供了可以共享所有权的智能指针.

4. weak_ptr (弱引用)
   // todo







