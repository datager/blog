---
title: unix programing
date: 2021-02-16 18:18:15
tags: unix, c, linux, apue
---
[toc]



1. 弃用root用户
2. 重构老代码
3. 书要看, 习题代码要敲

内容

1. IO
   1. 3 5 14
2. 文件系统 
   1. 4 6 7
3. 进程  
   1. 信号10
   2. 多线程10, 11
4. 进程间通信
   1. 进程基础+多进程8
   2. 守护进程13
   3. 15, 16

# todo

C: https://www.bilibili.com/video/BV18p4y167Md

APUE笔记: https://www.cnblogs.com/0xcafebabe/tag/APUE/

man手册: vim下用shift+k可跳转到man手册

# 标准IO

I/O: 落盘是一切实现的基础, 分为2种

- stdio: 标准IO

- sysIO: 系统IO(文件IO)

优先用标准IO

标准是一组接口: 

- 移植性好, linux sysIO/win sysIO 都会实现该接口, 例如fopen是标准IO, linux是open的实现, win是openfile的实现
- 合并系统调用: 加速

```c
stdio: FILE类型贯穿始终
```

## fopen

```c
# include <stdio.h>
# include <stdlib.h>
# include <errno.h>
# include <string.h>

int main() {
    FILE *fp;
    fp = fopen("tmp", "r+");
    if (fp == NULL) {
//        fprintf(stderr, "open failed! errno: %d\n", errno);
        perror("fopen()");
        fprintf(stderr, "fopen(): %s\n", strerror(errno));
        exit(1);
    }
    puts("ok");

    exit(0);
}
```

- Fopen() 创建的FILE是在堆上(一般有fopen_fclose, malloc-free这种逆操作的都是分配在堆上的)

```c
FILE *open(const char *path, const char *mode){
  FILE *tmp = NULL;
  tm = malloc(sizeof(FILE));
  tmp->xx=xx;
  ...
    return tmp;
}

不可能放在栈是因为, if return &(函数内局部变量), 函数结束后变量就销毁了.
不可能放在静态区, 静态区只会有1份, 是因为该函数若被调用N次, 第2+次之后会覆盖第上一次打开的FILE;
```

## fclose和文件权限

一个进程最多打开几个文件

```c
# include <stdio.h>
# include <stdlib.h>
# include <errno.h>
# include <string.h>

int main() {
    int count = 0;
    FILE *fp = NULL;

    while (1) {
        fp = fopen("tmp", "r");
        if (fp == NULL) {
            perror("fopen()");
            break;
        }
        count++;
    }

    printf("count = %d\n", count);

    exit(0);
}
```

```bash
 ~/go/src/github.com/sonnary/apue/y   master ●✚  ./p5-fopen
fopen(): Too many open files
count = 253

用ulimit -a可看到上限256, 再加上stdin, stdout, stderr这3个, 总共253+3=256;
 ~/go/src/github.com/sonnary/apue/y   master ●✚  ulimit -a
-t: cpu time (seconds)              unlimited
-f: file size (blocks)              unlimited
-d: data seg size (kbytes)          unlimited
-s: stack size (kbytes)             8192
-c: core file size (blocks)         0
-v: address space (kbytes)          unlimited
-l: locked-in-memory size (kbytes)  unlimited
-u: processes                       709
-n: file descriptors                256

vim /etc/security/limits.conf 改soft nofile和hard nofile上限值(单进程级, reboot生效, ulimit -n当终端退出后则失效) && vim /etc/sysctl.conf 修改fs.file-max值(系统级, reboot生效)
https://www.mdeditor.tw/pl/pFyl
http://www.3mu.me/linux%E4%BF%AE%E6%94%B9open-files%E6%95%B0%E5%8F%8Aulimit%E5%92%8Cfile-max%E7%9A%84%E5%8C%BA%E5%88%AB/
```

- umask

  新创建的文件的chmod默认是0666 & ~umask()

  例如: umask为0022, 新创建的文件就是0666-0022=0644

```bash
root@k8s-wolf-minion-47:~# umask
0022
root@k8s-wolf-minion-47:~# touch 123
root@k8s-wolf-minion-47:~# ll | grep 123
-rw-r--r--  1 root root   27069 Feb 18 23:23 123
```

## fputc和pgetc

- copy命令

```c
# include <stdio.h>
# include <stdlib.h>

int main(int argc, char **argv) {
    FILE *fps = NULL;
    FILE *fpd = NULL;
    int ch;
    int put_ret_code;

    if (argc != 3) {
        fprintf(stderr, "Usage: %s <src_file_name>, <dst_file_name> need 2 arguments", argv[0]);
        exit(1);
    }

    fps = fopen(argv[1], "r");
    if (fps == NULL) {
        fclose(fps);
        perror("fopen");
        exit(1);
    }
    fpd = fopen(argv[2], "w");
    if (fpd == NULL) {
        perror("fopen");
        exit(1);
    }

    while (1) {
        ch = fgetc(fps);
        if (ch == EOF) {
            break;
        }
        put_ret_code = fputc(ch, fpd);
        if (put_ret_code == EOF) {
            break;
        }
    }

    fclose(fps);
    fclose(fpd);
}

```

```bash
 ~/go/src/github.com/sonnary/apue/y   master ●✚  ./p8-fgetc /etc/services /tmp/out
 ~/go/src/github.com/sonnary/apue/y   master ●✚  diff /etc/services /tmp/out
 是相同的, 模拟了linux的cp命令
 
 边界条件
 ✘  ~/go/src/github.com/sonnary/apue/y   master ●✚  make p8-fgetc && ./p8-fgetc
cc     p8-fgetc.c   -o p8-fgetc
Usage: ./p8-fgetc, need 2 arguments%
```

- 计算文件的字节数命令

```c
# include <stdio.h>
# include <stdlib.h>

int main(int argc, char **argv) {

    if (argc != 2) {
        fprintf(stderr, "Usage: need 1 arg\n");
        exit(1);
    }

    FILE *fp = fopen(argv[1], "r");
    if (fp == NULL) {
        perror("fopen");
        exit(1);
    }

    long count = 0;
    while (fgetc(fp) != EOF) {
        count++;
    }

    fclose(fp);

    fprintf(stdout, "count: %ld\n", count);
    exit(0);
}
```

```bash
drwxr-xr-x   9 sunyuchuan  staff   288  2 19 00:20 .
drwxr-xr-x  27 sunyuchuan  staff   864  2 19 00:15 ..
-rwxr-xr-x   1 sunyuchuan  staff  8568  2 18 22:56 p5-fopen
-rw-r--r--   1 sunyuchuan  staff   347  2 18 22:55 p5-fopen.c
-rwxr-xr-x   1 sunyuchuan  staff  8756  2 18 23:53 p8-fgetc
-rw-r--r--   1 sunyuchuan  staff   764  2 18 23:55 p8-fgetc.c
-rwxr-xr-x   1 sunyuchuan  staff  8760  2 19 00:20 p8-fgetccount
-rw-r--r--   1 sunyuchuan  staff   428  2 19 00:20 p8-fgetccount.c
-rw-r--r--   1 sunyuchuan  staff     0  2 18 22:42 tmp
~/go/src/github.com/sonnary/apue/y   master ●✚  ./p8-fgetccount p8-fgetc
Usage: need 1 arg
✘  ~/go/src/github.com/sonnary/apue/y   master ●✚  make p8-fgetccount
cc     p8-fgetccount.c   -o p8-fgetccount
~/go/src/github.com/sonnary/apue/y   master ●✚  ./p8-fgetccount p8-fgetc
count: 8756
~/go/src/github.com/sonnary/apue/y   master ●✚  ./p8-fgetccount p8-fgetccount.c
count: 428
```

## fputs fgets

```c
# include <stdio.h>
# include <stdlib.h>
# define BUFSIZE 100

int main(int argc, char **argv) {
    FILE *fps = NULL;
    FILE *fpd = NULL;
    char buf[BUFSIZE];

    if (argc != 3) {
        fprintf(stderr, "Usage: %s <src_file_name>, <dst_file_name> need 2 arguments", argv[0]);
        exit(1);
    }

    fps = fopen(argv[1], "r");
    if (fps == NULL) {
        fclose(fps);
        perror("fopen");
        exit(1);
    }
    fpd = fopen(argv[2], "w");
    if (fpd == NULL) {
        perror("fopen");
        exit(1);
    }

    while ( fgets(buf, BUFSIZE, fps) != NULL ) {
        fputs(buf, fpd);
    }

    fclose(fps);
    fclose(fpd);

    exit(0);
}

```

## fread fwrite

- 要注意 这一对儿函数用的时候要把块大小设置为1
  - 因为其行为为返回实际操作的块个数, 所以无法区分, 一般最佳实践为块大小设置为1
    - eg: 如果1个块1有10Bytes, 但拿到了0Bytes, 返回值为0
    - eg: 如果1个块1有10Bytes, 但拿到了5Bytes, 返回值为0
    - eg: 如果1个块1有10Bytes, 但拿到了10Bytes, 返回值为1

```c
# include <stdio.h>
# include <stdlib.h>
# define BUFSIZE 100

int main(int argc, char **argv) {
    FILE *fps = NULL;
    FILE *fpd = NULL;
    char buf[BUFSIZE];

    if (argc != 3) {
        fprintf(stderr, "Usage: %s <src_file_name>, <dst_file_name> need 2 arguments", argv[0]);
        exit(1);
    }

    fps = fopen(argv[1], "r");
    if (fps == NULL) {
        fclose(fps);
        perror("fopen");
        exit(1);
    }
    fpd = fopen(argv[2], "w");
    if (fpd == NULL) {
        perror("fopen");
        exit(1);
    }

  	int n = 0;
    while ( ( n = fread(buf, 1, BUFSIZE, fps) ) > 0 ) {
        fwrite(buf, 1, n, fpd);
    }

    fclose(fps);
    fclose(fpd);

    exit(0);
}
```

## atoi和sprintf

是逆操作

## fseeko和ftello

C99(为了移植)支持fseek和ftell, 因为返回的是long类型, 且只能用正数部分, 所以为0.5*2^32=2G大小, 不够文件大小用

方言(不考虑移植)推荐用fsseko和ftello, 其返回为off_t, 一般在编译时用`gcc a.c -o a -D_FILE_OFFSET_BITS=64`定义

## 文件位置函数

fseek, ftell, rewind 

```c
# include <stdio.h>
# include <stdlib.h>

int main(int argc, char **argv) {

    if (argc != 2) {
        fprintf(stderr, "Usage: need 1 arg\n");
        exit(1);
    }

    FILE *fp = fopen(argv[1], "r");
    if (fp == NULL) {
        perror("fopen");
        exit(1);
    }

    int seek_success = fseek(fp, 0, SEEK_END);
  	if (seek_success != 0) {
      perror("seek fail");
      exit(1);
    }
  
    printf("%ld\n", ftell(fp));

    fclose(fp);
    exit(0);
}
```

```bash
 ~/go/src/github.com/sonnary/apue/y   master ●✚  ls -l flen.c
-rw-r--r--  1 sunyuchuan  staff  467  2 19 10:05 flen.c
 ~/go/src/github.com/sonnary/apue/y   master ●✚  ./flen flen.c
467
```

- fseek可用于SEEK_END, 如迅雷下载, 把文件抻开成很大的大小(eg: 2GB), 然后分段用多线程下载
- rewind就是fseek(fp, 0, SEEK_SET)的封装, seek到文件头

## 缓冲区fflush

printf默认是行缓冲, 所以一般加`fprintf("xxx %d\n", xxx);`

缓冲区大多是好事, 用于合并系统调用

- 全缓冲: 满了强制刷 (用于非终端设备)
- 行缓冲: 换行or满了时刷新, (用于终端设备(如stdout等))
- 无缓冲: stderr

## getline

在makefile中写#define

```makefile
CFLAGS+=-D_FILE_OFFSET_BITS=64 -D_GNU_SOURCE
```

相当于在c文件开头写define, 会更易读

```c
#define _GNU_SOURCE
#include <stdio.h>

int main() {
  
}
```

- getline

```c
# include <stdio.h>
# include <stdlib.h>
# include <string.h>

int main(int argc, char **argv) {
  if (argc < 2) {
    fprintf(stderr, "Usage...\n");
    exit(1);
  }

  FILE *fp = fopen(argv[1], "r");
  if (fp == NULL) {
    perror("fopen");
    exit(1);
  }

  char *linebuf = NULL;
  size_t linesize = 0;
  while(1) {
    if (getline(&linebuf, &linesize, fp) < 0 ) {
      break;
    }
    printf("%lu\n", strlen(linebuf));
  }

  fclose(fp);
  exit(0);
}
```

```bash
 ~/go/src/github.com/sonnary/apue/y   master ●✚  ./getline makefile
45
```



# 系统IO

open, close, read, write, lseek

文件描述符是整数(数组下标), 优先返回可用的最小的fd

每个进程有个文件描述符数组(size为ulimit -n的长度), 数组下标为文件描述符, 值为FILE结构体指针, FILE结构体指向了文件inode

