---
title: unix programing
date: 2021-02-16 18:18:15
tags:
---
[CLion工程中只能有一个main函数 &&怎么同时编写多个main函数的C文件](https://blog.csdn.net/justinzwd/article/details/85206640)
```makefile
cmake_minimum_required(VERSION 3.13)
project(address C)

set (CMAKE_C_STANDARD 90)

add_executable(main01 Chapter-01/1.4.3.c)
```

[Unix环境编程中的apue.h和err_quit、err_sys问题](https://blog.csdn.net/u012814984/article/details/44751595)

## 1.4 文件和目录
- ls
```c
#include "../apue.3e/include/apue.h"
# include"error.c"

#define BUFFSIZE 4096

int main(int argc, char *argv[]) {
    int n;
    char buf[BUFFSIZE];

    while ((n = read(STDIN_FILENO, buf, BUFFSIZE)) > 0)
        if (write(STDOUT_FILENO, buf, n) != n)
            err_sys("write error");

    if (n < 0)
        err_sys("read error");
    exit(0);
}
```

## 1.5输入和输出
### 有缓冲IO
- 读输入输出
```c
#include "../apue.3e/include/apue.h"
# include"error.c"

#define BUFFSIZE 4096

int main(int argc, char *argv[]) {
    int n;
    char buf[BUFFSIZE];

    while ((n = read(STDIN_FILENO, buf, BUFFSIZE)) > 0)
        if (write(STDOUT_FILENO, buf, n) != n)
            err_sys("write error");

    if (n < 0)
        err_sys("read error");
    exit(0);
}
```

- 重定向: 控制台输入 -> 控制台输出
```bash
 ~/go/src/github.com/sonnary/apue/Chapter-01   master ●  gcc -o a 1.5.3.c
 ~/go/src/github.com/sonnary/apue/Chapter-01   master ●  ./a
1
1
2
2
^C
```

- 重定向: 控制台输入 -> 文件输出
```bash
 ✘  ~/go/src/github.com/sonnary/apue/Chapter-01   master ●  ./a > data
123
456
^C
 ✘  ~/go/src/github.com/sonnary/apue/Chapter-01   master ●  cat data
123
456
```

- 重定向: 文件输入 -> 文件输出
```bash
 ~/go/src/github.com/sonnary/apue/Chapter-01   master ●  cat > infile
indata
^C
 ✘  ~/go/src/github.com/sonnary/apue/Chapter-01   master ●  cat infile
indata
 ~/go/src/github.com/sonnary/apue/Chapter-01   master ●  ./a < infile > outfile
 ~/go/src/github.com/sonnary/apue/Chapter-01   master ●  cat outfile
indata
```
### 无缓冲IO
```c
#include "../apue.3e/include/apue.h"
# include "error.c"

int main(int argc, char *argv[]) {
    int c;
    while ((c = getc(stdin)) != EOF)
        if (putc(c, stdout) == EOF)
            err_sys("output error");

    if (ferror(stdin))
        err_sys("input error");
    exit(0);
}
```

## 1.6程序和进程
- 进程ID
```c
#include "../apue.3e/include/apue.h"

int main(int argc, char *argv[]) {
    printf("hello world from process ID %ld\n", (long)getpid());
    exit(0);
}
```
