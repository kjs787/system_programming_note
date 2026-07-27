配置实验环境：  
使用**wsl + ubuntu20.24 + vscode**配置linux开发环境

该项目的基本流程是这样，先按照要求完成代码，将写完的代码使用make qemu编译到xv6中，

在这个小型操作系统中验证代码运行效果，使用 ./grade-lab-util xxx 测试代码是否通过标准。

  

## 实现sleep

```
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[])
{
    if (argc != 2)
    {
        printf("ERROR: argcs number error\n");
    }
    sleep(atoi(argv[1]));
    exit(0);
}

//使用该命令进行测试： ./grade-lab-util sleep
```

主要是熟悉基本流程，系统调用的声明在user.h中，一些类型的声明在types.h 和 stat.h

`sleep`系统调用的xv6内核代码:

```
uint64
sys_sleep(void)
{
  int n;
  uint ticks0;

  if(argint(0, &n) < 0)
    return -1;
  acquire(&tickslock);
  ticks0 = ticks;
  while(ticks - ticks0 < n){
    if(myproc()->killed){
      release(&tickslock);
      return -1;
    }
    sleep(&ticks, &tickslock);
  }
  release(&tickslock);
  return 0;
}
```

`sleep`的汇编代码：

```
sleep:
 li a7, SYS_sleep
 ecall
 ret
.global uptime
```

  

## 实现PingPong

```
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int p[2];

int main(int argc, char *argv[])
{
    if (argc != 1)
    {
        printf("ERROR: argcs number error\n");
        exit(1);
    }

    pipe(p);
    char buf[1];
    if (fork() == 0)
    {
        // 子进程
        close(p[1]);
        read(p[0], buf, 1);
        printf("%d: received ping\n", getpid());
        write(p[0], buf, 1);
    }
    else
    {
        close(p[0]);
        write(p[1], "p", 1);
        read(p[1], buf, 1);
        wait(0);
        printf("%d: received pong\n", getpid());
        // 父进程
    }

    exit(0);
}
```

使用系统调用pipe，在父子进程间建立一个管道，实现pingpong。

pipe要传入一个int[2]类型的数组指针，在内核建立一个缓冲区供进程通信。其实是分配俩个fd，一端读入，另一端读出。惯用法是父子进程各关闭不同一端，用另外的一段进行通信。

## 使用管道实现素数筛

多进程实现素数筛，用管道在进程间传递素数。

有几个关键点：

- 不使用的文件描述符需要关闭，每各进程的文件描述符表是独立的。
- 先创建管道，等同层所有符合要求的素数写入管道完成，再创建子进程。
- 宏定义WR, RD来标识管道的读写端

```
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#define WR 1
#define RD 0

int EXIT_STATS = 0;

void transmit_data(int lpipe[2], int rpipe[2], int first)
{

    int data;
    // 先将所有不能被整除的数写入右管道
    while (read(lpipe[RD], &data, sizeof(int)) == sizeof(int))
    {
        if (data % first != 0)
        {
            write(rpipe[WR], &data, sizeof(int));
        }
    }

    // 关闭不用的文件描述符
    // 对左管道的读取和右管道的写入已经完成
    close(lpipe[RD]);
    close(rpipe[WR]);
}

void primes(int lpipe[2])
{
    close(lpipe[WR]);

    int first;
    if (read(lpipe[RD], &first, sizeof(int)) == sizeof(int))
    {
        printf("prime %d\n", first);
        int p[2];
        pipe(p);
        transmit_data(lpipe, p, first);

        // 进行递归
        if (fork() == 0)
        {
            primes(p);
        }
        else
        {
            close(p[RD]);
            wait(0);
            exit(0);
        }
    }
    exit(0);
}

int main(int argc, char *argv[])
{
    int p[2];
    pipe(p);
    for (int i = 2; i <= 35; i++)
    {
        write(p[WR], &i, sizeof(int));
    }

    if (fork() == 0)
    {
        primes(p);
    }
    else
    {
        close(p[WR]);
        close(p[RD]);
        wait(0);
    }
    exit(0);
}
```

具体原理如下：

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1779588023418-c5857213-8424-43da-8e7a-582c10cde6ad.png)

  

## 实现find

  

借鉴ls.c代码实现逐层遍历文件，如果是文件夹则进行递归，遇到 “.”“..”跳过。

  

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1779631760260-7f3ba957-e738-4074-a822-831815c54001.png)

