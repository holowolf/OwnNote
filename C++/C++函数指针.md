# C++ 函数指针

函数指针是指向函数的指针变量

//TODO


用途为作为函数参数和回调函数

```C++
char * fun(char *p) {...} // 函数fun
char * (*pf)(char *p); // 函数指针pf
pf = fun; // 函数指针pf指向函数fun
pf(p); // 通过函数指针pf调用函数fun
```



