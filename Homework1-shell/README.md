[Homework1-shell](https://pdos.csail.mit.edu/6.828/2014/homework/xv6-shell.html)

## 作业简介
这次作业只是简单的添加代码，让`sh.c`支持执行命令、I/O重定向、管道

## 执行简单的命令
让程序支持执行简单的命令，如`ls`，如果不支持环境变量，则要输入`/bin/ls`

程序已经能帮我们解析好了输入的命令，我们只需要在函数`runcmd`中的`cast ’ ‘`里添加对命令的执行，输入的`type`为`’ ‘`说明输入的是命令。

我们只要简单的调用`execv`系统调用即可：
```c
char *PATH[] = {"/bin/", "/usr/bin/", "",};
int npath = sizeof(PATH) / sizeof(char*);
int i = 0;
for(; i < npath; ++i) {
    char path[128] = {0};
    strcpy(path, PATH[i]);
    strcat(path, ecmd->argv[0]);
    execv(path, ecmd->argv);  // 只要不出错就不会返回
}
fprintf(stderr, "command not found\n");
```
编译后执行：
```shell
$ ./sh
6.828$ ls
lab       pcasm-book.pdf  sh.c  t.sh   xv6-chinese.pdf
Makefile  sh              tags  x.txt
```
## I/O重定向
要支持I/O重定向即支持`<`和`>`，这里涉及文件的操作，要用到`open`和`close`系统调用，还要理解什么是文件描述符，以及`dup2`系统调用的使用

`dup2(oldfd, newfd)`会复制`oldfd`到`newfd`，这样我们就可利用这个特性，标准输入和标准输出的文件描述符分别为0和1，
那么我们在打开文件时，把标准输入或标准输出(取决于是`<`还是`>`)关掉(`dup2`会先关闭`newfd`)，
然后再调用`dup2(oldfd, newfd)`则复制fd为0或1，这样命令的输入或输出就会使用文件的内容

```c
int fd;
if(rcmd->type == '>')
    fd = open(rcmd->file, rcmd->mode, S_IRWXU);
else
    fd = open(rcmd->file, rcmd->mode);
if (fd < 0)
{
    fprintf(stderr, "open error\n");
}
if (dup2(fd, rcmd->fd) == -1)
{
    perror("dup2");
    exit(EXIT_FAILURE);
}
runcmd(rcmd->cmd);
```
编译后执行：
```shell
$ ./sh
6.828$ echo "6.828 is cool" > x.txt
6.828$ cat x.txt
"6.828 is cool"
```

## 执行管道
管道的执行类似之前的重定向，不同的是管道不用操作文件(这里的文件指的是普通的文件)

用`fork`两次生成的两个子进程执行管道左边和右边的命令
```c
if (pipe(p) == -1) {
    perror("pipe");
    exit(EXIT_FAILURE);
}
if(fork1() == 0) {
    close(p[0]);
    if(dup2(p[1], STDOUT_FILENO) == -1) {
        perror("dup2");
        exit(EXIT_FAILURE);
    }
    close(p[1]);
    runcmd(pcmd->left);
}
if (fork1() == 0) {
    close(p[1]);
    if(dup2(p[0], STDIN_FILENO) == -1) {
        perror("dup2");
        exit(EXIT_FAILURE);
    }
    close(p[0]);
    runcmd(pcmd->right);
}

close(p[0]);
close(p[1]);
wait(&r);
wait(&r);
```

## 最终测试
```shell
$ sh < t.sh
     10      10      69
     10      10      69
```
