---
title: C++内存错误检测利器--AddressSanitizer
date: 2023-12-05T13:27:10+08:00
lastmod: 2023-12-05T13:27:10+08:00
draft: false
tags:
  - C++
---
# 一、概述

## 1、常见内存问题场景

-   **野指针：**指针未初始化就使用(非法的随机值)、指针越界非法访问，或指向一个已释放的对象等。
-   **内存泄露：**申请的堆内存使用完毕后忘记释放，内存还占着，但地址丢失，自己已经不能控制这块内存，而系统也不能再次将它分配给需要的程序。内存泄漏次数多了就会导致内存溢出。
-   **内存溢出：**Out Of Memory，简称OOM，指系统已经不能再分配出你所需要的空间。
-   **内存踩踏：**指访问了不合法的地址（访问了不属于自己的地址），如果访问的地址是其他变量的地址并进行了修改，就会破坏别人的数据，从而导致程序运行异常。常发生在buffer overflow，野指针操作，write after free等场景。

## 2、常见内存检测工具

在AddressSanitizer出现之前，市面上就已经存在了许多内存检测器，例如：

-   Dr.Memory：检测未初始化的内存访问、double free、use after free 等错误
-   Mudflap：检测指针的解引用，静态插桩
-   Insure++：检测内存泄漏
-   Valgrind：可以检测非常多的内存错误

其中，Dr.Memory、Insure++ 和 Mudflap 虽然在运行时造成的额外损耗比较少，但是检测场景有限；Valgrind 虽然能够在许多场景的检测出错误，但是它实现了自己的一套 ISA 并在其之上运行目标程序，因此它会严重拖慢目标程序的速度。而 AddressSanitizer 在设计时就综合考虑了检测场景、速度的影响因素，结合了 Mudflap 的静态插桩、Valgrind 的多场景检测能力，故本文主要讲解AddressSanitizer。

## 3、什么是AddressSanitizer

AddressSanitizer即地址消毒技术，简称ASan，是一个快速的内存错误检测工具。它可以用来检测内存问题，例如缓冲区溢出或对悬空指针的非法访问等。

检测类型：

-   **Use after free(dangling pointer dereference)：**释放后使用（堆上分配的空间free之后被再次使用）。
-   **Heap buffer overflow：**堆缓冲区溢出（访问的区域在堆上, 且超过了分配的空间）。
-   **Stack buffer overflow：**栈缓冲区溢出（访问的区域在栈上, 且超过了分配给它的空间）。
-   **Global buffer overflow：**全局缓冲区溢出（访问的区域是全局变量, 且超过了分配给它的空间）。
-   **Use after return：**Return后使用（函数在栈上的局部变量在函数返回后被使用默认不开启）。
-   **Use after scope：**在作用域外使用（局部变量离开作用域以后继续使用）。
-   **Initialization order bugs：**初始化顺序错误（检查全局变量或静态变量初始化的时候有没有利用未初始化的变量，默认不开启）。
-   **Memory leaks：**内存泄漏（未释放堆上分配的内存）。

据谷歌的工程师介绍 ，ASan 已在 chromium 项目上检测出了300多个潜在的未知bug，而且在使用 ASan 作为内存错误检测工具对程序性能损耗也是及其可观的。根据检测结果显示可能导致性能降低2倍左右，比Valgrind（官方给的数据大概是降低10-50倍）快了一个数量级。而且相比于Valgrind只能检查到堆内存的越界访问和悬空指针的访问，ASan 不仅可以检测到堆内存的越界和悬空指针的访问，还能检测到栈和全局对象的越界访问。这也是 ASan 在众多内存检测工具的比较上出类拔萃的重要原因，基本上现在 C/C++ 项目都会使用ASan来保证产品质量，尤其是大项目中更为需要。

从gcc 4.8开始，AddressSanitizer成为gcc的一部分，但还不完善。要获得更好的体验，建议使用4.9及以上版本。

## 4、AddressSanitizer检测原理

ASan接管了每次内存分配/释放，并且每一次对内存的读/写都加上了一个检查 (需要编译器的配合)。

