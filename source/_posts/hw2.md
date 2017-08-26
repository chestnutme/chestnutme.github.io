title: hw2-shell
date: 2017/04/16
tags:
	- os
	- xv6

---
# hw2 shell
## 目标
hw2主要是根据xv6,学习linux的系统调用,实现一个简化版的shell.
目标可以分解为:
> * 阅读源代码
> * 添加执行命令的功能
> * 添加IO重定向
> * 添加管道功能

## part1: command
### 命令的种类
#### 命令参数个数
> 首先,一个命令的参数总数是限制的,xv6 shell规定一个命令的参数最多只能有10个.
`#define MAXARGS 10`

<!-- more -->

#### 命令的基类
> 所有的命令都必须有一个种类。这里声明一个`struct cmd`的作用是为了用来指向特定的某个cmd的类型。注意后面声明的`struct`都是在一个结构的开头int指明了一个type.后面可以把这个结构体在`struct *cmd`之间进行转换.这里采用了C++的面向对象的思想，struct cmd是execcmd, redircmd, pipecmd的基类。后面要处理各种命令的时候，都是使用的是`struct *cmd`类型的指针。
实际结构如下：
```c
pipecmd {
	- left = pipecmd {
				   - left: a
				   - right: b
			  }
   - right = execcmd {
   				   - cmd: c
       	  }
}

// All commands have at least a type. Have looked at the type, the code
// typically casts the *cmd to some specific cmd type.
struct cmd {
  int type;          //  ' ' (exec), | (pipe), '<' or '>' for redirection
};
```
#### execcmd
> 可以直接执行的命令。也就是说不存在IO重定向，管道等情况。所以这里的type注释写的是空格' '。
```c
struct execcmd {
  int type;              // ' '
  char *argv[MAXARGS];   // arguments to the command to be exec-ed
};
```
#### redircmd
重定向输入输出的命令。这里记录下的参数包含了：
> * 命令类型，重定向输入还是重定向输出
> * 要执行的命令
> * 输入或者输出文件
> * 打开文件的模式
> * 文件描述符
```c
struct redircmd {
  int type;          // < or >
  struct cmd *cmd;   // the command to be run (e.g., an execcmd)
  char *file;        // the input/output file
  int mode;          // the mode to open the file with
  int fd;            // the file descriptor number to use for the file
};
```
#### pipecmd
pipecmd执行是实现重定向输出的情况.
```c
struct pipecmd {
  int type;          // |
  struct cmd *left;  // left side of pipe
  struct cmd *right; // right side of pipe
};
```

### 命令的执行
命令的执行则主要是在runcmd函数里面进行。runcmd根据命令的类型来决定执行何种操作:
> * 如果是未定义的命令类型，直接报错，并退出。
> * 如果是execcmd类型，需要添加执行代码。实际上作业需要完成的代码也是在这里完成。
> * '>'和'<'表示输入输出重定向。
> * '|'管道操作，就是把前面一个程序的输出，重定向为后面一个程序的输入。
上面这些功能，都处于未完成的状态。都还需要添加代码。
```c
// Execute cmd.  Never returns.
void runcmd(struct cmd *cmd)
{
  int p[2], r;
  struct execcmd *ecmd;
  struct pipecmd *pcmd;
  struct redircmd *rcmd;
  if(cmd == 0)
    exit(0);
  switch(cmd->type){
  default:
    fprintf(stderr, "unknown runcmd\n");
    exit(-1);
  case ' ':
    ecmd = (struct execcmd*)cmd;
    if(ecmd->argv[0] == 0)
      exit(0);
    fprintf(stderr, "exec not implemented\n");
    // Your code here ...
    break;
  case '>':
  case '<':
    rcmd = (struct redircmd*)cmd;
    fprintf(stderr, "redir not implemented\n");
    // Your code here ...
    runcmd(rcmd->cmd);
    break;
  case '|':
    pcmd = (struct pipecmd*)cmd;
    fprintf(stderr, "pipe not implemented\n");
    // Your code here ...
    break;
  }
  exit(0);
}
```

### 命令的获取
从终端中读取需要执行的命令。getcmd的三个任务:
> * 判断是不是终端，如果是，那么输出"6.828$"提标符。
> * 清空内存，然后读取输入。
> * 判断是不是读入了EOF结束标志。如果是，返回-1。
```c
int getcmd(char *buf, int nbuf)
{
  if (isatty(fileno(stdin)))
    fprintf(stdout, "6.828$ ");
  memset(buf, 0, nbuf);
  fgets(buf, nbuf, stdin);
  if(buf[0] == 0) // EOF
    return -1;
  return 0;
}
```

### 主程序
主程序比较简单,分为两部分：
> * 查看是不是cd命令，如果是，那么调用chdir系统调用。
> * 否则调用fork，利用子进程rumcmd。
```c
int main(void)
{
  static char buf[100];
  int fd, r;
  // Read and run input commands.
  while(getcmd(buf, sizeof(buf)) >= 0){
    if(buf[0] == 'c' && buf[1] == 'd' && buf[2] == ' '){
      // Clumsy but will have to do for now.
      // Chdir has no effect on the parent if run in the child.
      buf[strlen(buf)-1] = 0;  // chop \n
      if(chdir(buf+3) < 0)
        fprintf(stderr, "cannot cd %s\n", buf+3);
      continue;
    }
    if(fork1() == 0)
      runcmd(parsecmd(buf));
    wait(&r);
  }
  exit(0);
}
```

