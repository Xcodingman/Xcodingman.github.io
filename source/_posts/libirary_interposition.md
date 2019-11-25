---
title: Library Interpostion
date: 2019/11/17
---
# Library Interposition
最近在看CSAPP的第七章Linking，其中提到对库函数调用的修改（library interposition),特意记录下来与大家分享，并让自己加深巩固理解。
首先，一共有三种interpostion的方法，分别是在编译阶段，链接阶段和运行阶段实现对调用函数的修改，他们之间的不同可以用下面这张表格来表示

|注入时机|注入需要|
|:--:|:--:|
|编译时|文件源代码|
|链接时|可重定位目标文件|
|运行时|可执行目标文件|
***
## 编译时注入
编译时注入需要文件的源代码，这里引用csapp里面的代码,我们运行的代码为int.c,在没有注入的情况下代码里调用libc.so中的malloc和free函数，文件mymalloc.c里面编写的时我们注入使用到的malloc和free函数，并在我们自己编写的malloc.h里面声明。
编译时，我们首先编译mymalloc.c成可重定位目标文件，然后将可重定位目标文件mymalloc.o和int.c编译生成最后的可执行文件。命令如下：
```
gcc -DCOMPILETIME -c mymalloc.c
gcc -I. -o intc int.c mymalloc.o 
```
第一行中-c代表生成可重定位目标文件，-DCOMPILETIME对应mymalloc.c中的#ifdef COMPILETIME。第二行中的-I.是关键，它告诉编译器在预编译阶段先从"."目录中寻找malloc.h，然后再去系统目录下寻找。所以这导致int.c中的include malloc.c为我们定义的malloc.h。这样的话，malloc和free函数就在malloc.h中被替换成了mymalloc和myfree函数。
因为-I是在预编译阶段对malloc.h进行偷梁换柱，所以必须需要源文件的源代码才能使得注入成功。
```
//int.c
#include <stdio.h>
#include <malloc.h>

int main(){
    
    int *p=malloc(32);
    printf("complete task\n");
    free(p);
    return 0;
}
```
```
//malloc.h
#define malloc(size) mymalloc(size)
#define free(ptr) myfree(ptr)

void *mymalloc(size_t size);
void myfree(void *ptr);
```
```
//mymalloc.c
#ifdef COMPILETIME
#include <stdio.h>
#include <malloc.h>

//malloc wrapper function 

void *mymalloc(size_t size){
    void *ptr = malloc(size);
    printf("malloc(%d)=%p\n",(int)size,ptr);
    return ptr;
}

//free wrapper function
void myfree(void *ptr){
    free(ptr);
    printf("free(%p)\n",ptr);
}
#endif
```
运行./intc，可以看到结果如下：
```
saxon@ubuntu:~/Desktop/library_interposition$ gcc -DCOMPILETIME -c mymalloc.c 
saxon@ubuntu:~/Desktop/library_interposition$ gcc -I. -o int int.c mymalloc.o 
saxon@ubuntu:~/Desktop/library_interposition$ ./int 
malloc(32)=0x5612ed48b260
complete task
free(0x5612ed48b260)
```
## 链接时注入
与编译时注入不同，链接时注入发生在链接阶段，链接阶段是对两个可重定位目标文件进行链接，可以对符号进行重定位。在注入前，我们需要准备如下的mymalloc.c文件并对它进行编译成可重定位文件。里面的函数定义与链接器的命令有关。编译命令如下：
```
gcc -DLINKTIME -c mymalloc.c
gcc -c int.c
```
```
//mymalloc.c
#ifdef LINKTIME
#include <stdio.h>
//#include <malloc.h>
void *__real_malloc(size_t size);
void __real_free(void *ptr);
//malloc wrapper function 

void *__wrap_malloc(size_t size){
    void *ptr = __real_malloc(size);
    printf("malloc(%d)=%p\n",(int)size,ptr);
    return ptr;
}

//free wrapper function
void __wrap_free(void *ptr){
    __real_free(ptr);
    printf("free(%p)\n",ptr);
}
#endif
```
当我们获得两个可重定位目标文件后，然后就是链接器的工作了。我们从命令上来分析链接器如何完成注入，命令如下：
```
gcc -Wl,--wrap,malloc -Wl,--wrap,free -o intl int.o mymalloc.o 

```
-Wl告诉编译器将后面的参数传给链接器，--wrap参数会将符号f解析为__wrap_f，而将__real_f解析成f,在我们的命令里就是会把malloc解析成__wrap_malloc，而__real_malloc则会被解析成malloc（就是libc里面的malloc函数）这样一来，就完成了malloc函数的注入，free也是同样的道理。最后运行的结果如下。
```
saxon@ubuntu:~/Desktop/library_interposition$ gcc -DLINKTIME -c mymalloc.c
saxon@ubuntu:~/Desktop/library_interposition$ gcc -c int.c 
saxon@ubuntu:~/Desktop/library_interposition$ gcc -Wl,--wrap,malloc -Wl,--wrap,free -o intl int.o mymalloc.o 
saxon@ubuntu:~/Desktop/library_interposition$ gcc -Wl,--wrap,malloc -Wl,--wrap,free -o intl int.o mymalloc.o 
saxon@ubuntu:~/Desktop/library_interposition$ ./intl 
malloc(32)=0x55cb46bac260
complete task
free(0x55cb46bac260)
```
## 运行时注入
由于编译时注入和链接时注入分别需要源文件的源代码和源文件的可重定位目标文件，在现实中应用的场景不多，而运行时注入只需要源文件的可执行文件即可完成注入。这种机制的实现得益于动态链接器的一个系统变量LD_PRELOAD,程序在寻找动态链接库之前，会先到这个变量定义的路径中去寻找，然后再去系统定义的路径中去寻找动态链接库。mymalloc.c文件如下：
```
//mymalloc.c
#ifdef RUNTIME
#define _GNU_SOURCE
#include <stdio.h>
#include <stdlib.h>
#include <dlfcn.h>

/* malloc wrapper function */
void *malloc(size_t size)
{
    void *(*mallocp)(size_t size);
    char *error;
    static __thread int print_times = 0;
    print_times++;
    
    mallocp = dlsym(RTLD_NEXT, "malloc"); /* Get address of libc malloc */
    if ((error = dlerror()) != NULL) {
        fputs(error, stderr);
        exit(1);
    }
    char *ptr = mallocp(size); /* Call libc malloc */
    if (print_times == 1)
    {
        printf("malloc(%d) = %p\n", (int)size, ptr);
    }
    return ptr;
}

/* free wrapper function */
void free(void *ptr)
{
    void (*freep)(void *) = NULL;
    char *error;

    if (!ptr)
        return;

    freep = dlsym(RTLD_NEXT, "free"); /* Get address of libc free */
    if ((error = dlerror()) != NULL) {
        fputs(error, stderr);
        exit(1);
    }
    freep(ptr); /* Call libc free */
    printf("free(%p)\n", ptr);
}
#endif
/* $end interposer */
```
这里，和csapp里面的源代码有点不同，不同之处在于静态变量print_times，在csapp里面没有这个变量。由于新的malloc函数中调用了printf函数，而printf函数在实现时可能也调用了malloc函数，这会导致两个函数不断的互相调用对方，最后导致栈溢出，报出segmentation fault。引入print_times后，由于是静态变量，只有当第一次调用malloc是，才会调用printf函数，避免了互相调用的死循环。
首先，我们编译动态库和可执行文件：
```
gcc -DRUNTIME -shared -fpic -o mymalloc.so mymalloc.c -ldl
gcc -o intr int.c
```
然后在运行代码时指定LD_PRELOAD变量并运行可执行文件./intr
```
LD_PRELOAD="./mymalloc.so"  ./intr 
```
最后发现运行结果如下：
```
saxon@ubuntu:~/Desktop/library_interposition$ LD_PRELOAD="./mymalloc.so"  ./intr 
malloc(32) = 0x5583182ab260
complete task
free(0x5583182ab260)
```