```bash
        thread 1
     +-------------+            +-------------+     +-------+
fd1  +val: &FILE1  +------------>             +-----> inode1|
     |             |            |   FILE1     |     +-------+
     +-------------+            +-------------+     +-------+
fd2  +val: &FILE2  |            |             +---->+ inode2|
     |             |            |   FILE2     |     +-------+
     +-------------+            +-------------+
     |             |
     |             |
     +-------------+
     |    ...      |
     |             |
     +-------------+


        thread 2
     +-------------+            +-------------+     +-------+
fd1  +val: &FILE1  +------------>             +-----> inode1|
     |             |            |   FILE1     |     +-------+
     +-------------+            +-------------+     +-------+
fd2  +val: &FILE2  |            |             +---->+ inode2|
     |             |            |   FILE2     |     +-------+
     +-------------+            +-------------+
     |             |
     |             |
     +-------------+
     |    ...      |
     |             |
     +-------------+

```

- man 2 open, C没有重载, 所以C是用变参函数实现的, (若传入的参数个数不符, C中的fprintf("%d %d", a,b,c)不会报错, 而C++重载是固定参数的则会报错)

```c
int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
```

## read write

```c
# include <stdio.h>
# include <stdlib.h>
# include <sys/types.h>
# include <sys/uio.h>
# include <unistd.h>
# include <fcntl.h>
# define BUFSIZE 1024

int main(int argc, char **argv) {
  int sfd, dfd;
  
  if (argc < 3) {
    fprintf(stderr, "Usage: 3 args...");
    exit(1);
  }
  
  sfd = open(argv[1], O_RDONLY);
  if (sfd < 0) {
    perror("open");
    exit(1);
  }
  
  dfd = open(argv[2], O_WRONLY|O_CREAT|O_TRUNC,0600);
  if (dfd < 0) {
    close(sfd);
    perror("open");
    exit(1);
  }
  
  char buf[BUFSIZE];
  int readed = 0, writted = 0;
  while(1) {
    readed = read(sfd, buf, BUFSIZE);
    if (readed < 0) {
      perror("read fail");
      break;
    }
    if (readed == 0) {
      fprintf(stdout, "all readed done");
      break;
    }
    
    int pos = 0;
    int towrite = readed;
    while(towrite > 0) {
      writted = write(dfd, buf + pos, towrite);
      if (writted < 0) {
        perror("write fail");
        exit(1);
      }
      
      towrite -= writted;
      pos += writted;
    }
  }
  
  close(sfd);
  close(dfd);
  
  exit(0);
}
```

标准IO: 引入缓冲区, 可以合并系统调用, 增大吞吐量

标准IO和文件IO不可混用, 因为有缓冲区的区别, 所以pos是不同的

```c
# include <stdio.h>
# include <stdlib.h>
# include <unistd.h>

int main() {
  putchar('a');
  write(1, "b", 1);
  
  putchar('a');
  write(1, "b", 1);
  
  putchar('a');
  write(1, "b", 1);
  
  exit(0);
}
```

```bash
root@k8s-wolf-minion-47:~# make write
cc     write.c   -o write
root@k8s-wolf-minion-47:~# ./write
bbbaaa

strace ./write 跟踪系统调用
write(1, "b", 1b)                        = 1
write(1, "b", 1b)                        = 1
write(1, "b", 1b)                        = 1
write(1, "aaa", 3aaa)                      = 3
exit_group(0)                           = ?
```

# 原子操作

dup() dup2()

- 输出从stdout重定向到文件

```c
# include <stdio.h>
# include <stdlib.h>
# include <sys/types.h>
# include <sys/stat.h>
# include <unistd.h>
# include <fcntl.h>
# define TMP_FILE "/tmp/out"
int main() {
  close(1);
  
  int fd = open(TMP_FILE, O_RDONLY|O_WRONLY|O_CREAT|O_TRUNC, 0600);
  if (fd < 0) {
    perror("open");
    exit(1);
  }
  
  puts("hello!");
  exit(0);
}
```

```bash
~/go/src/github.com/sonnary/apue/y   master ●✚  ./dup
~/go/src/github.com/sonnary/apue/y   master ●✚  cat /tmp/out
hello!
```

- dup: 复制一个最小的fd, 使原fd和新fd都指向同一个FILE, 该FILE指向inode(注意FILE内使用fd引用计数器的)

```c
# include <stdio.h>
# include <stdlib.h>
# include <sys/types.h>
# include <sys/stat.h>
# include <unistd.h>
# include <fcntl.h>
# define TMP_FILE "/tmp/out"
int main() {
  int fd = open(TMP_FILE, O_RDONLY|O_WRONLY|O_CREAT|O_TRUNC, 0600);
  if (fd < 0) {
    perror("open");
    exit(1);
  }
  
  close(1);
  dup(fd); 
  close(fd);
  
  puts("hello!");
  exit(0);
}
```

```bash
 ~/go/src/github.com/sonnary/apue/y   master ●✚  rm /tmp/out
 ~/go/src/github.com/sonnary/apue/y   master ●✚  ./dup1
 ~/go/src/github.com/sonnary/apue/y   master ●✚  cat /tmp/out
hello!
```

- dup2是原子操作

```c
# include <stdio.h>
# include <stdlib.h>
# include <sys/types.h>
# include <sys/stat.h>
# include <unistd.h>
# include <fcntl.h>
# define TMP_FILE "/tmp/out"
int main() {
  int fd = open(TMP_FILE, O_RDONLY|O_WRONLY|O_CREAT|O_TRUNC, 0600);
  if (fd < 0) {
    perror("open");
    exit(1);
  }
  
  dup2(fd, 1);
  
  if (fd != 1)
  	close(fd);
  
  puts("hello!");
  exit(0);
}
```

```bash
 ~/go/src/github.com/sonnary/apue/y   master ●✚  ./dup2
 ~/go/src/github.com/sonnary/apue/y   master ●✚  cat /tmp/out
hello!
```

## fcntl 管理文件描述符

Ioctl 设备相关的内容

/dev/fd/目录: 畜牧路, 显示的是当前进程的目录描述符信息



# 文件系统

- stat: 通过文件路径获取属性, 面对符号链接文件是获取的是所指向文件的属性
- fstat: 通过文件描述符获取属性
- lstat: 面对符号链接文件是获取的是符号链接文阿基内的属性

```c
# include <stdio.h>
# include <stdlib.h>
# include <sys/types.h>
# include <sys/stat.h>
# include <unistd.h>
# include <fcntl.h>

off_t flen(char *filename) {
  struct stat data;
  
  if (stat(filename, &data) != 0) {
    return 0;
  }
  return data.st_size;
}

int main(int argc, char **argv) {
  if (argc < 2) {
    fprintf(stderr, "Usage...\n");
    exit(1);
  }
  
  printf("%lld\n", flen(argv[1]));
}
```

```bash
 ~/go/src/github.com/sonnary/apue/y   master ●✚  ll flen2.c
-rw-r--r--  1 sunyuchuan  staff   406B  2 19 17:23 flen2.c
 ~/go/src/github.com/sonnary/apue/y   master ●✚  ./flen2 flen2.c
406
 ~/go/src/github.com/sonnary/apue/y   master ●✚  stat flen2.c
16777220 105969084 -rw-r--r-- 1 sunyuchuan staff 0 406 "Feb 19 17:23:40 2021" "Feb 19 17:23:34 2021" "Feb 19 17:23:34 2021" "Feb 19 17:23:34 2021" 4096 8 0 flen2.c
```

- 文件真正占用磁盘容量是Blocks * BytesPerBlock(一般为512Bytes), 而不是file的size
- linux的`cp`命令支持空洞文件拷贝, 如果src文件里read出的一段内容都是`\0`的话, 只会记录却并不会write`\0`到dst文件, 最终在fst文件做lseek偏移
- 注意设置为LL才不会溢出int上限

```c
# include <stdio.h>
# include <stdlib.h>
# include <sys/types.h>
# include <sys/stat.h>
# include <unistd.h>
# include <fcntl.h>

int main(int argc, char **argv) {
   if (argc < 2) {
     fprintf(stderr, "Usage: ...\n");
     exit(1);
   }
  
  int fd = open(argv[1], O_WRONLY|O_CREAT|O_TRUNC, 0600);
  if (fd < 0) {
    perror("open");
    exit(1);
  }
  lseek(fd, 5LL*1024LL*1024LL*1024LL-1LL, SEEK_SET);
  write(fd, "", 1);
  
  exit(0);
}
```

```bash
root@k8s-wolf-minion-47:~# vim big.c
root@k8s-wolf-minion-47:~# make big
cc     big.c   -o big
root@k8s-wolf-minion-47:~# ./big bigfile


root@k8s-wolf-minion-47:~# ll bigfile
-rw------- 1 root root 5368709120 Feb 19 18:42 bigfile


root@k8s-wolf-minion-47:~# stat bigfile
  File: 'bigfile'
  Size: 5368709120	Blocks: 8          IO Block: 4096   regular file
Device: fc00h/64512d	Inode: 76546720    Links: 1
Access: (0600/-rw-------)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2021-02-19 18:42:09.195734809 +0800
Modify: 2021-02-19 18:42:09.195734809 +0800
Change: 2021-02-19 18:42:09.195734809 +0800
 Birth: -
 
 root@k8s-wolf-minion-47:~# rm /tmp/bigfile.bk
root@k8s-wolf-minion-47:~# time bigfile /tmp/bigfile.bk
bigfile: command not found

real	0m0.160s
user	0m0.120s
sys	0m0.040s
root@k8s-wolf-minion-47:~#
```

- 5G文件占用的是4KB(因为8块, 每块512Bytes)
- st_mode是16位的位图, 表示文件的类型
- 粘住位, 一般是`/tmp`的最后一位是t
- FAT文件系统: 静态单链表, 轻量级, 缺陷: 上限有限制, 单链表只能单向
- UFS: 

## 文件

硬链接: 就是目录项, 缺点: 不能给分区建立, 不能给目录建立

软链接: 即符号链接, 其值为`指向的文件名`, 优点: 可以给分区建立, 可以给目录建立

link unlink remove rename

utime: 最后读/更改的时间

## 目录

```c
opendir()
closedir()
readdir();
rewinddir();
```

##argc argv

```c
    ./main abc 123 456 789

          argv

         +----------+
argv[0]  |          |         +-------+ "./main"
         |          +-------> +-------+
         +----------+
argv[1]  |          |         +-------+ "abc"
         |          +-------> +-------+
         +----------+
argv[2]  |          |         +-------+ "123"
         |          +-------> +-------+
         +----------+
argv[3]  |          |                   "456"
         |          |
         +----------+
argv[4]  |          |                   "789"
         |          |
         +----------+
```

## glob

解析模式/通配符

 ```c
# include <stdio.h>
# include <stdlib.h>
# include <sys/types.h>
# include <sys/stat.h>
# include <unistd.h>
# include <fcntl.h>
# include <glob.h>
# define PAT "/etc/a*.conf"

int main(int argc, char *argv[]) {
  if (argc != 1) {
    fprintf(stderr, "Usage: 1 argvs");
    exit(1);
  }
  
  glob_t globres;
  int i, err;
  
  err = glob(PAT, 0, NULL, &globres);
  if (err) {
    printf("Error code = %d\n", err);
    exit(1);
  }
  
  for (i = 0; i < globres.gl_pathc; i++) {
    puts(globres.gl_pathv[i]);
  }
  
  globfree(&globres);
  
  exit(0);
}
 ```

## readdir

```c
# include <stdio.h>
# include <stdlib.h>
# include <sys/types.h>
# include <sys/stat.h>
# include <unistd.h>
# include <fcntl.h>
# include <glob.h>
# include <dirent.h>

# define PAT "/etc"

int main(int argc, char *argv[]) {
  DIR *dp;
  struct dirent *cur;
  
  dp = opendir(PAT);
  if (dp == NULL) {
    perror("opendir()");
    exit(1);
  }
  
  while((cur = readdir(dp)) != NULL)
    puts(cur->d_name);
  
  closedir(dp);
  
  exit(0);
}
```

## du

占用的磁盘空间大小, KBytes为单位

```bash
root@k8s-wolf-minion-47:~# stat read.c
  File: 'read.c'
  Size: 1068      	Blocks: 8          IO Block: 4096   regular file
Device: fc00h/64512d	Inode: 76546716    Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2021-02-19 13:24:30.344781444 +0800
Modify: 2021-02-19 13:24:28.736767302 +0800
Change: 2021-02-19 13:24:28.736767302 +0800
 Birth: -
 
root@k8s-wolf-minion-47:~# du read.c
4	read.c

8个block * 512Bytes/block = 4KBytes
```

自己实现du如下