**算法思路**：如果想防住Buffer Overflow漏洞，只需要在每块内存区域右端（或两端，能防overflow和underflow）加一块区域（RedZone），使RedZone的区域的影子内存（Shadow Memory)设置为不可写即可。
![](https://i.imgur.com/jVPnrV9.png)


[![](https://blog.yanjingang.com/wp-content/uploads/2023/03/asan-shadow-memory-1024x610.png)](https://blog.yanjingang.com/wp-content/uploads/2023/03/asan-shadow-memory.png)

防护缓冲区溢出的基本步骤：

-   在被保护的全局变量、堆、栈前后创建 redzone，并将 redzone 标记为中毒状态。
-   将缓冲区和 redzone 每 8 字节对应 1 字节的映射方式建立影子内存区（影子内存区使用函数 MemToShadow 获取）。
-   出现对 redzone 的访问（读写执行）行为时，由于 redzone 对应的影子内存区被标记为中毒状态触发报错。
-   报错信息包含发生错误的进程号、错误类型、出错的源文件名、行号、函数调用关系、影子内存状态。其中影子内存状态信息中出错的部分用中括号标识出来。
-   中毒状态：内存对应的 shadow 区标记该内存不能访问的状态。

![](https://blog.yanjingang.com/wp-content/uploads/2023/03/asan.png "asan")

ASan主要包括两部分：插桩(Instrumentation)和动态运行库(Run-time library)。

-   **插桩：**主要是针对在llvm编译器级别对访问内存的操作(store，load，alloca等)，将它们进行处理。为了防止buffer overflow，需要将原来分配的内存两边分配额外的内存Redzone，并将这两边的内存加锁，设为不能访问状态(中毒状态)。
-   **动态运行库：**主要提供一些运行时的复杂的功能(比如poison/unpoison shadow memory)以及将malloc,free等系统调用函数hook住。在使用函数 free 释放内存时，所释放的内存被隔离开来（暂时不会被分配出去），并被标记为与RedZone相同的中毒状态，中毒的内存一旦被访问，即可被检测到。ASan 使用 shadow memory 跟踪哪些字节为正常内存，哪些字节为中毒内存。字节可以标记为完全正常（shadow memory 值为 0）、完全中毒（shadow memory 值为负值）或前面 k 个字节未中毒（shadow memory 值为 k）。如果 shadow memory 显示某个字节中毒，则 ASan 会使程序崩溃，并输出有用的调试信息，包括调用堆栈、影子内存映射、内存违例类型、读取或写入的内容、导致违例的计算机以及内存内容。

插桩示例：

```
// 原始代码：
void foo() {
  char a[8];
  ...
  return;
}

// 插桩后的检测代码：
void foo() {
  char redzone1[32];  // 32-byte aligned
  char a[8];          // 32-byte aligned
  char redzone2[24];
  char redzone3[32];  // 32-byte aligned
  int  *shadow_base = MemToShadow(redzone1);
  shadow_base[0] = 0xffffffff;  // poison redzone1
  shadow_base[1] = 0xffffff00;  // poison redzone2, unpoison 'a'
  shadow_base[2] = 0xffffffff;  // poison redzone3
  ...
  shadow_base[0] = shadow_base[1] = shadow_base[2] = 0; // unpoison all
  return;
}
```

从以上示例中可以看到ASan将malloc/free函数进行了替换，在malloc函数中额外的分配了Redzone区域的内存，将与Redzone区域对应的影子内存加锁，主要的内存区域对应的影子内存不加锁。free函数将所有分配的内存区域加锁，并放到了隔离区域的队列中(保证在一定的时间内不会再被malloc函数分配)，可检测Use after free类的问题。

# 二、使用

#### 1、用法

###### 1.1、启用AddressSanitizer

用-fsanitize=address选项编译和链接你的程序，用-fno-omit-frame-pointer编译，以得到更容易理解stack trace：

```
gcc -Werror-rdynamic-fsanitize=address -fno-omit-frame-pointer -g test.cc -o test
```

或在CMakeLists.txt中配置：

```
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror -rdynamic -fsanitize=address -fno-omit-frame-pointer -g")
```

###### 1.2、export编译选项

ASAN\_OPTIONS是Address-Sanitizier的运行选项环境变量，可以根据需要选择性设置：

-   halt\_on\_error=0：检测内存错误后继续运行
-   detect\_leaks=1:使能内存泄露检测
-   malloc\_context\_size=15：内存错误发生时，显示的调用栈层数为15
-   log\_path=/home/xos/asan.log:内存检查问题日志存放文件路径
-   suppressions=$SUPP\_FILE:屏蔽打印某些内存错误
-   detect\_stack\_use\_after\_return=1：检查访问指向已被释放的栈空间
-   handle\_segv=1：处理段错误；也可以添加handle\_sigill=1处理SIGILL信号
-   quarantine\_size=4194304:内存cache可缓存free内存大小4M

例如：

```
export ASAN_OPTIONS=halt_on_error=0:detect_leaks=1:malloc_context_size=15:log_path=./asan.log
```

#### 2、场景测试

###### 2.1、 (heap) use after free 释放后使用

下面的代码中，分配array数组并释放，然后返回它的一个元素。

```
$ vim test_mem.cc
1 /**
2  * @filetest_mem.cc
3  * @brief   mem问题asan检测测试
4  * @authoryanjingang
5  * @date2023-3-13
6  * @note    g++ test_mem.cc -std=c++11  -Wall  -Werror -rdynamic -fsanitize=address -fno-omit-frame-pointer -g -o build/test_mem
7  */
8 
9 
10 // 堆上分配的空间被free之后再次使用
11 int use_after_free(){
12    int *array = new int[100];
13    delete[] array;
14    return array[1];
15 }
16 
17 int main(int argc, char **argv){
18     use_after_free();
19 }


// build
$ g++ test_mem.cc -std=c++11  -Wall  -Werror -rdynamic -fsanitize=address -fno-omit-frame-pointer -g -o build/test_mem

// test
$ ./build/test_mem
```

从下图提示的错误信息中，我们可以非常明确的看到内存异常访问信息：

-   **ERROR：**异常类型为heap-use-after-free堆内存释放后被使用。
-   **READ：**异常操作类型为读，在T0线程，位置在test\_mem.cc:14行。
-   **freed：**内存释放位置在test\_mem.cc:13行。
-   **previously allocated：**内存分配位置在test\_mem.cc:12行。
-   **fa/fd：**最下方的堆内存中，fa表示Redzone防护缓冲区，fd表示已被free释放的堆内存区域。

[![](https://blog.yanjingang.com/wp-content/uploads/2023/03/use-after-free-1024x737.png)](https://blog.yanjingang.com/wp-content/uploads/2023/03/use-after-free.png)

###### 2.2、 heap buffer overflow 堆缓存访问溢出

如下代码中，访问的位置超出堆上数组array的边界。

```
17  // 堆缓冲区溢出
18  int heap_buffer_overflow(){
19     int* array = new int[100];
20     int res = array[100];
21     delete [] array;
22     return res;
23 }
...
28 heap_buffer_overflow();
```

下图提示的错误信息指出：

-   **ERROR：**异常类型为heap-buffer-overflow堆缓冲区溢出。
-   **READ：**异常操作类型为读，在T0线程，位置在test\_mem.cc:20行。
-   **allocated：**内存分配位置在test\_mem.cc:19行。
-   **fa：**最下方的堆内存中，fa表示Redzone防护缓冲毒区，被异常访问。

[![](https://blog.yanjingang.com/wp-content/uploads/2023/03/heap-buffer-overflow-1024x598.png)](https://blog.yanjingang.com/wp-content/uploads/2023/03/heap-buffer-overflow.png)

###### 2.3、 stack buffer overflow 栈缓存访问溢出

如下代码中，访问的位置超出栈上数组array的边界。

```
24 // 栈缓冲区溢出
25 int stack_buffer_overflow(){
26     int array[100];
27     return array[100];
28 }
...
35 stack_buffer_overflow();
```

下图提示的错误信息指出：

-   **ERROR：**异常类型为stack-buffer-overflow栈缓冲区溢出。
-   **READ：**异常操作类型为读，在T0线程，位置在test\_mem.cc:27行。
-   **Address：**栈块在线程T0的栈上448偏移位置上，Memory access at offset 448 overflows this variable。
-   **f1/f3：**f1为Stack Left Redzone防护缓冲毒区，f3为Stack Right Redzone防护缓冲毒区，这里被异常访问。

[![](https://blog.yanjingang.com/wp-content/uploads/2023/03/stack-buffer-overflow-1024x598.png)](https://blog.yanjingang.com/wp-content/uploads/2023/03/stack-buffer-overflow.png)

###### 2.4、 global buffer overflow 全局缓冲访问溢出

如下代码中，访问的位置超出全局数组array的边界。

```
29 // 全局缓冲区溢出
30 int array[100];
31 int global_buffer_overflow(){
32     return array[100];
33 }
...
40 global_buffer_overflow();
```

下图提示的信息指出：

-   **ERROR：**异常类型为global-buffer-overflow全局缓冲区溢出。
-   **READ：**异常操作类型为读，在T0线程，位置在test\_mem.cc:32行。
-   **global variable：**全局缓存块在test\_mem.cc:30行定义。
-   **f9：**f9为Global Redzone防护缓冲毒区，这里被异常访问。

[![](https://blog.yanjingang.com/wp-content/uploads/2023/03/global-buffer-overflow-1024x500.png)](https://blog.yanjingang.com/wp-content/uploads/2023/03/global-buffer-overflow.png)

###### 2.5、 memory leaks 内存泄露

检测内存的LeakSanitizer是集成在AddressSanitizer中的一个相对独立的工具，它工作在检查过程的最后阶段。下面代码中，p指向的内存没有释放。

```
34 // 内存泄漏
35 void* p;    // p指向的内存没有释放
36 int memory_leaks(){
37     p = malloc(7);
38     p = 0;
39     return 0;
40 }
...
49 memory_leaks();
```

下图的错误信息指出：

-   异常类型为memory leaks内存泄漏。
-   缓存块在test\_mem.cc:37行定义，但未释放。

[![](https://blog.yanjingang.com/wp-content/uploads/2023/03/memory-leaks.png)](https://blog.yanjingang.com/wp-content/uploads/2023/03/memory-leaks.png)

# 三、其他

ASan也不是万能的，它在打开的情况下对运行性能有明显影响，在某些情况下也会出现误报，在实际的使用过程中可以作为一个测试流水线环节进行检测，以提高系统的稳定性。

#### 1、内存泄漏误报场景

-   结构体非 4 字节对齐：报错提示结构体 A 内存泄漏，A 内存的指针存放在结构体 B 中，A 内存指针在结构体 B 中的偏移量非 4 的整数倍，由于 ASan 扫描内存时是按照 4 字节偏移进行，从而扫描不到 A 内存指针导致误报。解决方法：对非4字节对齐的结构体进行整改。
-   信号栈内存：该内存是在信号处理函数执行时做栈内存用的，其指针会保存在内核中，所以在用户态的 ASan 扫描不到，产生误报；
-   内存指针偏移后保存：
-   存在ASan未监控的内存接口：
-   越界太离谱，越界访问的地址不在 buffer 的 redzone 内：
-   对于memcpy的dest和src是在同一个malloc的内存块中时，内存重叠的情况无法检测到。
-   ASan对于overflow的检测依赖于安全区，而安全区总归是有大小的。它可能是64bytes，128bytes或者其他什么值，但不管怎么样终归是有限的。如果某次踩踏跨过了安全区，踩踏到另一片可寻址的内存区域，ASan同样不会报错。这是ASan的另一种漏检。
-   ASan对于UseAfterFree的检测依赖于隔离区，而隔离时间是非永久的。也就意味着已经free的区域过一段时间后又会重新被分配给其他人。当它被重新分配给其他人后，原先的持有者再次访问此块区域将不会报错。因为这一块区域的shadow memory不再是0xfd。所以这算是ASan漏检的一种情况。

#### 2、在项目中的应用注意事项

-   项目的构建方案应当有编译选项，能随时启用/关闭ASan。
-   项目送测阶段可以打开ASan，以帮助暴露更多的低概率诡异问题。
-   请勿在生产版本中启用ASan，其会降低程序运行速度大概2-5倍，并会出现内存持续增长现象（占用的RedZone并不会自动释放，所以会出现内存溢出的假象，关闭ASan现象即会消失）。
-   实际开发测试过程中通过ASan扫出的常见问题有：多线程下临界资源未加保护导致同时出现读写访问，解决方案一般是对该资源恰当地加锁即可；内存越界，如申请了N字节的内存却向其内存地址拷贝大于N字节的数据，这种情况在没有开启ASan的情况下一般都很难发现。
-   一些显而易见的访问无效内存操作可能会被编译器优化而会漏报。

yan 3.13

参考：
https://github.com/google/sanitizers/wiki/AddressSanitizerAlgorithm

https://blog.csdn.net/u013171226/article/details/126876335

https://blog.csdn.net/yuanbinquan/article/details/106767635

https://www.jianshu.com/p/3a2df9b7c353

https://blog.yanjingang.com/?p=7225