### typedef 用法

使用def 定义结构体别名
```c
struct Person
{
    char name[64];
    int age;
};

typedef struct Person myPerson;

typedef struct Person1
{
    char name[64];
    int age;
} myPerson1;

void typeof_todo_run()
{
    myPerson to;
    to.age = 23;
    strcpy(to.name, "Jack Simth");

    printf("%d\n", to.age);
    printf("%s\n", to.name);
}
```

### void数据类型
void 就是"无类型", void* 无类型指针, 无类型指针可以指向任何类型的数据.  
void 定义变量是没有意义的, 当你定义void a, 编译器会报错.  
void 真正用在两个方面 :   
> 对函数返回的限定  
对函数参数的限定

### 函数调用流程

宏函数 不是一个真的函数
```c
#define MYADD(x, y) ((x) + (y))
```

在内存动态存储区中分配nmemb块长度为size字节的连续区域, calloc自动分配内置0.
```c
#include<stdlib.h>
void *calloc(size_t nmemb, size_t size);
```

开辟空间, 并释放内存
```c
void calloc_todo()
{
    int *p = calloc(10, sizeof(int));
    for (int i = 0; i < 10; ++i)
    {
        p[i] = i + 1;
    }
    for (int i = 0; i < 10; ++i)
    {
        printf("%d", p[i]);
    }
    if (p != NULL)
    {
        free(p);
        p = NULL;
    }
}
``` 

重新分配用malloc或者calloc函数在堆中分配内存空间大小.
realloc 不会自动清理增加的内存, 需要手动清理, 如果指定地址后面有连续空间,  
那么就会在已有地址基础上增加内存, 如果指定的地址后面没有空间, 那么realloc会重新分配新的连续内存,  
把旧内存的值拷贝到新内存, 同时释放旧内存.
```c
#inclue<stdlib.h>
void *realloc(void *ptr, size_t size);
```

重新开辟
```c
void realloc_todo()
{
    int *p = malloc(sizeof(int) * 10);
    for (int i = 0; i < 10; ++i)
    {
        p[i] = i + 1;
    }
    for (int i = 0; i < 10; ++i)
    {
        printf("%d", p[i]);
    }
    printf("%d\n", p);
    p = realloc(p, sizeof(int) * 200);
    printf("%d\n", p);
    for (int i = 0; i < 15; ++i)
    {
        printf("%d", p[i]);
    }
}
```
### 数组指针用法

1. sizeof 和数组名取地址 &arr.
2. 这两种情况下数组名不是指向首元素的指针.
3. 在以上两种情况下,  数组名是数组类型
4. 除了上面两点之外, 数据名在其他任何情况下都是指向首元素的地址
```c
void printArray(int *arr, int len)
{
    for (int i = 0; i < len; ++i)
    {
        printf("%d", arr[i]);
        printf("%d", *(arr+i));
    }
}

void array_data_todo()
{
    int arr[] = {1, 2, 3, 4};
    printf("sizeof arr : %lu\n", sizeof(arr));
    printf("&arr addr : %d\n", &arr);
    printf("&arr addr : %d\n", &arr+1);

    int *p = arr;
    // 数组名是一个常量指针.
    printArray(arr, 4);
}
```

### 二维数组

// 对于二维数组同一维数组, 除了sizeof对数组名取地址外,那么数组名就是指向数组首元素的指针.