```c
# include <stdio.h>
# include <stdlib.h>
# include <sys/types.h>
# include <sys/stat.h>
# include <unistd.h>
# include <fcntl.h>
# include <glob.h>
# include <dirent.h>
# include <string.h>
# define PATHSIZE 1024

static int path_noloop(const char *path) {
	char *pos = strrchr(path, '/');
  if ( pos == NULL )
    exit(1);
  if ((strcmp(pos+1, ".") == 0) || (strcmp(pos+1, "..") == 0))
    return 0;
  return 1;
}

static int mydu(char *path) {
  struct stat statres;
  glob_t globres;
  char nextpath[PATHSIZE];
  int sum = 0;
  
  if (lstat(path, &statres) < 0) {
    perror("lstat");
    exit(1);
  }
  
  if (!S_ISDIR(statres.st_mode))
    return statres.st_blocks;
  
  strncpy(nextpath, path, PATHSIZE);
  strncat(nextpath, "/*", PATHSIZE);
  glob(nextpath, 0, NULL, &globres);
  
  strncpy(nextpath, path, PATHSIZE);
  strncat(nextpath, "/.*", PATHSIZE);
  glob(nextpath, GLOB_APPEND, NULL, &globres);
  
  int i = 0;
  for (i = 0; i < globres.gl_pathc; i++) {
    if (!path_noloop(globres.gl_pathv[i]))
      continue;
    sum += mydu(globres.gl_pathv[i]);
  }
  sum += statres.st_blocks;
  
  globfree(&globres);
  return sum;
}

int main(int argc, char **argv) {
  if (argc != 2) {
    fprintf(stderr, "Usage: 1 argv");
    exit(1);
  }
  
  printf("%d", mydu(argv[1])/2); // 因为512Bytes为一个块, 所以占用的Bytes数为块数*512/1024=块数/2
  
  exit(1);
}
```

```bash
root@k8s-wolf-minion-47:./du /etc/
6860	/etc/
root@k8s-wolf-minion-47: ./mydu /etc/
6860%


root@k8s-wolf-minion-47: du glob.c
8	glob.c
root@k8s-wolf-minion-47: ./mydu glob.c
8%
```

- Fdisk -l可以看到磁盘块大小

## 系统数据文件和信息

/etc/passwd

getpwuid getpwunam



/etc/passwd

Getgrgid getgrnam



/etc/shadow

## 单进程环境

- main函数

int main(int argc, char *argc[])

### 进程的终止

正常终止:

- c共main函数返回

```c
int main() {
  return 0;
}
return 0是给当前进程的父进程看的

./main
echo $?打印上次执行结果 若成功则为0 
```

- exit()

```c
int main() {
  exit(0);
}

可返回-128~127
```

​	atexit钩子函数(压入栈), 即go的defer, c++的析构函数

```c
# include<stdio.h>
# include<stdlib.h>
static void f1(void) {
	printf("f1 working\n");
}
static void f2(void) {
	printf("f2 working\n");
}
static void f3(void) {
	printf("f3 working\n");
}

int main(int argc, char **argv) {
	printf("begin\n");
  atexit(f1);
  atexit(f2);
  atexit(f3);
  printf("end\n");
  exit(0);
}
```

```bash
 ~/go/src/github.com/sonnary/apue/y   master ●✚  ./atexit
begin
end
f3 working
f2 working
f1 working
```

- `_exit()` 或`_Exit()`

  是系统调用, 不执行钩子函数和进程资源清理

- 最后一个现场从其启动例程返回

- 最后一个进程调用pthread_exit

异常终止:

- abort()
- 接到一个信号并终止
- 最后一个线程对其取消请求作出响应

### 命令行参数分析

```c
getopt()
getopt_long()
```

- 可以传参

```c
# include <stdio.h>
# include <stdlib.h>
# include <time.h>
# define TIMESTRSIZE 1024

int main() {
  struct tm *tm;
  char timestr[TIMESTRSIZE];
  
 	time_t stamp = time(NULL);
  tm = localtime(&stamp);
  strftime(timestr, TIMESTRSIZE, "Now:%Y-%m-%d", tm);
  puts(timestr);
  
  exit(0);
}
```

```c
# include <stdio.h>
# include <stdlib.h>
# include <time.h>
# include <unistd.h>
# include <string.h>
# define TIMESTRSIZE 1024
# define FMTSTRSIZE 1024

/**
* -y: year
* -m: month
* -d: day
* -H: hour
* -M: minute
* -s: second
*/
int main(int argc, char **argv) {
  struct tm *tm;
  char timestr[TIMESTRSIZE];
  int c;
  char fmtstr[FMTSTRSIZE];
  fmtstr[0] = '\0';
  
 	time_t stamp = time(NULL);
  tm = localtime(&stamp);
  
  while(1) {
    c = getopt(argc, argv, "HMSymd");
    if (c < 0) {
      break;
    }
    
    switch(c) {
      case 'H':
        strncat(fmtstr, "%H ", FMTSTRSIZE);
        break;
      case 'M':
        strncat(fmtstr, "%M ", FMTSTRSIZE);
        break;
      case 'S':
        strncat(fmtstr, "%S ", FMTSTRSIZE);
        break;
      case 'y':
        strncat(fmtstr, "%y ", FMTSTRSIZE);
        break;
      case 'm':
        strncat(fmtstr, "%m ", FMTSTRSIZE);
        break;
      case 'd':
        strncat(fmtstr, "%d ", FMTSTRSIZE);
        break;
      default:
        break;
    }
  }
  
  strftime(timestr, TIMESTRSIZE, fmtstr, tm);
  puts(timestr);
  
  exit(0);
}
```

```bash
root@k8s-wolf-minion-47:~# date
Sun Feb 21 22:36:13 CST 2021
root@k8s-wolf-minion-47:~# ./getopt -M -S -m -d
36 13 02 21
```

- 下面是带参数的选项, 就是加冒号

```c
# include <stdio.h>
# include <stdlib.h>
# include <time.h>
# include <unistd.h>
# include <string.h>
# define TIMESTRSIZE 1024
# define FMTSTRSIZE 1024

/**
* -y: year
* -m: month
* -d: day
* -H: hour
* -M: minute
* -s: second
*/
int main(int argc, char **argv) {
  struct tm *tm;
  char timestr[TIMESTRSIZE];
  int c;
  char fmtstr[FMTSTRSIZE];
  fmtstr[0] = '\0';
  
 	time_t stamp = time(NULL);
  tm = localtime(&stamp);
  
  while(1) {
    c = getopt(argc, argv, "H:MSy:md");
    if (c < 0) {
      break;
    }
    
    switch(c) {
      case 'H':
        if (strcmp(optarg, "12") == 0) 
          strncat(fmtstr, "%I(%P) ", FMTSTRSIZE);
        else if (strcmp(optarg, "24") == 0) {
          strncat(fmtstr, "%H ", FMTSTRSIZE);
        } else {
          fprintf(stderr, "Invalid arguiments");
        }
        strncat(fmtstr, "%H ", FMTSTRSIZE);
        break;
      case 'M':
        strncat(fmtstr, "%M ", FMTSTRSIZE);
        break;
      case 'S':
        strncat(fmtstr, "%S ", FMTSTRSIZE);
        break;
      case 'y':
        if (strcmp(optarg, "2") == 0) 
          strncat(fmtstr, "%y ", FMTSTRSIZE);
        else if (strcmp(optarg, "4") == 0)
          strncat(fmtstr, "%Y ", FMTSTRSIZE);
        else 
          fprintf(stderr, "Invalid arguments of -y");
        strncat(fmtstr, "%y ", FMTSTRSIZE);
        break;
      case 'm':
        strncat(fmtstr, "%m ", FMTSTRSIZE);
        break;
      case 'd':
        strncat(fmtstr, "%d ", FMTSTRSIZE);
        break;
      default:
        break;
    }
  }
  
  strftime(timestr, TIMESTRSIZE, fmtstr, tm);
  puts(timestr);
  
  exit(0);
}
```

```bash
root@k8s-wolf-minion-47:~# date
Sun Feb 21 22:40:31 CST 2021
root@k8s-wolf-minion-47:~# ./getopt -M -S -m -d -y 4 -H 12
40 35 02 21 21 10(pm)22

root@k8s-wolf-minion-47:~# date
Sun Feb 21 22:41:19 CST 2021
root@k8s-wolf-minion-47:~# ./getopt -H 12
10(pm)22

root@k8s-wolf-minion-47:~# date
Sun Feb 21 22:45:39 CST 2021
root@k8s-wolf-minion-47:~# ./getopt -y 4 -m -d
2021 21 02 21
root@k8s-wolf-minion-47:~# ./getopt -y 2 -m -d
21 21 02 21
```

- 下面是非选项传参, 如果传参有文件&&文件可打开则写入文件, 否则写入stdout
  - 原理是非选项传参会被`getopt()函数`识别为`1`

```c
# include <stdio.h>
# include <stdlib.h>
# include <time.h>
# include <unistd.h>
# include <string.h>
# define TIMESTRSIZE 1024
# define FMTSTRSIZE 1024

/**
* -y: year
* -m: month
* -d: day
* -H: hour
* -M: minute
* -s: second
*/
int main(int argc, char **argv) {
  FILE *fp = stdout;
  struct tm *tm;
  char timestr[TIMESTRSIZE];
  int c;
  char fmtstr[FMTSTRSIZE];
  fmtstr[0] = '\0';
  
 	time_t stamp = time(NULL);
  tm = localtime(&stamp);
  
  while(1) {
    c = getopt(argc, argv, "-H:MSy:md");
    if (c < 0) {
      break;
    }
    
    switch(c) {
      case 1:
        if (fp == stdout) {
          fp = fopen(argv[optind-1], "w");
          if (fp == NULL) {
            perror("fopen()");
            fp = stdout;
          } 
        }
        break;
      case 'H':
        if (strcmp(optarg, "12") == 0) 
          strncat(fmtstr, "%I(%P) ", FMTSTRSIZE);
        else if (strcmp(optarg, "24") == 0) {
          strncat(fmtstr, "%H ", FMTSTRSIZE);
        } else {
          fprintf(stderr, "Invalid arguiments");
        }
        strncat(fmtstr, "%H ", FMTSTRSIZE);
        break;
      case 'M':
        strncat(fmtstr, "%M ", FMTSTRSIZE);
        break;
      case 'S':
        strncat(fmtstr, "%S ", FMTSTRSIZE);
        break;
      case 'y':
        if (strcmp(optarg, "2") == 0) 
          strncat(fmtstr, "%y ", FMTSTRSIZE);
        else if (strcmp(optarg, "4") == 0)
          strncat(fmtstr, "%Y ", FMTSTRSIZE);
        else 
          fprintf(stderr, "Invalid arguments of -y");
        strncat(fmtstr, "%y ", FMTSTRSIZE);
        break;
      case 'm':
        strncat(fmtstr, "%m ", FMTSTRSIZE);
        break;
      case 'd':
        strncat(fmtstr, "%d ", FMTSTRSIZE);
        break;
      default:
        break;
    }
  }
  
  strncat(fmtstr, "\n", FMTSTRSIZE);
  strftime(timestr, TIMESTRSIZE, fmtstr, tm);
  fputs(timestr, fp);
  
  if (fp != stdout)
  	fclose(fp);
  exit(0);
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ ./getopt -y 4 /tmp/o0 -m /tmp/o1 -d /tmp/o2
ubuntu@k8s-wolf-minion-47:~$ cat /tmp/o0
2021 21 02 21
ubuntu@k8s-wolf-minion-47:~$ cat /tmp/o1
cat: /tmp/o1: No such file or directory
ubuntu@k8s-wolf-minion-47:~$ cat /tmp/o2
cat: /tmp/o2: No such file or directory
```

### 环境变量

都存在environ的全局变量中(类似errno全局变量), 是一个字符串数组

```c
# include <stdio.h>
# include <stdlib.h>

extern char **environ;
int main() {
  int i = 0;
  for (i = 0; environ[i] != NULL; i++)
    puts(environ[i]);
	exit(0);
}
```

- 运行结果和`export`相同, 会打印所有环境变量

```bash
 ~/go/src/github.com/sonnary/apue/y   master ●✚  ./myenv
TERM_SESSION_ID=w0t1p0:AF2FA44C-F867-49A5-9309-1D3BCDDC8926
SSH_AUTH_SOCK=/private/tmp/com.apple.launchd.WpTXLeu8Cx/Listeners
LC_TERMINAL_VERSION=3.4.4

 ~/go/src/github.com/sonnary/apue/y   master ●✚  export
TERM_SESSION_ID=w0t1p0:AF2FA44C-F867-49A5-9309-1D3BCDDC8926
SSH_AUTH_SOCK=/private/tmp/com.apple.launchd.WpTXLeu8Cx/Listeners
LC_TERMINAL_VERSION=3.4.4
```

- 一些函数

```c
getenv();
setenv(); // 申请在堆上, 可指定是否覆盖
unsetenv(); // 释放老堆的值, 在堆上申请新值
putenv(); // 不好用, 因为参数没有const修饰
```

```c
# include <stdio.h>
# include <stdlib.h>

int main() {
  puts(getenv("PATH"));
  exit(0);
}
```

```bash
 ~/go/src/github.com/sonnary/apue/y   master ●✚  ./getenv
/Users/sunyuchuan/bin:/usr/local/bin:/usr/local/bin:/usr/bin:/bin:
```

- C程序的存储空间布局

32位 4G虚拟内存空间, 3G用户态, 1G内核态. 3G=0x08048000~0xC0000000

