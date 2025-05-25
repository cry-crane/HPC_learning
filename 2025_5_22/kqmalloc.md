# Kqmalloc库函数说明

## 函数说明

KQMalloc专为单线程应用而涉及。此分配器只支持单线程应用，可极大地提升应用性能。KQMalloc是为鲲鹏920处理器涉及的，具有128字节L3缓存线大小，4KB系统页和2MB透明大页。该库当前已优化的函数有malloc、calloc、realloc、free、memcpy、memset。

## 使用方式

由于KQMalloc库优化的是标准库中的函数，所以只需要使用预加载的方式即可加速。  
`LD_PRELOAD=/usr/local/ksl/lib/libkqmalloc.so ./run_your_application`  
也可以动态加载到应用程序中，但不建议使用这种方法，因为与应用程序使用的另一个分配器共享虚拟地址空间有几个限制。

**注意**  
如需最高性能的使用KQMALLOC库，应在系统中设置开启透明大页，并设置大页大小为2MB。  

可通过设置大页相关环境变量变更默认使用大页数目，如:`export HUAWEI_SERIAL_GREAT_MALLOC_2MB_HUGE_PAGE_NUM=800`  

## malloc

内存分配:`void *malloc(size_t size)`;

**参数**  
|参数名|描述|取值范围|输入/输出|
|---|--|---|---|
|size|申请内存的字节数|非负数|输入|

**返回值**  
- 成功：返回指向分配内存的指针
- 失败：返回空指针

**示例**

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include "kqmalloc.h"

#define KQMALLOC_TEST_LEN 10

void MallocExample()
{
    int8_t *res = (int8_t *)malloc(KQMALLOC_TEST_LEN);
    if (res == NULL) {
        printf("res is null\n");
    } else {
        printf("malloc pointer address: %lx\n", res);
        free(res);
    }
}

int main(void) {
    MallocExample();
    return 0;
}
```

**运行结果**

`malloc pointer address: ffff97sr8174`  


## calloc

连续内存块分配并且初始化内存为0。  
`void *calloc(size_t number_of_objects,size_t size_of_objects)`;

**参数**

|参数名|描述|取值范围|输入/输出|
|--|--|--|--|
|number_of_objects|申请内存块的数量|非负数|输入|
|size_of_objects|申请内存卡的字节数|非负数|输入|

**返回值**

- 成功：返回指向分配内存的指针。
- 失败：返回空指针。

**示例**

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include "kqmalloc.h"

#define KQMALLOC_TEST_LEN 10

void CallocExample()
{
    int8_t *res = (int8_t *)calloc(KQMALLOC_TEST_LEN, KQMALLOC_TEST_LEN);
    if (res == NULL) {
        printf("res is null\n");
    } else {
        printf("calloc pointer address: %lx\n", res);
        free(res);
    }
}

int main(void) {
    CallocExample();
    return 0;
}
```

**运行结果**

`calloc pointer address: ffffafe4c690`

## realloc

内存重新分配并将旧内存中数据拷贝至新内存中。  
`void *realloc(void *old_pointer,size_t new_size)`

**参数**

|参数名|描述|取值范围|输入/输出|
|--|--|--|--|
|new_pointer|指向需重新分配的旧内存指针|非空|输入|
|new_size|重新申请内存的字节数|非负数|输入|

**返回值**

- 成功：返回指向新分配内存的指针
- 失败：返回空指针


**示例**

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include "kqmalloc.h"

#define KQMALLOC_TEST_LEN 10

void ReallocExample()
{
    int8_t *old_pointer = (int8_t *)malloc(KQMALLOC_TEST_LEN * sizeof(int8_t));
    if (old_pointer == NULL) {
        printf("old_pointer is null\n");
        return;
    } else {
        printf("old_pointer address: %lx\n", old_pointer);
        for (int8_t i = 0; i < KQMALLOC_TEST_LEN; ++i) {
            old_pointer[i] = i;
        }
        printf("old_pointer test data:");
        for (int8_t i = 0; i < KQMALLOC_TEST_LEN; ++i) {
            printf(" %d", old_pointer[i]);
        }
        printf("\n");
    }

    int8_t *new_pointer = (int8_t *)realloc(old_pointer, 2 * KQMALLOC_TEST_LEN * sizeof(int8_t));
    if (new_pointer == NULL) {
        printf("new_pointer is null\n");
        free(old_pointer);
        return;
    } else {
        printf("new_pointer address: %lx\n", new_pointer);
        printf("new_pointer test data:");
        for (int8_t i = 0; i < KQMALLOC_TEST_LEN; ++i) {
            printf(" %d", new_pointer[i]);
        }
        printf("\n");
    }
    free(old_pointer);
    free(new_pointer);
}