### homework
代码实现如下:
```c
void runcmd(struct cmd *cmd)
{
  int p[2], r;
  struct execcmd *ecmd;
  struct pipecmd *pcmd;
  struct redircmd *rcmd;

  char * path;
  path = malloc(MAXPATH * sizeof(char*));

  if(cmd == 0)
    exit(0);

  switch(cmd->type){
  default:
    fprintf(stderr, "unknown runcmd\n");
    exit(-1);

  case ' ':
    ecmd = (struct execcmd*)cmd;
    if(ecmd->argv[0] == 0)
      exit(0);

    // 任务 1: 实现简单指令的调用与缺省路径
    strcat(path, ecmd->argv[0]);
        // My code here...
    strcpy(path, "");
    strcat(path, ecmd->argv[0]);
    if(!access(path, F_OK)) {
      execv(path, ecmd->argv);
      break;
    }

    strcpy(path, "/bin/");
    strcat(path, ecmd->argv[0]);
    if(!access(path, F_OK)) {
      execv(path, ecmd->argv);
      break;
    }

    strcpy(path, "/usr/bin/");
    strcat(path, ecmd->argv[0]);
    if(!access(path, F_OK)) {
      execv(path, ecmd->argv);
      break;
    }

    fprintf(stderr, "exec not implemented\n");
    break;

  // 任务 2: 实现I/O重定向  
  case '>':
  case '<':
    rcmd = (struct redircmd*)cmd;

    // sub 1. 关闭被重定向覆盖的文件描述符
    close(rcmd->fd);

    // sub 2. 处理系统调用失败的情况, 注意文件权限(0777)为八进制数保证符合约定
    if (open(rcmd->flie, rcmd->mode, 0777) < 0) {
      fprintf(stderr, "open file %s failed.", rcmd->file);
      exit(0);
    }

    runcmd(rcmd->cmd);
    break;

  // 任务 3. 实现管道功能
  case '|':
    pcmd = (struct pipecmd*)cmd;

    // sub 1. 调用系统接口pipe()创建一个管道, 首先检查失败情况
    if(pipe(p) < 0){
        fprintf(stderr, "create pipe failed!\n");
        exit(0);
    }

    // sub 2. 根据管道的语义, 创建子进程来处理两个过程
    if((fork() == 0)) {
        close(STDOUT_FILENO);
        dup(p[1]);
        close(p[0]);
        close(p[1]);
        runcmd(pcmd->left);
    } else {
        close(0);
        dup(p[0]);
        close(p[0]);
        close(p[1]);
        runcmd(pcmd->right);
    }
    wait(&r);
    break;
  }
  exit(0);
}
```

## 总结
通过这个调用图可以看出命令处理的层次结构。
![call graph](http://img.blog.csdn.net/20150419132244069)
> * 首先根据|管道来切分块。比如{block_a} | {block_b} | {block_c} | {block_d}。处理逻辑不是通过for循环来处理，而是通过递归调用来解决。每一个block的处理都是通过parseexec来完成。
```c
struct cmd*
parsepipe(char **ps, char *es)
{
  struct cmd *cmd;
  cmd = parseexec(ps, es);
  if(peek(ps, es, "|")){
    gettoken(ps, es, 0, 0);
    cmd = pipecmd(cmd, parsepipe(ps, es));
  }
  return cmd;
}
```
> * parseexec处理的时候，代码里面是同时处理了execcmd, redircmd这两种命令。也就是说，如果从C++的角度来看，类的继承关系就是：cmd->execcmd->redircmd。每个block的处理方式如下：
```c
argc = 0;
ret = parseredirs(ret, ps, es);
while(!peek(ps, es, "|")){
  if((tok=gettoken(ps, es, &q, &eq)) == 0)
    break;
  if(tok != 'a') {
    fprintf(stderr, "syntax error\n");
    exit(-1);
  }
  cmd->argv[argc] = mkcopy(q, eq);
  argc++;
  if(argc >= MAXARGS) {
    fprintf(stderr, "too many args\n");
    exit(-1);
  }
  ret = parseredirs(ret, ps, es);
}
cmd->argv[argc] = 0;
```
> * 如果没有IO重定向，那么parseredirs函数相当于空函数。没有任何作用。
> * parseredirs的处理就比较简单，就只负责处理execcmd的<input.txt或者>output.txt部分。
```c
while(peek(ps, es, "<>")){
  tok = gettoken(ps, es, 0, 0);
  if(gettoken(ps, es, &q, &eq) != 'a') {
    fprintf(stderr, "missing file for redirection\n");
    exit(-1);
  }
  switch(tok){
  case '<':
    cmd = redircmd(cmd, mkcopy(q, eq), '<');
    break;
  case '>':
    cmd = redircmd(cmd, mkcopy(q, eq), '>');
    break;
  }
}
```

## 参考链接
1. [hw2 shell](https://pdos.csail.mit.edu/6.828/2016/homework/xv6-shell.html)
2. [xv6 system call](https://pdos.csail.mit.edu/6.828/2016/xv6/book-rev9.pdf)
3. [6.828 shell](https://pdos.csail.mit.edu/6.828/2016/homework/sh.c)