```bash
   4G   +-----------------+
kernel space              |
        |                 |
   3G   +---------------------------+ 0XC000000
        |                 |
        |                 |
        |                 |
        |                 |
        |                 |
        |                 |
 user space               |
        |                 |
        |                 |
        |                 |
        |                 |
        |                 |
        |                 |
        +---------------------------+0X08408000
        |                 |
        |                 |
  0     +-----------------+

```

```c
# include <stdio.h>
# include <stdlib.h>

int main() {
  puts(getenv("PATH"));
  getchar();
  exit(0);
}
```

```bash
root@k8s-wolf-minion-47:~# ps aux | grep getenv
root      93166  0.0  0.0   4352   672 pts/0    S+   16:21   0:00 ./getenv

root@k8s-wolf-minion-47:~# pmap 93166
93166:   ./getenv
0000000000400000      4K r-x-- getenv
0000000000600000      4K r---- getenv
0000000000601000      4K rw--- getenv
0000000001145000    132K rw---   [ anon ]
00007f5ce8073000   1792K r-x-- libc-2.23.so
00007f5ce8233000   2048K ----- libc-2.23.so
00007f5ce8433000     16K r---- libc-2.23.so
00007f5ce8437000      8K rw--- libc-2.23.so
00007f5ce8439000     16K rw---   [ anon ]
00007f5ce843d000    152K r-x-- ld-2.23.so
00007f5ce8658000     12K rw---   [ anon ]
00007f5ce8662000      4K r---- ld-2.23.so
00007f5ce8663000      4K rw--- ld-2.23.so
00007f5ce8664000      4K rw---   [ anon ]
00007ffc9817b000    132K rw---   [ stack ]
00007ffc981f2000     12K r----   [ anon ]
00007ffc981f5000      8K r-x--   [ anon ]
ffffffffff600000      4K r-x--   [ anon ]
 total             4356K
```

- pmap(1)命令

### 库

- 动态库

- 静态库

- 手工装载库

- 加载动态库的流程: dlopen, dlsym, dlerror, dlclose

  ```c
  #include <stdio.h>
  #include <stdlib.h>
  #include <dlfcn.h>
  #include <gnu/lib-names.h>  /* Defines LIBM_SO (which will be a
                                          string such as "libm.so.6") */
  int
    main(void)
  {
    void *handle;
    double (*cosine)(double);
    char *error;
  
    handle = dlopen(LIBM_SO, RTLD_LAZY);
    if (!handle) {
      fprintf(stderr, "%s\n", dlerror());
      exit(EXIT_FAILURE);
    }
  
    dlerror();    /* Clear any existing error */
  
    cosine = (double (*)(double)) dlsym(handle, "cos");
  
    /* According to the ISO C standard, casting between function
                  pointers and 'void *', as done above, produces undefined results.
                  POSIX.1-2003 and POSIX.1-2008 accepted this state of affairs and
                  proposed the following workaround:
  
                      *(void **) (&cosine) = dlsym(handle, "cos");
  
                  This (clumsy) cast conforms with the ISO C standard and will
                  avoid any compiler warnings.
  
                  The 2013 Technical Corrigendum to POSIX.1-2008 (a.k.a.
                  POSIX.1-2013) improved matters by requiring that conforming
                  implementations support casting 'void *' to a function pointer.
                  Nevertheless, some compilers (e.g., gcc with the '-pedantic'
                  option) may complain about the cast used in this program. */
  
    error = dlerror();
    if (error != NULL) {
      fprintf(stderr, "%s\n", error);
      exit(EXIT_FAILURE);
    }
  
    printf("%f\n", (*cosine)(2.0));
    dlclose(handle);
    exit(EXIT_SUCCESS);
  }
  ```

  

  ### 栈 静态区管理

### 函数跳转

```c
setjmp()
longjmp()
```

- 比goto功能高级: 可以跨函数

```c
# include <stdio.h>
# include <stdlib.h>
static void d(void) {
  printf("%s(): Begin.\n", __FUNCTION__);
  printf("%s(): End.\n", __FUNCTION__);
}

static void c(void) {
  printf("%s(): Begin.\n", __FUNCTION__);
  printf("%s(): Call d().\n", __FUNCTION__);
  d();
  printf("%s():d() returned.\n", __FUNCTION__);
  printf("%s(): End.\n", __FUNCTION__);
}

static void b(void) {
  printf("%s(): Begin.\n", __FUNCTION__);
  printf("%s(): Call c().\n", __FUNCTION__);
  c();
  printf("%s():c() returned.\n", __FUNCTION__);
  printf("%s(): End.\n", __FUNCTION__);
}

static void a(void) {
  printf("%s(): Begin.\n", __FUNCTION__);
  printf("%s(): Call b().\n", __FUNCTION__);
  b();
  printf("%s():b() returned.\n", __FUNCTION__);
  printf("%s(): End.\n", __FUNCTION__);
}

void main() {
  printf("%s(): Begin.\n", __FUNCTION__);
  printf("%s(): Call a().\n", __FUNCTION__);
  a();
  printf("%s():a() returned.\n", __FUNCTION__);
  printf("%s(): End.\n", __FUNCTION__);
  
  exit(0);
}
```

Main -> a() -> b() -> c() -> d()

```bash
 +------------+
 |            |  d
+-------------+
 |            |
 |            |  c
 +------------+
 |            |
 |            |  b
 |            |
 +------------+
 |            |
 |            |   a
 |            |
 +------------+
 |            |
 |            |   main
 |            |
 +------------+
```

```c
root@k8s-wolf-minion-47:~# ./jump
main(): Begin.
main(): Call a().
a(): Begin.
a(): Call b().
b(): Begin.
b(): Call c().
c(): Begin.
c(): Call d().
d(): Begin.
d(): End.
c():d() returned.
c(): End.
b():c() returned.
b(): End.
a():b() returned.
a(): End.
main():a() returned.
main(): End.
```

- 函数a做setjump, 函数d做longjump跳到a处, 改写为如下
  - 能跨函数跳, 交出函数执行权+执行环境
  - setjmp()执行一次, 返回两次

```c
# include <stdio.h>
# include <stdlib.h>
# include <setjmp.h>

static jmp_buf save;

static void d(void) {
  printf("%s(): Begin.\n", __FUNCTION__);
  printf("%s(): jump now\n", __FUNCTION__);
  longjmp(save, 6);
  printf("%s(): End.\n", __FUNCTION__);
}

static void c(void) {
  printf("%s(): Begin.\n", __FUNCTION__);
  printf("%s(): Call d().\n", __FUNCTION__);
  d();
  printf("%s():d() returned.\n", __FUNCTION__);
  printf("%s(): End.\n", __FUNCTION__);
}

static void b(void) {
  printf("%s(): Begin.\n", __FUNCTION__);
  printf("%s(): Call c().\n", __FUNCTION__);
  c();
  printf("%s():c() returned.\n", __FUNCTION__);
  printf("%s(): End.\n", __FUNCTION__);
}

static void a(void) {
  printf("%s(): Begin.\n", __FUNCTION__);
	int ret = setjmp(save);
  if (ret == 0) {
    printf("%s(): Call b().\n", __FUNCTION__);
    b();
    printf("%s():b() returned.\n", __FUNCTION__);
  } else {
    printf("%s(): jumped back here with code %d\n", __FUNCTION__, ret);
  }
  printf("%s(): End.\n", __FUNCTION__);
}

void main() {
  printf("%s(): Begin.\n", __FUNCTION__);
  printf("%s(): Call a().\n", __FUNCTION__);
  a();
  printf("%s():a() returned.\n", __FUNCTION__);
  printf("%s(): End.\n", __FUNCTION__);
  
  exit(0);
}
```

```bash
root@k8s-wolf-minion-47:~# ./setjmp
main(): Begin.
main(): Call a().
a(): Begin.
a(): Call b().
b(): Begin.
b(): Call c().
c(): Begin.
c(): Call d().
d(): Begin.
d(): jump now
a(): jumped back here with code 6
a(): End.
main():a() returned.
main(): End.
```

### 资源获取与控制(如ulimit)

```c
gettlimit();
setrlimit();
```

组合就是ulimit命令

# 进程

## 进程标识符pid

类型pid_t 有符号的16位整数, 最多3w个进程

`ps axf`

`ps axm`

`ps ax -L`看轻量进程

进程号是顺次向下使用

`getpid()`

`getppid()`

## 进程的产生

### fork()

- 在2个进程返回
  - 父进程 返回 子进程pid(失败时父进程 返回 -1 && 设置errno)
  - 子进程 返回 0

- 复制父进程(一模一样, 包括已执行到的位置)

- 除了以下不同: 
  - pid不同, ppid也不同, `fork()`返回值不同
  - 未决信号和文件锁 不继承
  - 资源利用量清0

```bash
      parent process          child process

  +----------+                +----------+
  |          |                |          |
  |          |                |          |
  |          |                |          |
+-------------+已 执 行 处      +--------------+  已 执 行 处
  |          |                |          |
  |          |                |          |
  |          |    fork        |          |
  |          |  +--------->   |          |
  |          |                |          |
  |          |                |          |
  |          |                |          |
  +----------+                +----------+

```

```c
# include <stdio.h>
# include <stdlib.h>
# include <unistd.h>
int main() {
  pid_t pid;
  printf("[%d] Begin!\n", getpid());
  
  pid = fork();
  if (pid < 0) {
    perror("fork()");
    exit(1);
  } else if (pid == 0) {
    printf("[%d]: child is working\n", getpid());
  } else if (pid > 0) {
    printf("[%d]: parent is working\n", getpid());
  }
  
  printf("[%d] End!\n", getpid());
  
  exit(0);
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ ./fork
[160859] Begin!
[160859]: parent is working
[160859] End!
[160860]: child is working
[160860] End!

 或 
 
 ubuntu@k8s-wolf-minion-47:~$ ./fork
[160859] Begin!
[160859]: child is working
[160859] End!
[160860]: parent is working
[160860] End!
```

- `调度器的调度策略` 决定 哪个进程先运行(可能child或parent被调度的顺序是不确定的)
- 下面的代码加了getchar(), 使程序停下来, 用ps观察

```c
# include <stdio.h>
# include <stdlib.h>
# include <unistd.h>
int main() {
  pid_t pid;
  printf("[%d] Begin!\n", getpid());
  
  pid = fork();
  if (pid < 0) {
    perror("fork()");
    exit(1);
  } else if (pid == 0) {
    printf("[%d]: child is working\n", getpid());
  } else if (pid > 0) {
    printf("[%d]: parent is working\n", getpid());
  }
  
  printf("[%d] End!\n", getpid());
  
  getchar();
  exit(0);
}
```

```bash
ps axf 找进程号

160027 ?        Ss     0:00  \_ sshd: ubuntu [priv]
160039 ?        S      0:00  |   \_ sshd: ubuntu@pts/0
160040 pts/0    Ss     0:00  |       \_ -bash
161677 pts/0    S+     0:00  |           \_ ./fork
161678 pts/0    S+     0:00  |               \_ ./fork
```

#### 一定要先fflush()再fork()

- 去掉getchar, 重定向到文件中
  - Begin被打印2次
  - 解决办法, 在fork前用`fflush()`刷新所有成功打开的流

```c
# include <stdio.h>
# include <stdlib.h>
# include <unistd.h>
int main() {
  pid_t pid;
  printf("[%d] Begin!\n", getpid());
  
  pid = fork();
  if (pid < 0) {
    perror("fork()");
    exit(1);
  } else if (pid == 0) {
    printf("[%d]: child is working\n", getpid());
  } else if (pid > 0) {
    printf("[%d]: parent is working\n", getpid());
  }
  
  printf("[%d] End!\n", getpid());
  
  exit(0);
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ ./fork > /tmp/out
ubuntu@k8s-wolf-minion-47:~$ cat /tmp/out
[162452] Begin!
[162452]: parent is working
[162452] End!
[162452] Begin!
[162453]: child is working
[162453] End!
```

- 增加`fflush()`改进为如下

```c
# include <stdio.h>
# include <stdlib.h>
# include <unistd.h>
int main() {
  pid_t pid;
  printf("[%d] Begin!\n", getpid());
  
  fflush(NULL);  
  pid = fork();
  if (pid < 0) {
    perror("fork()");
    exit(1);
  } else if (pid == 0) {
    printf("[%d]: child is working\n", getpid());
  } else if (pid > 0) {
    printf("[%d]: parent is working\n", getpid());
  }
  
  printf("[%d] End!\n", getpid());
  
  exit(0);
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ ./fork > /tmp/out
ubuntu@k8s-wolf-minion-47:~$ cat /tmp/out
[163515] Begin!
[163515]: parent is working
[163515] End!
[163516]: child is working
[163516] End!
```

在begin放到缓冲区, 然后fork后, 父子进程的缓冲区各有一句`"begin"`, 所以为了避免就需要fflush()

#### 计算质数

- 单机版