int main(void) {
    ReallocExample();
    return 0;
}
```


**运行结果**

```
old_pointer address: ffff921c8174
old_pointer test data: 0 1 2 3 4 5 6 7 8 9
new_pointer address: ffff921c87ec
new_pointer test data: 0 1 2 3 4 5 6 7 8 9 
```

## free

释放已分配内存。  
`void free(void *p);`

**参数**  

|参数|描述|取值范围|输入/输出|
|--|--|--|--|
|p|指向需释放内存的指针|非空|输入|


## memcpy

从原内存中拷贝指定长度字节数的数据至目标内存中，  
`void *memcpy(void *destination,const void *source,size_t size);`

**参数**

|参数名|描述|取值范围|输入/输出|
|--|--|--|--|
|destination|指向用于存储复制内容的目标内存指针|非空|输出|
|size|被复制的字节数|非负数|输入|

**返回值**

- 成功：返回指向目标存储区内存的指针
- 失败：返回空指针

**示例**  

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#include "kqmalloc.h"

#define KQMALLOC_TEST_LEN 10

void MemcpyExample()
{
    int8_t *source = (int8_t *)malloc(KQMALLOC_TEST_LEN * sizeof(int8_t));
    if (source == NULL) {
        printf("source is null\n");
        return;
    } else {
        printf("source address: %lx\n", source);
        for (int8_t i = 0; i < KQMALLOC_TEST_LEN; ++i) {
            source[i] = i;
        }
        printf("source test data:");
        for (int8_t i = 0; i < KQMALLOC_TEST_LEN; ++i) {
            printf(" %d", source[i]);
        }
        printf("\n");
    }

    int8_t *destination = (int8_t *)malloc(KQMALLOC_TEST_LEN * sizeof(int8_t));
    if (destination == NULL) {
        printf("destination is null\n");
        free(source);
        return;
    } else {
        printf("destination address: %lx\n", destination);
        printf("destination test data before memcpy:");
        for (int8_t i = 0; i < KQMALLOC_TEST_LEN; ++i) {
            printf(" %d", destination[i]);
        }
        printf("\n");
    }
    int8_t *res = memcpy(destination , source, KQMALLOC_TEST_LEN * sizeof(int8_t));
    printf("res address: %lx\n", res);
    if (res != NULL) {
        printf("res test data after memcpy:");
        for (int8_t i = 0; i < KQMALLOC_TEST_LEN; ++i) {
            printf(" %d", res[i]);
        }
        printf("\n");
    }

    free(source);
    free(destination);
}

int main(void) {
    MemcpyExample();
    return 0;
}
```

**运行结果**


```cpp
source address: ffffbb202174
source test data: 0 1 2 3 4 5 6 7 8 9
destination address: ffffbb202168
destination test data before memcpy: 92 33 32 -69 -1 -1 0 0 0 0
res address: ffffbb202168
res test data after memcpy: 0 1 2 3 4 5 6 7 8 9
```

## memset

将目标内存中指定长度字节数的数据设置为指定的值

`void *memset(void *destination,int value,size_t size)`

**参数**

|参数名|描述|取值范围|输入/输出|
|--|--|--|--|
|destination|指向用于存储重置内容的目标内存指针|非空|输出|
|value|重置的目标值|任意值|输入|
|size|被重置的字节数|非负数|输入|

**返回值**

- 成功： 返回指向目标存储区内存的指针
- 失败： 返回空指针

**示例**

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#include "kqmalloc.h"

#define KQMALLOC_TEST_LEN 10

void MemsetExample()
{
    int8_t *destination = (int8_t *)malloc(KQMALLOC_TEST_LEN * sizeof(int8_t));
    if (destination == NULL) {
        printf("destination is null\n");
        return;
    } else {
        printf("destination address: %lx\n", destination);
        for (int8_t i = 0; i < KQMALLOC_TEST_LEN; ++i) {
            destination[i] = i;
        }
        printf("destination test data before memcpy:");
        for (int8_t i = 0; i < KQMALLOC_TEST_LEN; ++i) {
            printf(" %d", destination[i]);
        }
        printf("\n");
    }
    int8_t *res = memset(destination , 1, KQMALLOC_TEST_LEN * sizeof(int8_t));
    printf("res address: %lx\n", res);
    if (res != NULL) {
        printf("res test data after memcpy:");
        for (int8_t i = 0; i < KQMALLOC_TEST_LEN; ++i) {
            printf(" %d", res[i]);
        }
        printf("\n");
    }

    free(destination);
}

int main(void) {
    MemsetExample();
    return 0;
}
```

**运行结果**

```cpp
destination address: ffff9fe17174
destination test data before memcpy: 0 1 2 3 4 5 6 7 8 9
res address: ffff9fe17174
res test data after memcpy: 1 1 1 1 1 1 1 1 1 1
```


## kqMallocGetVersion

获取KQMalloc产品版本信息。  
`KqMallocResult KqMallocGetVersion(KqMallocProVersion *packageInfo);`

**参数**

|参数名|描述|取值范围|输入/输出|
|--|--|--|--|
|packageInfo|指向产品信息结构体的指针|非空|输出|

**返回值**

- 成功：返回KQMALLOC_STS_NO_ERR。
- 失败：返回错误码。

**错误码**

|错误码|描述|
|--|--|
|KQMALLOC_STS_NULL_PTR_ERR|KqMallocProVersion指针为空指针|


**示例**

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include "kqmalloc.h"

void GetProductVersionExample()
{
    KqMallocProVersion versionGet;
    KqMallocResult result = KqMallocGetVersion(&versionGet);
    if (result == KQMALLOC_STS_NO_ERR) {
        printf("Product Name: %s\n", versionGet.productName);
        printf("Product Version: %s\n", versionGet.productVersion);
        printf("Component Name: %s\n", versionGet.componentName);
        printf("Component Version: %s\n", versionGet.componentVersion);
        printf("Component AppendInfo: %s\n", versionGet.componentAppendInfo);
        printf("Software Name: %s\n", versionGet.softwareName);
        printf("Software Version: %s\n", versionGet.softwareVersion);
    }
}

int main(void) {
    GetProductVersionExample();
    return 0;
}
```

**运行结果**

```cpp
Product Name: Kunpeng Boostkit
Product Version: 23.0.0
Component Name: BoostKit-ksl
Component Version: 2.1.0
Component AppendInfo: gcc
Software Name: boostkit-ksl-kqmalloc
Software Version: 2.1.0
```