```
#include "kernel/types.h"
#include "kernel/stat.h"
#include "kernel/fs.h"
#include "user/user.h"

//提取搜索的文件名，p指针回退到字符串/后的第一个字母，返回p指针
char *
fmtname(char *path)
{
    char *p;
    // Find first character after last slash.
    for (p = path + strlen(path); p >= path && *p != '/'; p--)
        ;
    p++;
    // Return blank-padded name.
    return p;
}

void find(char *path, char *find_file_name)
{
    char buf[512], *p;
    int fd;
    struct dirent de;
    struct stat st;

    if ((fd = open(path, 0)) < 0)
    {
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }

    if (fstat(fd, &st) < 0)
    {
        fprintf(2, "find: cannot stat %s\n", path);
        close(fd);
        return;
    }

    switch (st.type)
    {
        case T_FILE:
            if (strcmp(fmtname(path), find_file_name) == 0)
            {
                printf("%s\n", path);
            }
            break;

        case T_DIR:

            if (strlen(path) + 1 + DIRSIZ + 1 > sizeof buf)
            {
                printf("find: path too long\n");
                break;
            }
            //进行路径拼接
            strcpy(buf, path);
            p = buf + strlen(buf);
            *p++ = '/';

            while (read(fd, &de, sizeof(de)) == sizeof(de))
            {
                if (de.inum == 0)
                    continue;
                //进行路径拼接
                memmove(p, de.name, DIRSIZ);
                p[DIRSIZ] = 0;

                if (stat(buf, &st) < 0)
                {
                    printf("find: cannot stat %s\n", buf);
                    continue;
                }

                //如果是文件夹进行递归
                if (st.type == T_DIR)
                {
                    printf("%s\n", fmtname(buf));
                    if (strcmp(fmtname(buf), ".") == 0)
                    {
                        continue;
                    }

                    if (strcmp(fmtname(buf), "..") == 0)
                    {
                        continue;
                    }

                    find(buf, find_file_name);
                    continue;
                }

                if (strcmp(fmtname(buf), find_file_name) == 0)
                {
                    printf("%s\n", buf);
                }
            }
            break;
    }
    close(fd);
}

int main(int argc, char *argv[])
{
    if (argc < 3)
    {
        fprintf(2, "ERROR: argc number too less\n");
        exit(1);
    }

    for (int i = 2; i < argc; i++)
    {
        find(argv[1], argv[i]);
    }
    exit(0);
}
```

  

  

## 实现xargs

管道重定向，将第一个进程的标准输出重定向到第二个进程的标准输入

xargs读入整行作为新命令的参数，一直读取直到遇见\n。值得注意的是，buf的最后一位以\0结尾，不能以\n结尾，否则在进行文件匹配时会出错。

fork+exec的使用很常见，子进程执行新命令，父进程等待。exec（char * 命令文件名，char ** 参数列表），我们只需要拼接参数列表即可。

  

错误：exec的参数列表有误

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1780905622818-83e70e26-311e-4ae3-9719-3b37a97cabb1.png)

  

可以和其他命令配合使用

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1780911820982-8808f180-c3a7-486d-ba99-1931b83d44c0.png)

  

```
#include "kernel/types.h"
#include "kernel/param.h"
#include "user/user.h"

#define STDIN_FD 0
int main(int argc, char *argv[])
{

    char *exec_argv[MAXARG];
    if (argc + 2 > MAXARG)
    {
        printf("Xargs: args is too much!\n");
        exit(2);
    }
    for (int i = 0; i < argc - 1; i++)
    {
        exec_argv[i] = argv[i + 1];
    }

    char buf[512];
    char p = 0;
    int i = 1;
    while (1)
    {
        // 读入一整行，每次读一个字母，直到'\n' ，以\0结尾
        while (i < 511)
        {
            
            int ret = read(STDIN_FD, &p, 1);
            if (ret == 0)
            {
                //程序结束的条件，读取完毕
                exit(0);
            }

            if (p == '\n')
            {
                buf[i - 1] = '\0';
                break;
            }
            else
            {
                buf[i - 1] = p;
            }
            i++;
        }

        if (fork() == 0)
        {
            // 子进程执行附加命令

            // 重要: 更改命名的参数行,实现xargs的功能
            exec_argv[argc - 1] = buf;
            exec_argv[argc] = 0;

            exec(argv[1], exec_argv);
            printf("Exec is failed!\n");
            exit(1);
        }
        else
        {
            // 父进程等待子进程退出
            wait(0);
        }

        // 清空数组
        memset(buf, 0, sizeof(buf));
        p = 0;
        i = 1;
    }
    return 0;
}
```