```c
# include <stdio.h>
# include <stdlib.h>
# include <unistd.h>
# define LEFT 30000000
# define RIGHT 30000200
int main() {
  int i, j, mark;
  for (i = LEFT; i <= RIGHT; i++) {
    mark = 1;
    for (j = 2; j < i/2; j++) {
      if (i % j == 0) {
        mark = 0;
        break;
      }
    }
    if (mark) {
      printf("%d is a primer\n", i);
    }
  }

    exit(0);
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ time ./primer0 > /dev/null

real	0m1.187s
user	0m1.188s
sys	0m0.000s
```

- 201个子进程
  - 父进程fork201次
  - 子进程负责执行, 但子进程不应继续fork孙进程(子进程需要有退出方法)

```c
# include <stdio.h>
# include <stdlib.h>
# include <unistd.h>
# define LEFT 30000000
# define RIGHT 30000200
int main() {
  pid_t pid;
  int i, j, mark;
  for (i = LEFT; i <= RIGHT; i++) {

    pid = fork();
    if (pid < 0) {
			perror("fork");
      exit(1);
    }
    
    if (pid == 0) {
      mark = 1;
      for (j = 2; j < i/2; j++) {
        if (i % j == 0) {
          mark = 0;
          break;
        }
      }
      if (mark) {
        printf("%d is a primer\n", i);
      }   
      exit(0); // 子进程的退出方法, 防止子进程再fork孙进程
    }

  }

  exit(0);
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ ./primer1 | wc -l
18
ubuntu@k8s-wolf-minion-47:~$ time ./primer1 > /dev/null

real	0m0.020s
user	0m0.000s
sys	0m0.020s
// 性能取决于有几核, 若有x个核, 则y个进程排队等待被x个核调度, 则性能会提升x倍, 速度会变为y/x.
```

- 下面先让父进程结束

```c
# include <stdio.h>
# include <stdlib.h>
# include <unistd.h>
# define LEFT 30000000
# define RIGHT 30000200
int main() {
  pid_t pid;
  int i, j, mark;
  for (i = LEFT; i <= RIGHT; i++) {

    pid = fork();
    if (pid < 0) {
			perror("fork");
      exit(1);
    }
    
    if (pid == 0) {
      mark = 1;
      for (j = 2; j < i/2; j++) {
        if (i % j == 0) {
          mark = 0;
          break;
        }
      }
      if (mark) {
        printf("%d is a primer\n", i);
      }   
      sleep(1000);
      exit(0); // 子进程的退出方法, 防止子进程再fork孙进程
    }

  }

  exit(0);
}
```

```bash
 42259 pts/0    S+     0:00 ./primer1
 42260 pts/0    S+     0:00 ./primer1
 42261 pts/0    S+     0:00 ./primer1
 42262 pts/0    S+     0:00 ./primer1
 42263 pts/0    S+     0:00 ./primer1
 42264 pts/0    S+     0:00 ./primer1
 42265 pts/0    S+     0:00 ./primer1
 42266 pts/0    S+     0:00 ./primer1
 42267 pts/0    S+     0:00 ./primer1
 42268 pts/0    S+     0:00 ./primer1
 42269 pts/0    S+     0:00 ./primer1
 42270 pts/0    S+     0:00 ./primer1
 42271 pts/0    S+     0:00 ./primer1
 42272 pts/0    S+     0:00 ./primer1
 42273 pts/0    S+     0:00 ./primer1
 42274 pts/0    S+     0:00 ./primer1
 42275 pts/0    S+     0:00 ./primer1
 42276 pts/0    S+     0:00 ./primer1
 42277 pts/0    S+     0:00 ./primer1
 42278 pts/0    S+     0:00 ./primer1
 42279 pts/0    S+     0:00 ./primer1
 42280 pts/0    S+     0:00 ./primer1
 42281 pts/0    S+     0:00 ./primer1
 42282 pts/0    S+     0:00 ./primer1
 42283 pts/0    S+     0:00 ./primer1
```

- 下面是父进程sleep(1000)
  - 当父进程sleep1000满后, 父进程exit(0), 子进程变为`Z+`僵尸进程
  - init收留各孤儿进程, 当子进程exit()后, init会将各子进程收尸, 变为孤儿进程(所以僵尸态是一闪而过的) 
  - 僵尸态不占什么资源, 基本就是一个结构体指针(包含pid, 进程退出状态), 但是会占pid号(需要释放pid号)

```c
# include <stdio.h>
# include <stdlib.h>
# include <unistd.h>
# define LEFT 30000000
# define RIGHT 30000200
int main() {
  pid_t pid;
  int i, j, mark;
  for (i = LEFT; i <= RIGHT; i++) {

    pid = fork();
    if (pid < 0) {
			perror("fork");
      exit(1);
    }
    
    if (pid == 0) {
      mark = 1;
      for (j = 2; j < i/2; j++) {
        if (i % j == 0) {
          mark = 0;
          break;
        }
      }
      if (mark) {
        printf("%d is a primer\n", i);
      }   
      exit(0); // 子进程的退出方法, 防止子进程再fork孙进程
    }

  }

  sleep(1000);
  exit(0);
}
```

```bash
36188 ?        Ss     0:00  \_ sshd: ubuntu [priv]
 36190 ?        S      0:00  |   \_ sshd: ubuntu@pts/0
 36191 pts/0    Ss     0:00  |       \_ -bash
 43360 pts/0    S+     0:00  |           \_ ./primer1
 43361 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43362 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43363 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43364 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43365 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43366 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43367 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43368 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43369 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43370 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43371 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43372 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43373 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43374 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43375 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43376 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43377 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43378 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
 43379 pts/0    Z+     0:00  |               \_ [primer1] <defunct>
```

### init进程

- 是1号进程, 是所有进程的祖先进程

### vfork()

写时复制已合并到fork里了

## 进程的消亡及释放资源

```c
# include <stdio.h>
# include <stdlib.h>
# include <unistd.h>
# define LEFT 30000000
# define RIGHT 30000200
int main() {
  pid_t pid;
  int i, j, mark;
  for (i = LEFT; i <= RIGHT; i++) {

    pid = fork();
    if (pid < 0) {
			perror("fork");
      exit(1);
    }
    
    if (pid == 0) {
      mark = 1;
      for (j = 2; j < i/2; j++) {
        if (i % j == 0) {
          mark = 0;
          break;
        }
      }
      if (mark) {
        printf("%d is a primer\n", i);
      }   
      exit(0); // 子进程的退出方法, 防止子进程再fork孙进程
    }

  }
  
  for (i = LEFT; i <= RIGHT; i++) {
    wait(NULL); // 循环201次, 等201个子进程exit后, 收201个子进程的收尸
  }

  sleep(1000);
  exit(0);
}
```

- 分配方法
  - 分块法
  - 交叉分配法
  - 池

```c
# include <stdio.h>
# include <stdlib.h>
# include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
# define LEFT 30000000
# define RIGHT 30000200
# define N 3
int main() {
	pid_t pid;
	int i, j, n, mark;

	for (n = 0; n < N; n++) {
		pid = fork();
		if (pid < 0) {
			perror("");
			exit(1);
		}

		if (pid == 0) {
			for (i = LEFT+n; i <= RIGHT; i+=N) {
				mark = 1;
				for (j = 2; j < i/2; j++) {
					if (i % j == 0) {
						mark = 0;
						break;
					}
				}
				if (mark) {
					printf("[第%d个进程] 数字%d is a primer\n", n, i);
				}
			}
			exit(0); // 子进程的退出方法, 防止子进程再fork孙进程
		}
	}

	for (n = 0; n < N; n++) {
		wait(NULL); // 循环3次, 等3个子进程exit后, 收3个子进程的收尸
	}

	exit(0);
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ ./primerN
[第1个进程] 数字30000001 is a primer
[第2个进程] 数字30000023 is a primer
[第1个进程] 数字30000037 is a primer
[第2个进程] 数字30000041 is a primer
[第2个进程] 数字30000059 is a primer
[第1个进程] 数字30000049 is a primer
[第2个进程] 数字30000071 is a primer
[第1个进程] 数字30000079 is a primer
[第2个进程] 数字30000083 is a primer
[第1个进程] 数字30000109 is a primer
[第2个进程] 数字30000137 is a primer
[第1个进程] 数字30000133 is a primer
[第2个进程] 数字30000149 is a primer
[第1个进程] 数字30000163 is a primer
[第2个进程] 数字30000167 is a primer
[第1个进程] 数字30000169 is a primer
[第1个进程] 数字30000193 is a primer
[第1个进程] 数字30000199 is a primer

ubuntu@k8s-wolf-minion-47:~$ ./primerN | wc -l
18

ubuntu@k8s-wolf-minion-47:~$ time ./primerN > /dev/null
real	0m0.817s
user	0m1.444s
sys	0m0.000s
```

## exec函数族

- 注意先fflush(), 再exec()

```c
# include <stdio.h>
# include <stdlib.h>
# include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

/**
* date +%s
*/
int main() {
  puts("Begin!");
  
  fflush(NULL);
  execl("/bin/date", "date", "+%s", NULL); // 用date进程代替本进程, 若成功则不会返回, 失败才会返回
  
  perror("execl");
  exit(1);

  puts("End!");
  exit(0);
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ ./ex > /tmp/out
ubuntu@k8s-wolf-minion-47:~$ cat /tmp/out
1613972219
```

### fork+exec+wait组合

```c
# include <stdio.h>
# include <stdlib.h>
# include <unistd.h>
# include <sys/types.h>
# include <sys/wait.h>

int main() {
  puts("BEGIN");
  
  fflush(NULL);
  pid_t pid = fork();
  if (pid < 0) {
    perror("fork()");
    exit(0);
  }
  
  if (pid == 0) {
    execl("/bin/date", "date", "+%s", NULL);
    perror("execl()");
    exit(1);
  }
  
  wait(NULL);
  
  puts("END");
  
  exit(0);
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ ./few
BEGIN
1613972883
END
```

```c
          father process                                       child process
            file desc table         stdin                       file desc table
    +----------------+              +----+                +------------------------+
 0  |                |              |    | <-----------+  |                        |
    |                | +--------->  +----+                +------------------------+
    +----------------+               stdout               |                        |
 1  |                |            +------+ <--------------+                        |
    |                | +--------> |      |                |                        |
    +----------------+            +------+                +------------------------+
2   |                |              stderr                |                        |
    |                | +--------> +-------+ <-----------+ |                        |
    +----------------+            +-------+               +------------------------+
    |                |                                    |                        |
    |                |                                    |                        |
    |                |                                    |                        |
    |                |           fork()                   |                        |
    |                |  +----------------------------->   |                        |
    |                |                                    |                        |
    |                |                                    |                        |
    |                |                                    |                        |
    |                |                                    |                        |
    |                |                                    |                        |
    |                |                                    |                        |
    |                |                                    |                        |
    +----------------+                                    +------------------------+
```

### sleep(100)

```c
# include <stdio.h>
# include <stdlib.h>
# include <unistd.h>
# include <sys/types.h>
# include <sys/wait.h>

int main() {
  puts("BEGIN");
  
  fflush(NULL);
  pid_t pid = fork();
  if (pid < 0) {
    perror("fork()");
    exit(0);
  }
  
  if (pid == 0) {
    execl("/bin/sleep", "sleep", "100", NULL);
    perror("execl()");
    exit(1);
  }
  
  wait(NULL);
  
  puts("END");
  
  exit(0);
}
```

```bash
 66706 ?        Ss     0:00  \_ sshd: ubuntu [priv]
 66718 ?        S      0:00  |   \_ sshd: ubuntu@pts/0
 66719 pts/0    Ss     0:00  |       \_ -bash
 70741 pts/0    S+     0:00  |           \_ ./sleep
 70742 pts/0    S+     0:00  |               \_ sleep 100
```

### myshell练习

