---
title:  'nohup和&的使用说明'
date:  2019-6-14
tags: linux
categories: [操作系统,linux]
---

### 1.nohup和&后台运行

#### 1.1 nohup

功能：不挂断运行命令

语法：nohup Command [ Arg … ] [　& ]

​		无论是否将 nohup 命令的输出重定向到终端，输出都将附加到当前目录的 nohup.out 文件中。

　　如果当前目录的 nohup.out 文件不可写，输出重定向到 $HOME/nohup.out 文件中。

　　如果没有文件能创建或打开以用于追加，那么 Command 参数指定的命令不可调用。

退出状态：该命令返回下列出口值： 　　

　　126： 可以查找但不能调用 Command 参数指定的命令。 　　

　　127： nohup 命令发生错误或不能查找由 Command 参数指定的命令。 　　

　　否则，nohup 命令的退出状态是 Command 参数指定命令的退出状态。



#### 1.2 &

功能：命令在后台运行，功能与`Ctrl+z`相同，一般配合nohup一起使用 

eg：`nohup ~/user/test.sh>output.log 2>&1 &`

命令详解：

- `nohup ~/user/test.sh>output.log` 不挂断运行`test.sh`，输出结果重定向到当前目录的`output.log`

- 最后的`&` 表示后台运行

- `2>&1` `0`表示键盘输入，`1`屏幕输出即标准输出，`2`表示错误输出。其中`2>&1`表示将错误信息重定向到标准输出

  试想一下，如果`2>&1`指将错误信息重定向到标准输出，那`2>1`指什么？

  分别尝试`2>1`，`2>&1`

```shell
$ ls >outfile
$ cat outlog 
outlog
test.sh
$ ls xxx>outfile
ls: cannot access xxx: No such file or directory
$ cat outfile
 (这里是空)
$ ls xxx 2>1
$ cat 1(可以看出，将错误信息重定向到文件1里面了)
ls: cannot access xxx: No such file or directory
```

​	也就是说`2>1`会将错误信息重定向到文件1里面，所以`2>&1`中的`&1`指标准输出

### 2. 查看后台运行的进程

#### 2.1 jobs的使用

>  **jobs命令**用于显示Linux中的任务列表及任务状态，包括后台运行的任务。该命令可以显示任务号及其对应的进程号。其中，任务号是以普通用户的角度进行的，而进程号则是从系统管理员的角度来看的。一个任务可以对应于一个或者多个进程号。

语法： jobs(选项)(参数)

选项

> -l：显示进程号；
> -p：仅任务对应的显示进程号；
> -n：显示任务状态的变化；
> -r：仅输出运行状态（running）的任务；
> -s：仅输出停止状态（stoped）的任务。

常用命令： jobs  -l

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2019-5-30/jobs_1.png)

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2019-5-30/jobs_2.png)

其中，输出信息的第一列表示任务编号，第二列表示任务所对应的进程号，第三列表示任务的运行状态，第四列表示启动任务的命令。

缺点：jobs命令只看当前终端生效的，关闭终端后，在另一个终端jobs已经无法看到后台跑得程序了，此时利用ps（进程查看命令）

#### 2.2  ps的使用

> **ps命令**用于报告当前系统的进程状态。可以搭配[kill](http://man.linuxde.net/kill)指令随时中断、删除不必要的程序。ps命令是最基本同时也是非常强大的进程查看命令，使用该命令可以确定有哪些进程正在运行和运行的状态、进程是否结束、进程有没有僵死、哪些进程占用了过多的资源等等，总之大部分信息都是可以通过执行该命令得到的。

常用命令：`ps -aux`

> a:显示所有程序 
> u:以用户为主的格式来显示 
> x:显示所有程序，不以终端机来区分

通常与`nohup &`配合使用，用于查看后台进程ID 配合 kill命令杀掉程序

常用命令：`ps -aux|grep test.sh| grep -v grep` 

注：`grep -v grep` 用grep -v参数可以将grep命令排除掉

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/2019-5-30/ps_1.png)

