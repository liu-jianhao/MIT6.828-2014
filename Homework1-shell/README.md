[Homework1-shell](https://pdos.csail.mit.edu/6.828/2014/homework/xv6-shell.html)

## 作业简介
这次作业只是简单的添加代码，让`sh.c`支持执行命令、I/O重定向、管道

## 执行简单的命令
让程序支持执行简单的命令，如`ls`，如果不支持环境变量，则要输入`/bin/ls`

程序已经能帮我们解析好了输入的命令，我们只需要在函数`runcmd`中的`cast ’ ‘`里添加对命令的执行，输入的`type`为`’ ‘`说明输入的是命令。

我们只要简单的调用`execv`系统调用即可：
```c
char PATH[] = "/bin/";
char path[128];
strcpy(path, PATH);
strcat(path, ecmd->argv[0]);
execv(path, ecmd->argv);
```
编译后执行：
```shell
$ ./sh
6.828$ ls
lab       pcasm-book.pdf  sh.c  t.sh   xv6-chinese.pdf
Makefile  sh              tags  x.txt
```
## I/O重定向
要支持I/O重定向即支持`<`和`>`，这里涉及文件的操作，要用到`open`和`close`系统调用，还要理解什么是文件描述符，以及`dup`系统调用的使用

`dup(fd)`会复制`fd`到最小的文件描述符，这样我们就可利用这个特性，标准输入和标准输出的文件描述符分别为0和1，
那么我们在打开文件时，把标准输入或标准输出(取决于是`<`还是`>`)关掉，然后再调用`dup(fd)`则复制fd为0或1，
这样命令的输入或输出就会使用文件的内容

```c
int mask = S_IRUSR | S_IWUSR;
int fd = open(rcmd->file, rcmd->mode, mask);
close(rcmd->fd);
dup(fd);
runcmd(rcmd->cmd);
```
编译后执行：
```shell
$ ./sh
6.828$ echo "6.828 is cool" > x.txt
6.828$ cat x.txt
"6.828 is cool"
```