```c
# include <stdio.h>
# include <stdlib.h>
# include <unistd.h>
# include <sys/types.h>
# include <sys/wait.h>
# include <string.h>
# include <glob.h>
# define DELIMS " \t\n"

struct cmd_st {
  glob_t globres;
};

static void prompt(void) {
  printf("mysh-0.1$ ");
}

static void parse(char *line, struct cmd_st *res) {
  char *tok;
  int i = 0;
  
  while(1) {
    tok = strsep(&line, DELIMS);
    if (tok == NULL)
      break;
    if (tok[0] == '\0')
      continue;
  	glob(tok, GLOB_NOCHECK|GLOB_APPEND * i, NULL, &res->globres);
    i = 1; 
  }
}

int main() {
  
  char *linebuf = NULL;
  size_t linebuf_size = 0;
  struct cmd_st cmd;
  
  while(1) {
    prompt();
    if( getline(&linebuf, &linebuf_size, stdin) < 0 ) {
      break;
    }
    parse(linebuf, &cmd);
    if (0) { // 是内部命令
      
    } else { // 是外部命令
      pid_t pid = fork();
      if (pid < 0) {
        perror("fork()");
        exit(1);
      }
      if (pid == 0) {
        // 子进程
        execvp(cmd.globres.gl_pathv[0], cmd.globres.gl_pathv);
        perror("execvp()");
        exit(1);
      }else {
        // 父进程
        wait(NULL);
      }
    }
  }
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ make mysh
cc     mysh.c   -o mysh

ubuntu@k8s-wolf-minion-47:~$ ./mysh
mysh-0.1$ ls

big.c  ex.c  few.c  fork.c    getopt.c	mydu.c	mysh.c	      primer0	 primer1    primerN    read.c	 sleep	  time.c
ex     few   fork   getenv.c  jump.c	mysh	platformData  primer0.c  primer1.c  primerN.c  setjmp.c  sleep.c  write.c

mysh-0.1$ pwd
/home/ubuntu

mysh-0.1$ ll
execvp(): No such file or directory

mysh-0.1$ ls -l
total 164
-rw-r--r-- 1 ubuntu ubuntu  443 Feb 19 18:42 big.c
-rwxr-xr-x 1 ubuntu ubuntu 8752 Feb 22 13:36 ex
-rw-r--r-- 1 ubuntu ubuntu  342 Feb 22 13:36 ex.c
-rwxr-xr-x 1 ubuntu ubuntu 8904 Feb 22 13:47 few
-rw-r--r-- 1 ubuntu ubuntu  394 Feb 22 13:47 few.c
-rwxr-xr-x 1 ubuntu ubuntu 8864 Feb 22 01:10 fork
-rw-r--r-- 1 ubuntu ubuntu  420 Feb 22 01:10 fork.c
-rw-r--r-- 1 ubuntu ubuntu  105 Feb 21 16:20 getenv.c
-rw-r--r-- 1 ubuntu ubuntu 1922 Feb 21 23:01 getopt.c
-rw-r--r-- 1 ubuntu ubuntu  981 Feb 21 16:58 jump.c
-rw-r--r-- 1 ubuntu ubuntu 1262 Feb 20 11:54 mydu.c
-rwxr-xr-x 1 ubuntu ubuntu 9176 Feb 22 17:42 mysh
-rw-r--r-- 1 ubuntu ubuntu 1168 Feb 22 17:42 mysh.c
lrwxrwxrwx 1 root   root     13 Sep 14 16:31 platformData -> /platformData
-rwxr-xr-x 1 ubuntu ubuntu 8656 Feb 22 10:03 primer0
-rw-r--r-- 1 ubuntu ubuntu  372 Feb 22 10:03 primer0.c
-rwxr-xr-x 1 ubuntu ubuntu 8808 Feb 22 10:33 primer1
-rw-r--r-- 1 ubuntu ubuntu  563 Feb 22 10:33 primer1.c
-rwxr-xr-x 1 ubuntu ubuntu 8808 Feb 22 12:28 primerN
-rw-r--r-- 1 ubuntu ubuntu  770 Feb 22 12:28 primerN.c
-rw-r--r-- 1 ubuntu ubuntu 1068 Feb 19 13:24 read.c
-rw-r--r-- 1 ubuntu ubuntu 1228 Feb 21 21:54 setjmp.c
-rwxr-xr-x 1 ubuntu ubuntu 8912 Feb 22 13:56 sleep
-rw-r--r-- 1 ubuntu ubuntu  396 Feb 22 13:56 sleep.c
-rw-r--r-- 1 ubuntu ubuntu  294 Feb 21 22:21 time.c
-rw-r--r-- 1 ubuntu ubuntu  206 Feb 19 13:41 write.c
mysh-0.1$
```



### 用户权限和组权限

`u+s, g+s`的实现是通过fork + exec然后切换为对应effective user实现的

## 观摩课: 解释器文件

```bash
root@ubuntu:/home/ubuntu# chgrp ubuntu mysu
root@ubuntu:/home/ubuntu# ll mysu.c
-rw-r--r-- 1 root ubuntu 429 Feb 22 18:31 mysu

root@ubuntu:/home/ubuntu# chmod u+s mysu
root@ubuntu:/home/ubuntu# ll mysu.c
-rwSr--r-- 1 root ubuntu 429 Feb 22 18:31 mysu
```



```c
# include <stdio.h>
# include <stdlib.h>
# include <unistd.h>
# include <sys/types.h>
# include <sys/stat.h>
# include <sys/wait.h>

int main(int argc, char **argv) {
  if (argc < 3) {
    fprintf(stderr, "Usage...\n");
    exit(1);
  }

  pid_t pid = fork();
  if (pid < 0) {
    perror("fork");
    exit(1);
  }
  if (pid ==0) {
    setuid(atoi(argv[1]));
    execvp(argv[2], argv+2);
    perror("execvp");
    exit(1);
  }
  wait(NULL);
  exit(0);
}
```

```bash
ubuntu@ubuntu:~$ cat /etc/shadow
cat: /etc/shadow: Permission denied

ubuntu@ubuntu:~$ ./mysu 0 cat /etc/shadow
root:*:18519:0:99999:7:::
daemon:*:17743:0:99999:7:::
bin:*:17743:0:99999:7:::
sys:*:17743:0:99999:7:::
sync:*:17743:0:99999:7:::
games:*:17743:0:99999:7:::
man:*:17743:0:99999:7:::
lp:*:17743:0:99999:7:::
mail:*:17743:0:99999:7:::
news:*:17743:0:99999:7:::
uucp:*:17743:0:99999:7:::
proxy:*:17743:0:99999:7:::
www-data:*:17743:0:99999:7:::
backup:*:17743:0:99999:7:::
list:*:17743:0:99999:7:::
irc:*:17743:0:99999:7:::
gnats:*:17743:0:99999:7:::
nobody:*:17743:0:99999:7:::
systemd-timesync:*:17743:0:99999:7:::
systemd-network:*:17743:0:99999:7:::
systemd-resolve:*:17743:0:99999:7:::
systemd-bus-proxy:*:17743:0:99999:7:::
syslog:*:17743:0:99999:7:::
_apt:*:17743:0:99999:7:::
lxd:*:18519:0:99999:7:::
messagebus:*:18519:0:99999:7:::
uuidd:*:18519:0:99999:7:::
dnsmasq:*:18519:0:99999:7:::
sshd:*:18519:0:99999:7:::
ubuntu:$6$Be.6KxDz$5MH6Hm2yqudKmqroQO8uA04WdsdBoeE/JLR395vRGmANkRMlcam8/hZra18hWQjE5d57z7BbLPyjsyTHQYv9q.:18519:0:99999:7:::
ftp:*:18519:0:99999:7:::
postgres:*:18519:0:99999:7:::
```

## system()

```c
# include <stdio.h>
# include <stdlib.h>
int main(int argc, char **argv) {
  system("date +%s > /tmp/out");
  exit(0);
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ ./sys
ubuntu@k8s-wolf-minion-47:~$ cat /tmp/out
1613993971
```

- 是fork, exec, wait的封装, system()会调/bin/sh -c, 内部实际上是如下原理

```c
# include <stdio.h>
# include <stdlib.h>
# include <unistd.h>
# include <sys/types.h>
# include <sys/wait.h>

int main() {
  puts("BEGIN");
  
  fflush(NULL);
  pid_t pid = fork();
  if (pid < 0) {
    perror("fork()");
    exit(0);
  }
  
  if (pid == 0) {
    execl("/bin/sh", "sh", "-c", "date +%s", NULL);
    perror("execl()");
    exit(1);
  }
  
  wait(NULL);
  
  puts("END");
  
  exit(0);
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ ./sys1
BEGIN
1613994175
END
```

## 进程会计

acct

## 进程时间

times

## 守护进程

httpd sshd vsftpd dhcp

守护进程是没有stdin的(会重定向或close), (因为守护进程是后台进程, 一旦stdin输入会把守护进程杀死)

如下3个特点:

- 子进程调用setsid()使sessionid, processid, processgroupid相同
- 并且其父进程退出, 使init为其父进程
- tty为空

getpgrp(), getpgid(), setpgid()

- 下面写一个守护进程

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/stat.h>
#include <fcntl.h>
#define FNAME "/tmp/out"
static int daemonize() {
  pid_t pid = fork();
  if (pid < 0) {
    perror("fork()");
    return -1;
  }
  if (pid > 0)  // parent
    exit(0);

  // ------以下为child------
  // 重定向stdin, stdout, stderr
  int fd = open("/dev/null", O_RDWR);
  if (fd < 0) {
    perror("open");
    return -1;
  }
  dup2(fd, 0);
  dup2(fd, 1);
  dup2(fd, 2);
  if (fd > 2) 
    close(fd);

  setsid();

  chdir("/");
  umask(0);// 如果不产生文件的话, 可以关掉umask

  return 0;
}

int main() {
  if (daemonize()) {
    exit(1);
  }
  FILE *fp = fopen(FNAME, "w");
  if (fp == NULL) {
    perror("fopen()");
    exit(1);
  }
  int i = 0;
  for ( i = 0; ; i++) {
    fprintf(fp, "%d", i);
    fflush(fp);
    sleep(1);
  }
  exit(0);
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ ./mydaemon
ubuntu@k8s-wolf-minion-47:~$ cat /tmp/out
01234ubuntu@k8s-wolf-minion-47:~$ cat /tmp/out
0123456ubuntu@k8s-wolf-minion-47:~$ cat /tmp/out
01234567ubuntu@k8s-wolf-minion-47:~$ cat /tmp/out
012345678ubuntu@k8s-wolf-minion-47:~$

ubuntu@k8s-wolf-minion-47:~$ ps axj
  PPID    PID   PGID    SID TTY       TPGID STAT   UID   TIME COMMAND
     1 126975 126975 126975 ?            -1 Ss    1000   0:00 ./mydaemon
     
ubuntu@k8s-wolf-minion-47:~$ kill 126975
```

- vsftpd的守护进程

```bash
root@k8s-wolf-minion-47:~# service vsftpd status
● vsftpd.service - vsftpd FTP server
   Loaded: loaded (/lib/systemd/system/vsftpd.service; enabled; vendor preset: enabled)
   Active: active (running) since Fri 2020-11-06 11:50:53 CST; 3 months 17 days ago
 Main PID: 1532 (vsftpd)
    Tasks: 1
   Memory: 1.2M
      CPU: 6ms
   CGroup: /system.slice/vsftpd.service
           └─1532 /usr/sbin/vsftpd /etc/vsftpd.conf

Warning: Journal has been rotated since unit was started. Log output is incomplete or unavailable.
root@k8s-wolf-minion-47:~# service vsftpd stop
root@k8s-wolf-minion-47:~# service vsftpd start
root@k8s-wolf-minion-47:~# service vsftpd start
root@k8s-wolf-minion-47:~# ps axj  | grep ftp
     1 145688 145688 145688 ?            -1 Ss       0   0:00 /usr/sbin/vsftpd /etc/vsftpd.conf
```

### 单实例守护进程-锁文件

- /var/run/xxx.pid`

```bash
root@k8s-wolf-minion-47:/var/run# ll | grep pid
-rw-r--r--  1 root     root        5 Nov  6 11:50 acpid.pid
srw-rw-rw-  1 root     root        0 Nov  6 11:50 acpid.socket=
-rw-r--r--  1 root     root        5 Nov  6 11:50 atd.pid
-rw-r--r--  1 root     root        5 Nov  6 11:50 crond.pid
-rw-r--r--  1 root     root        4 Nov  6 11:50 docker.pid
-rw-r--r--  1 root     root        5 Nov  6 11:50 irqbalance.pid
-rw-------  1 root     root        5 Nov  6 11:50 iscsid.pid
-rw-r--r--  1 root     root        4 Nov  6 11:50 lvmetad.pid
-rw-------  1 root     root        5 Nov  6 11:50 lxcfs.pid
-rw-r--r--  1 root     root        4 Nov  6 11:50 rsyslogd.pid
-rw-r--r--  1 root     root        6 Nov 23 11:14 sshd.pid
-rw-r--r--  1 root     root        5 Dec 14 10:47 supervisord.pid
root@k8s-wolf-minion-47:/var/run# cat supervisord.pid
7148
root@k8s-wolf-minion-47:/var/run# cat sshd.pid
38648


root@k8s-wolf-minion-47:/var/run# ps axj | grep 7148
  7148    910    910   7148 ?            -1 S        0   0:00 /bin/bash /home/ubuntu/platformTG/dbtool/latest/sv_start.sh
   910    911    910   7148 ?            -1 Sl       0  41:26 ./dbtool
     1   7148   7148   7148 ?            -1 Ss       0  86:27 /usr/bin/python /usr/bin/supervisord -n -c /etc/supervisor/supervisord.conf
     
     
root@k8s-wolf-minion-47:/var/run# ps axj | grep 38648
     1  38648  38648  38648 ?            -1 Ss       0   0:00 sshd: /usr/sbin/sshd -D -f /etc/ssh/sshd_config [listener] 0 of 10-100 startups
 38648 139349 139349 139349 ?            -1 Ss       0   0:00 sshd: ubuntu [priv]
 
 
 root@k8s-wolf-minion-47: ps axf -p 7148
  38648 ?        Ss     0:00 sshd: /usr/sbin/sshd -D -f /etc/ssh/sshd_config [listener] 0 of 10-100 startups
139349 ?        Ss     0:00  \_ sshd: ubuntu [priv]
139392 ?        S      0:00      \_ sshd: ubuntu@pts/0
139403 pts/0    Ss     0:00          \_ -bash
143215 pts/0    S      0:00              \_ sudo -i
143216 pts/0    S      0:00                  \_ -bash
146989 pts/0    R+     0:00                      \_ ps axf -p 7148
  7148 ?        Ss    86:27 /usr/bin/python /usr/bin/supervisord -n -c /etc/supervisor/supervisord.conf
   910 ?        S      0:00  \_ /bin/bash /home/ubuntu/platformTG/dbtool/latest/sv_start.sh
   911 ?        Sl    41:26      \_ ./dbtool
```

### 守护进程开机启动脚本文件

- `/etc/rc...`, 一般把启动命令(如启动pg/sshd/ftpd)放在这里, 会使守护进程开机启动, 从ubuntu看是软链接到了`/etc/init.d`

```bash
root@k8s-wolf-minion-47:/etc/rc0.d# ll
total 12
drwxr-xr-x   2 root root 4096 Dec 14 10:47 ./
drwxr-xr-x 101 root root 4096 Feb 22 23:14 ../
lrwxrwxrwx   1 root root   13 Sep 14 12:02 K01atd -> ../init.d/atd*
lrwxrwxrwx   1 root root   16 Sep 14 16:32 K01docker -> ../init.d/docker*
lrwxrwxrwx   1 root root   18 Nov 13 16:32 K01ebtables -> ../init.d/ebtables*
lrwxrwxrwx   1 root root   20 Sep 14 12:01 K01irqbalance -> ../init.d/irqbalance*
lrwxrwxrwx   1 root root   22 Sep 14 11:57 K01lvm2-lvmetad -> ../init.d/lvm2-lvmetad*
lrwxrwxrwx   1 root root   23 Sep 14 11:57 K01lvm2-lvmpolld -> ../init.d/lvm2-lvmpolld*
lrwxrwxrwx   1 root root   15 Sep 14 12:01 K01lxcfs -> ../init.d/lxcfs*
lrwxrwxrwx   1 root root   13 Sep 14 12:02 K01lxd -> ../init.d/lxd*
lrwxrwxrwx   1 root root   15 Sep 14 12:02 K01mdadm -> ../init.d/mdadm*
lrwxrwxrwx   1 root root   27 Nov 13 16:35 K01nfs-kernel-server -> ../init.d/nfs-kernel-server*
lrwxrwxrwx   1 root root   20 Sep 14 12:01 K01open-iscsi -> ../init.d/open-iscsi*
lrwxrwxrwx   1 root root   23 Sep 14 12:02 K01open-vm-tools -> ../init.d/open-vm-tools*
lrwxrwxrwx   1 root root   18 Sep 14 12:01 K01plymouth -> ../init.d/plymouth*
lrwxrwxrwx   1 root root   20 Sep 14 16:46 K01postgresql -> ../init.d/postgresql*
lrwxrwxrwx   1 root root   20 Sep 14 11:55 K01resolvconf -> ../init.d/resolvconf*
lrwxrwxrwx   1 root root   20 Dec 14 10:47 K01supervisor -> ../init.d/supervisor*
lrwxrwxrwx   1 root root   29 Sep 14 12:02 K01unattended-upgrades -> ../init.d/unattended-upgrades*
lrwxrwxrwx   1 root root   17 Sep 14 11:55 K01urandom -> ../init.d/urandom*
lrwxrwxrwx   1 root root   15 Sep 14 12:01 K01uuidd -> ../init.d/uuidd*
lrwxrwxrwx   1 root root   16 Sep 14 16:32 K01vsftpd -> ../init.d/vsftpd*
lrwxrwxrwx   1 root root   24 Sep 14 16:32 K02cgroupfs-mount -> ../init.d/cgroupfs-mount*
lrwxrwxrwx   1 root root   16 Sep 14 12:01 K02iscsid -> ../init.d/iscsid*
lrwxrwxrwx   1 root root   18 Sep 14 12:01 K03sendsigs -> ../init.d/sendsigs*
lrwxrwxrwx   1 root root   17 Sep 14 12:01 K04rsyslog -> ../init.d/rsyslog*
lrwxrwxrwx   1 root root   20 Sep 14 12:01 K05hwclock.sh -> ../init.d/hwclock.sh*
lrwxrwxrwx   1 root root   22 Sep 14 12:01 K05umountnfs.sh -> ../init.d/umountnfs.sh*
lrwxrwxrwx   1 root root   17 Nov 13 16:34 K06rpcbind -> ../init.d/rpcbind*
lrwxrwxrwx   1 root root   20 Nov 13 16:34 K07networking -> ../init.d/networking*
lrwxrwxrwx   1 root root   18 Nov 13 16:34 K08umountfs -> ../init.d/umountfs*
lrwxrwxrwx   1 root root   20 Nov 13 16:34 K09cryptdisks -> ../init.d/cryptdisks*
lrwxrwxrwx   1 root root   26 Nov 13 16:34 K10cryptdisks-early -> ../init.d/cryptdisks-early*
lrwxrwxrwx   1 root root   20 Nov 13 16:34 K11umountroot -> ../init.d/umountroot*
lrwxrwxrwx   1 root root   24 Nov 13 16:34 K12mdadm-waitidle -> ../init.d/mdadm-waitidle*
lrwxrwxrwx   1 root root   14 Nov 13 16:34 K13halt -> ../init.d/halt*
-rw-r--r--   1 root root  353 Jan 20  2016 README
root@k8s-wolf-minion-47:/etc/rc0.d# cd ..
root@k8s-wolf-minion-47:/etc# ll rc1.d/
total 12
drwxr-xr-x   2 root root 4096 Dec 14 10:47 ./
drwxr-xr-x 101 root root 4096 Feb 22 23:14 ../
lrwxrwxrwx   1 root root   13 Sep 14 12:02 K01atd -> ../init.d/atd*
lrwxrwxrwx   1 root root   16 Sep 14 16:32 K01docker -> ../init.d/docker*
lrwxrwxrwx   1 root root   18 Nov 13 16:32 K01ebtables -> ../init.d/ebtables*
lrwxrwxrwx   1 root root   20 Sep 14 12:01 K01irqbalance -> ../init.d/irqbalance*
lrwxrwxrwx   1 root root   22 Sep 14 11:57 K01lvm2-lvmetad -> ../init.d/lvm2-lvmetad*
lrwxrwxrwx   1 root root   23 Sep 14 11:57 K01lvm2-lvmpolld -> ../init.d/lvm2-lvmpolld*
lrwxrwxrwx   1 root root   15 Sep 14 12:01 K01lxcfs -> ../init.d/lxcfs*
lrwxrwxrwx   1 root root   13 Sep 14 12:02 K01lxd -> ../init.d/lxd*
lrwxrwxrwx   1 root root   15 Sep 14 12:02 K01mdadm -> ../init.d/mdadm*
lrwxrwxrwx   1 root root   27 Nov 13 16:35 K01nfs-kernel-server -> ../init.d/nfs-kernel-server*
lrwxrwxrwx   1 root root   20 Sep 14 12:01 K01open-iscsi -> ../init.d/open-iscsi*
lrwxrwxrwx   1 root root   23 Sep 14 12:02 K01open-vm-tools -> ../init.d/open-vm-tools*
lrwxrwxrwx   1 root root   20 Sep 14 16:46 K01postgresql -> ../init.d/postgresql*
lrwxrwxrwx   1 root root   20 Dec 14 10:47 K01supervisor -> ../init.d/supervisor*
lrwxrwxrwx   1 root root   13 Sep 14 12:02 K01ufw -> ../init.d/ufw*
lrwxrwxrwx   1 root root   15 Sep 14 12:01 K01uuidd -> ../init.d/uuidd*
lrwxrwxrwx   1 root root   16 Sep 14 16:32 K01vsftpd -> ../init.d/vsftpd*
lrwxrwxrwx   1 root root   24 Sep 14 16:32 K02cgroupfs-mount -> ../init.d/cgroupfs-mount*
lrwxrwxrwx   1 root root   16 Sep 14 12:01 K02iscsid -> ../init.d/iscsid*
lrwxrwxrwx   1 root root   17 Sep 14 12:01 K04rsyslog -> ../init.d/rsyslog*
lrwxrwxrwx   1 root root   17 Nov 13 16:34 K06rpcbind -> ../init.d/rpcbind*
-rw-r--r--   1 root root  369 Jan 20  2016 README
lrwxrwxrwx   1 root root   19 Sep 14 11:55 S01killprocs -> ../init.d/killprocs*
lrwxrwxrwx   1 root root   16 Sep 14 11:55 S02single -> ../init.d/single*
root@k8s-wolf-minion-47:/etc# ll -R rc* | grep postgres
lrwxrwxrwx   1 root root   20 Sep 14 16:46 K01postgresql -> ../init.d/postgresql*
lrwxrwxrwx   1 root root   20 Sep 14 16:46 K01postgresql -> ../init.d/postgresql*
lrwxrwxrwx   1 root root   20 Sep 14 16:46 S02postgresql -> ../init.d/postgresql*
lrwxrwxrwx   1 root root   20 Sep 14 16:46 S02postgresql -> ../init.d/postgresql*
lrwxrwxrwx   1 root root   20 Sep 14 16:46 S02postgresql -> ../init.d/postgresql*
lrwxrwxrwx   1 root root   20 Sep 14 16:46 S02postgresql -> ../init.d/postgresql*
lrwxrwxrwx   1 root root   20 Sep 14 16:46 K01postgresql -> ../init.d/postgresql*
```

## 系统日志

```bash
ubuntu@k8s-wolf-minion-47:/var/log$ ll | grep -v gz
total 823284
drwxrwxr-x 13 root   syslog        4096 Feb 21 06:25 ./
drwxr-xr-x 14 root   root          4096 Sep 14 16:46 ../
-rw-r--r--  1 root   root           505 Feb 10 11:03 alternatives.log
-rw-r--r--  1 root   root          1000 Feb  3 17:23 alternatives.log.1
-rw-r--r--  1 root   root        149090 Sep 16 16:13 ansible.log
drwxr-xr-x  2 root   root          4096 Feb  4 06:25 apt/
-rw-r-----  1 syslog adm          49055 Feb 22 20:55 auth.log
-rw-r-----  1 syslog adm         224972 Feb 21 06:25 auth.log.1
-rw-r--r--  1 root   root         57457 Jul 31  2018 bootstrap.log
-rw-rw----  1 root   utmp             0 Feb  1 06:25 btmp
-rw-rw----  1 root   utmp             0 Jan  1 06:25 btmp.1
drwxr-xr-x  2 root   root          4096 Jan  5 15:53 containers/
drwxr-xr-x  2 root   root          4096 Apr  9  2018 dist-upgrade/
-rw-r-----  1 root   adm             31 Jul 31  2018 dmesg
-rw-r--r--  1 root   root         27128 Feb 10 11:03 dpkg.log
-rw-r--r--  1 root   root         23078 Feb  3 17:40 dpkg.log.1
-rw-r--r--  1 root   root         32096 Feb 22 17:47 faillog
drwxr-xr-x  2 root   root          4096 Sep 14 11:55 fsck/
drwxr-xr-x  3 root   root          4096 Sep 14 12:05 installer/
-rw-r-----  1 syslog adm              0 Jan 18 06:25 kern.log
-rw-r-----  1 syslog adm           1522 Jan 12 11:26 kern.log.1
-rw-rw-r--  1 root   utmp        292876 Feb 22 20:40 lastlog
drwxr-xr-x  2 root   root          4096 Dec  8  2017 lxd/
drwxr-xr-x  2 root   root          4096 Jan  5 15:53 pods/
drwxrwxr-t  2 root   postgres      4096 Jan 31 06:25 postgresql/
drwxr-xr-x  2 root   root          4096 Dec 14 10:47 supervisor/
-rw-r-----  1 syslog adm      446618132 Feb 22 20:59 syslog
-rw-r-----  1 syslog adm      277778484 Feb 21 06:25 syslog.1
drwxr-xr-x  2 root   root          4096 Dec 11  2017 sysstat/
drwxr-x---  2 root   adm           4096 Oct 20 16:28 unattended-upgrades/
-rw-rw-r--  1 root   utmp         38016 Feb 22 20:50 wtmp
-rw-rw-r--  1 root   utmp         72576 Jan 29 18:09 wtmp.1
```

- syslogd守护进程负责

```bash
root@k8s-wolf-minion-47:~# ps aux | grep syslogd
syslog     1311  0.0  0.0 256392  4304 ?        Ssl   2020  60:28 /usr/sbin/rsyslogd -n
```

- Openlog 

- closelog

- Syslog 

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>
#include <errno.h>
#include <syslog.h>
#define FNAME "/tmp/out"
static int daemonize() {
  pid_t pid = fork();
  if (pid < 0) {
    return -1;
  }
  if (pid > 0)  // parent
    exit(0);

  // ------以下为child------
  // 重定向stdin, stdout, stderr
  int fd = open("/dev/null", O_RDWR);
  if (fd < 0) {
    return -1;
  }
  dup2(fd, 0);
  dup2(fd, 1);
  dup2(fd, 2);
  if (fd > 2) 
    close(fd);

  setsid();

  chdir("/");
  umask(0);// 如果不产生文件的话, 可以关掉umask

  return 0;
}

int main() {
  openlog("mydaemon", LOG_PID, LOG_DAEMON);
  
  if (daemonize()) {
    syslog(LOG_ERR, "daemonize() failed!"); // 注意这里不要加\n, syslog自己控制格式
    exit(1);
  } 
  syslog(LOG_INFO, "daemonize() seccessed!");
  
  FILE *fp = fopen(FNAME, "w");
  if (fp == NULL) {
    syslog(LOG_ERR, "fopen:%s", strerror(errno));
    exit(1);
  }
  syslog(LOG_INFO, "%s was opened.", FNAME);
  
  int i = 0;
  for ( i = 0; ; i++) {
    fprintf(fp, "%d", i);
    fflush(fp);
    syslog(LOG_DEBUG, "%d is printed.", i);
    sleep(1);
  }
  
  fclose(fp);
  closelog();
  exit(0);
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ ./mydaemon
ubuntu@k8s-wolf-minion-47:~$ ps -ef | grep mydaemon
ubuntu   143056      1  0 22:56 ?        00:00:00 ./mydaemon


root@k8s-wolf-minion-47:~# grep 143056 /var/log/syslog
Feb 22 22:56:11 k8s-wolf-minion-47 mydaemon[143056]: daemonize() seccessed!
Feb 22 22:56:11 k8s-wolf-minion-47 mydaemon[143056]: /tmp/out was opened.
Feb 22 22:56:11 k8s-wolf-minion-47 mydaemon[143056]: 0 is printed.
Feb 22 22:56:12 k8s-wolf-minion-47 mydaemon[143056]: 1 is printed.
Feb 22 22:56:13 k8s-wolf-minion-47 mydaemon[143056]: 2 is printed.
Feb 22 22:56:14 k8s-wolf-minion-47 mydaemon[143056]: 3 is printed.
Feb 22 22:56:15 k8s-wolf-minion-47 mydaemon[143056]: 4 is printed.
Feb 22 22:56:16 k8s-wolf-minion-47 mydaemon[143056]: 5 is printed.
Feb 22 22:56:17 k8s-wolf-minion-47 mydaemon[143056]: 6 is printed.
Feb 22 22:56:18 k8s-wolf-minion-47 mydaemon[143056]: 7 is printed.
Feb 22 22:56:19 k8s-wolf-minion-47 mydaemon[143056]: 8 is printed.
Feb 22 22:56:20 k8s-wolf-minion-47 mydaemon[143056]: 9 is printed.


root@k8s-wolf-minion-47:~# vim /etc/rsyslog.conf
```

# 并发

- 同步

- 异步: 事件到来时间未知, 事件到来的后果未知
  - 主动轮询法: 适合于事件发生频率频繁时, (eg钓鱼, 鱼很多, 适合轮询法: 每1min拿一个网子去捞鱼)
  - 被动通知法: 适合于事件发生频率稀疏时, (eg钓鱼, 鱼很少, 适合通知法: 甩钓竿和浮漂, 等浮漂上下浮动过快时通知自己去捞鱼, 省力气)



- 单核: 各进程按时间片分片, 没有真正意义的并行
- 多核: 才有并行



- 信号和多线程一般不会混用

## 通过信号实现并发

### 信号的概念

信号是软件中断, 信号(应用软件层)的响应依赖于中断(硬件层).

### signal()

```bash
ubuntu@k8s-wolf-minion-47:~$ kill -l
 1) SIGHUP	 2) SIGINT	 3) SIGQUIT	 4) SIGILL	 5) SIGTRAP
 6) SIGABRT	 7) SIGBUS	 8) SIGFPE	 9) SIGKILL	10) SIGUSR1
11) SIGSEGV	12) SIGUSR2	13) SIGPIPE	14) SIGALRM	15) SIGTERM
16) SIGSTKFLT	17) SIGCHLD	18) SIGCONT	19) SIGSTOP	20) SIGTSTP
21) SIGTTIN	22) SIGTTOU	23) SIGURG	24) SIGXCPU	25) SIGXFSZ
26) SIGVTALRM	27) SIGPROF	28) SIGWINCH	29) SIGIO	30) SIGPWR
31) SIGSYS	34) SIGRTMIN	35) SIGRTMIN+1	36) SIGRTMIN+2	37) SIGRTMIN+3
38) SIGRTMIN+4	39) SIGRTMIN+5	40) SIGRTMIN+6	41) SIGRTMIN+7	42) SIGRTMIN+8
43) SIGRTMIN+9	44) SIGRTMIN+10	45) SIGRTMIN+11	46) SIGRTMIN+12	47) SIGRTMIN+13
48) SIGRTMIN+14	49) SIGRTMIN+15	50) SIGRTMAX-14	51) SIGRTMAX-13	52) SIGRTMAX-12
53) SIGRTMAX-11	54) SIGRTMAX-10	55) SIGRTMAX-9	56) SIGRTMAX-8	57) SIGRTMAX-7
58) SIGRTMAX-6	59) SIGRTMAX-5	60) SIGRTMAX-4	61) SIGRTMAX-3	62) SIGRTMAX-2
63) SIGRTMAX-1	64) SIGRTMAX
```

- 1~31为标准信号
- 34~64为RT, real time实时信号



- 函数原型如下

```c
void (*signal(int signum, void (*func) (int))) (int);

typedef void (*sighandler_t)(int);
sighandler_t signal(int signum, sighandler_t handler);
```

- 例子如下

```c
# include <stdio.h>
# include <stdlib.h>
# include <signal.h>
# include <unistd.h>

int main() {
  int i;
  for (i = 0; i < 10; i++) {
    write(1, "*", 1);
    sleep(1);
  }
  exit(0);
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ ./star
***^C
```

^c会发送SIGINT信号

- 第二个例子

```c
# include <stdio.h>
# include <stdlib.h>
# include <signal.h>
# include <unistd.h>

int main() {
  int i;
  signal(SIGINT, SIG_IGN);
  for (i = 0; i < 10; i++) {
    write(1, "*", 1);
    sleep(1);
  }
  exit(0);
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ ./star
***^C*******
```

现在这样确实^c打断不了了, 因为忽略了SIGINT信号

- 第三个例子

```c
# include <stdio.h>
# include <stdlib.h>
# include <signal.h>
# include <unistd.h>

static void int_handler(int s) {
  write(1, "!", 1);
}

int main() {
  int i;
  signal(SIGINT, int_handler);
  for (i = 0; i < 10; i++) {
    write(1, "*", 1);
    sleep(1);
  }
  exit(0);
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ ./star
**^C!**^C!*^C!*^C!*^C!*^C!**ubuntu@k8s-wolf-minion-47:~$
```

- 信号会打断阻塞的系统调用
  - 如下例, 一直按住ctrl +C, 不到10s, 程序就结束了
  - open阻塞, read阻塞, 都会被signal打断, 需要判断errno的返回是否为EINTR, 若是则忽略(因为是被打断了), 否则才是真的出错了

```c
# include <stdio.h>
# include <stdlib.h>
# include <signal.h>
# include <unistd.h>

static void int_handler(int s) {
  write(1, "!", 1);
}

int main() {
  int i;
  signal(SIGINT, int_handler);
  for (i = 0; i < 10; i++) {
    write(1, "*", 1);
    sleep(1);
  }
  exit(0);
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ ./star
*^C!*^C!*^C!*^C!*^C!*^C!*^C!*^C!*^C!*^C!ubuntu@k8s-wolf-minion-47:~$ ^C
```

#### 生成core dumped

[u16.04开启coredump并设置core文件的产生位置]

(https://blog.csdn.net/u013937038/article/details/106757765/)

[如何利用core dump + gdb](https://blog.csdn.net/qq_39759656/article/details/82858101)

> 编译的时候带上gcc -g, 然后ulimit -c设置core size, 然后/etc/sysctl.conf设置core_pattern和core_uses_pid参数, 如果有段错误就会生成core文件, 然后用gdb调试可以找到时哪一行的问题, 注意一般core文件会很大

```c
#include <stdio.h>
int main(void) {
  int *a = NULL;
  *a = 0x1;
  return 0;
}
```

```bash
ubuntu@k8s-wolf-minion-47:~$ ./getline_code /tmp/out
Segmentation fault

ubuntu@k8s-wolf-minion-47:~$ ulimit -a | grep 'core file size'
core file size          (blocks, -c) 0

ubuntu@k8s-wolf-minion-47:~$ ulimit -c 10240
ubuntu@k8s-wolf-minion-47:~$ ulimit -a | grep 'core file size'
core file size          (blocks, -c) 10240

root@k8s-wolf-minion-47:/home/ubuntu# gcc a.c -o a -g
root@k8s-wolf-minion-47:~# ./a
Segmentation fault (core dumped)
root@k8s-wolf-minion-47:~# ls
a  a.c  drop_cluster.sh  platform_0815-release-c1f7aa9-20200814  ssh_20200914

root@k8s-wolf-minion-47:/home/ubuntu# ll /devo/null
total 116
drwxr-xr-x 2 root root   4096 Feb 23 00:52 ./
drwxr-xr-x 3 root root   4096 Feb 23 00:34 ../
-rw------- 1 root root 249856 Feb 23 00:52 core_a_1614013093_161183

root@k8s-wolf-minion-47:/home/ubuntu# gdb a
(gdb) core-file /devo/null/core_a_1614013093_161183
[New LWP 161183]
Core was generated by `./a'.
Program terminated with signal SIGSEGV, Segmentation fault.
#0  0x00000000004004e6 in main () at a.c:4
4	  *a = 0x1;

```

### 信号的行为不可靠

因为信号是未知的突发情况, 而不是人为写的逻辑代码, 所以是内核布置的现场, 每次可能不一样, unix引入了一个链式结构解决了这个问题

### 可重入函数

### 信号的响应过程

### 常用函数

#### kill()

#### Raise()

#### alarm()

#### Pause()

#### Abort()

#### System()

#### sleep()

### 信号集

### 信号屏蔽字/pending集的处理

### 扩展

#### sigsuspend()

#### sigaction();

#### Settitimer();

### 实时信号

## 通过多线程实现并发



















































# 其他

[CLion工程中只能有一个main函数 &&怎么同时编写多个main函数的C文件](https://blog.csdn.net/justinzwd/article/details/85206640)

```makefile
cmake_minimum_required(VERSION 3.13)
project(address C)

set (CMAKE_C_STANDARD 90)

add_executable(main01 Chapter-01/1.4.3.c)
```

[Unix环境编程中的apue.h和err_quit、err_sys问题](https://blog.csdn.net/u012814984/article/details/44751595)

## 文件和目录

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

## 输入和输出

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

## 程序和进程

- 进程ID
```c
#include "../apue.3e/include/apue.h"

int main(int argc, char *argv[]) {
    printf("hello world from process ID %ld\n", (long)getpid());
    exit(0);
}
```

## 出错处理

```c
#include "../apue.3e/include/apue.h"

#include <errno.h>

int main(int argc, char *argv[]) {
    fprintf(stderr, "EACCES: %s\n", strerror(EACCES));
    errno = ENOENT;
    perror(argv[0]);
    exit(0);
}

EACCES: Permission denied
~/go/src/github.com/sonnary/apue/cmake-build-debug/main17: No such file or directory
```

## 用户标识

```c
int main(int argc, char *argv[]) {
    printf("uid = %d, gid = %d\n", getuid(), getgid());
    exit(0);
}

uid = 501, gid = 20
 ✘  /etc  cat /etc/group | grep staff
staff:*:20:root
```

## 信号

```c
#include "../apue.3e/include/apue.h"
#include <sys/wait.h>

static void sig_int(int);

int main(int argc, char *argv[]) {
    char buf[MAXLINE];
    pid_t pid;
    int status;

    printf("pid: %d\n", getpid());

    if (signal(SIGINT, sig_int) == SIG_ERR)
        err_sys("signal error");
    printf("%% ");
    while (fgets(buf, MAXLINE, stdin) != NULL) {
        if (buf[strlen(buf) - 1] == '\n')
            buf[strlen(buf) - 1] = 0;

        if ((pid = fork()) < 0) {
            err_sys("fork error");
        } else if (pid == 0) {
            execlp(buf, buf, (char *) 0);
            err_ret("couldn't execute: %s", buf);
            exit(127);
        }

        if ((pid = waitpid(pid, &status, 0)) < 0)
            err_sys("waitpid error");
        printf("%% ");
    }
    exit(0);
}

void sig_int(int signo) {
    printf("interrupt\n%% ");
}
```

## 时间值

```bash
/etc  time cat 1>1
zsh: permission denied: 1
cat > 1  0.00s user 0.00s system 58% cpu 0.001 total
 ✘  /etc  time rm 1
rm: 1: No such file or directory
rm 1  0.00s user 0.00s system 64% cpu 0.003 total
```

## 系统调用和库函数

```bash
awk -f 2.5.4.awk
用awk转换为c代码, 然后执行该c代码(2.5.4.c)
```

