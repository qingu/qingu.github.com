---
layout: post
title: "大型机使用技巧汇总"
date: 2013-03-03 21:17
comments: true
categories: [Linux, AIX] 
---
### 0. 前言 ###
因为转模式的缘故，需要和大型机打交道，目前在IBM的AIX系统和神威的Linux系统之间来回倒腾，经常使用大型机如果掌握一些使用技巧，工作效率定会大大提高。本文用于持续总结使用大型机的一些技巧，与大家分享。

<!-- more -->

### 1. 登录 ###
Windows系统下可以用一些ssh登录软件登录大型机，如 `Xmanager` 、`SecureCRT` 等。

Linux系统下直接用终端 + ssh命令。

```
	ssh user_name@host_name
```

host_name一般是大型机IP地址。稍后会提示输入密码，注意输入的密码不可见。

但是这样每次登录都要输入命令和密码，挺烦人的。可以考虑写个脚本自动登录，这里用到工具 `expect` 。
确定自己系统里是否有 expect ：`which expect`。如果没有就装下。

脚本内容：

```
	#!/usr/bin/expect -f

	set user user_name
	set host host_name
	set password my_password
	set timeout -1

	spawn ssh  $user@$host
	expect {
		"*yes/no" { send "yes\r"; exp_continue}
		"*password:" { send "$password\r" }
	}
	interact
```	

将上面代码保存为 go ，设置脚本权限为 755， `chmod 755 go` 。以后直接执行脚本就行。

**Linux下还有一种省去输密码的方法：公钥登录**

在自己电脑终端上输入：

```
    ssh-keygen 
```

出现一系列提示，可以一路回车。结束后，在～/.ssh/下生成两个文件：id_rsa.pub 和 id_rsa。前者是你的公钥，后者是私钥。

再输入以下命令将公钥传到大型机上：

```
    ssh-copy-id user_name@host_name
```

以后你就可以直接输入 `ssh user_name@host_name` 登录。如果你想更省事，再自定义一个函数放到 ~/.bashrc中：

```
    sw(){
      ssh user_name@host_name
    }
```

以后直接在终端输入命令 `sw` 登录，是不是很方便。



scp 替代 ftp 传文件：
	scp [option] user@host1:file1 user@host2:file2

示例：

  * 复制本地文件到远程大型机上

	scp localfile user_name@host_name:/home/mydirctory

  * 复制远程大型机上文件到本地

    scp user_name@host_name:~/.bashrc ~

注：传输目录时加选项 `-r`     

### 2. 设置登录路径 ###
这个设置不一定所有人需要。我是因为用的老师的大型机账号，在里面建了个自己的目录，我的所有工作就是在这个目录里完成。
每次登录到大型机后，要 cd 很长的路径才能到达我的工作目录。为了减少工作量，可以利用 `*nix` 系统的自定义函数功能。

```
	cdgu() {
		cd /path/to/my_working_directory
		ls
	}
```

在 Linxu 系统 `~/.bashrc` 或 AIX 系统 `~/.profile` 添加类似以上的代码，保存后 `source ~/.bashrc` 或 `source ~/.profile` 使得修改立即生效。然后在终端里输入函数名 `cdgu` ，看看是不是直接进入自己的工作目录了。

注意起的函数名不要和系统已有命令冲突，可以事先用 `which 函数名` 测试。

### 3. Shell快捷健 ###
首先确定你的登录shell是哪种，用命令 `echo $SHELL` 查看。记住多少个快捷键为宜呢，个人认为 `Less is More` ，够用就好。

BASH Shell快捷键可以参考 [LinuxTOY][1] 和 [维基百科][2]这篇文章总结的。我将第一篇文章网页保存为pdf版本了，可以打印出来放在电脑旁边，没事看看。地址：[百度云盘下载][3] 。

AIX下默认ksh，功能没有bash强大。不过也有一些快捷键。

以下列出一些快捷键，bash 和 ksh 都可以使用（亲测可用）。

  * Ctrl + a ：移到命令行首
  * Ctrl + e ：移到命令行尾
  * Ctrl + f ：按字符前移（右向）
  * Ctrl + b ：按字符后移（左向）
  * Ctrl + u ：从光标处删除至命令行首 （ksh中表现为删除整行）
  * Ctrl + k ：从光标处删除至命令行尾
  * Ctrl + w ：从光标处删除至字首 （ksh表现为从光标处删除至命令行首）
  * Ctrl + d ：删除光标处的字符
  * Ctrl + h ：删除光标前的字符
  * Ctrl + p：历史中的上一条命令
  * Ctrl + n：历史中的下一条命令


[1]: http://linuxtoy.org/archives/bash-shortcuts.html "LinuxTOY"
[2]: http://en.wikipedia.org/wiki/Bash_(Unix_shell)#Keyboard_shortcuts "维基百科"
[3]: http://pan.baidu.com/share/link?shareid=431631&uk=1392639653 "百度云盘下载"

### 4. 常用命令 ###
以下列出一些常用命令：

  * cd ~ : 回到home目录
  * cd - : 回到上一次工作目录（相当于撤销上一次cd命令）
  * ls -l : 列出当前目录下所有文件详细属性（不包含隐藏文件，需要的话加 -a)
  * ll : 'ls -l --color=tty'的别名（快捷方式），目录和链接带色彩，以示区别
  * chmod a+x file : 设置文件权限为所有人可执行（u-用户，g-所属组，o-其他人,a-所有人| r-读权限，w-写权限，x-可执行权限）
  * grep : 文件搜索
  * tailf : 监控文件，常用于监控模式标准输出/错误文件的输出，例如  tailf std.error.0000

### 5. 编译问题 ###
  * AIX系统的xlf90编译器默认不能编译后缀名为 .f90的程序（真扯），需要加上编译选项 `-qsuffix=f=f90` 。如果经常编译f90程序的话，可考虑在 ~/.kshrc 中加上 
  
```    
    alias xx='xlf90 -qsuffix'
```


### 6. 大型机作业调度管理工具 ###

#### 神威-SLURM ####
神威使用 SLURM 管理作业提交、删除工作。这里的作业指的是并行作业，如果要执行串行程序，不需要以下这些命令。不过转模式一般需要同时利用多个cpu节点执行。

以下命令详细用法可以使用 `命令 --help` 或 `命令 --usage` 查看。

* srun ： 提交作业
  
常用选项：

  -n ntasks 利用ntasks个进程数执行程序

  -p partition 指定程序运行所在节点池

  -J jobname 指定作业名，这个是必须要有的

示例：

  执行grapes.exe程序，使用32个进程数，节点池选择NORMAL7

```
    srun -n 32 -p NORMAL7 -J GRAPES ./grapes.exe
```

* squeue : 查询提交作业状态

  常用选项：

  -u user_name 查询指定用户名的作业状态

只执行 squeue 命令，将输出大型机上所有用户提交的作业状态情况。输出一般分成以下几列

```
     JOBID PARTITION     NAME     USER  ST       TIME  NODES NODELIST(REASON)
```

各列含义：

  JOBID : 作业号，相当于该作业的”身份证号“

  PARTITION : 所在节点池

  NAME : 程序名称

  USER : 用户名

  ST : 作业状态， R-Runing（正在运行），PD-PenDing（资源不够，等待），CG-COMPLETING(作业正在完成过程中)， CA-CANCELLED（作业被人为取消掉了，一般是管理员干的）

  TIME ：作业执行时间

  NODES ：程序占的节点数

  NODELIST : 详细的节点清单

示例：

一般只关心自己作业的情况

```
      squeue -u user
```

* scancel : 取消作业

普通用户只能取消掉自己的作业，如果 scancel 别人作业，输出错误信息（因为你不是管理员上帝）

常用选项：

  -u, -user=user_name 取消掉该用户的提交的作业

  -n, --name=job_name 取消掉叫 job_name 的作业

示例：

一般常用squeue查询自己作业job_id，然后杀掉作业(可同时kill多个作业) 

```
    squeue -u user_name
    scancel job_id
```

如果确定是你一个人用这个账号，且想kill掉所有已提交的作业，可以

```
    squeue -u user_name
```

* sinfo : 查询节点池及节点状态，查看哪个节点池有空闲资源可以使用

* sbatch ： 执行批处理脚本，脚本里面包含一些大型机资源环境变量设置及srun提交作业命令

#### IBM-loadleveler ####

以下简单介绍常用几个命令，具体可参考[官方文档][4]

[4]: http://publib.boulder.ibm.com/infocenter/clresctr/vxrx/index.jsp?topic=%2Fcom.ibm.cluster.loadl.v5r1.load500.doc%2Fam2cr_conq.htm

* llrun : 提交mpi并行作业或串行交互作业

常用选项：

  -k keyword_list 使用空格分隔的 keyword=value形式指定作业配置参数

NOTE: 这个命令好像被IBM大型机禁止掉了，只能用llsubmit提交作业了，orz

* llq : 查询作业状态

常用选项：

  -u user_name 查询指定用户的作业状态， 只输入llq命令则列出所有用户作业情况

示例：

```
     llq -u user_name
```

* llcancel : 取消作业，一般后面跟作业号

* llstatus : 查询大型机资源状态

* llsubmit : 提交作业，后面跟命令文件

>命令文件应该包含对作业进行描述、对版本历史和变化进行跟踪的注释。每一行开头的 # 号（#）表示该行是一个注释。如果 # 之后是 @ 符号（@），就表示这是一个关键字。关键字用来指定与作业相关的输入、输出和错误文件，以及作业类。要查看管理员已经定义了哪些类，请使用 llclass 命令。

示例：

```
     llsubmit job.cmd
```

job.cmd 示例：

```
    #@ shell = /bin/ksh      #指定脚本解释器
    #@ job_type = parallel   #作业类型设为并行的，需要设置节点数
    #@ network.MPI = csss,shared,US   # MPI并行程序使用内部共享节点通信
    #@ output = $(host).$(jobid).out  # 标准输出重定向新文件                                 
    #@ error = $(host).$(jobid).err   # 标准错误重定向到新文件
    #@ wall_clock_limit = 30:00       # 限制程序只能运行 30 分钟

    #使用2个节点，每个节点上32个进程，共64个进程执行并行作业
    #@ tasks_per_node = 32            
    #@ node = 2

    #@ queue          #排队，等待资源到位后执行。没有这条语句，不会建立作业
    
    pwd               #显示当前工作目录
    echo $LOADL_PROCESSOR_LIST      #显示分配给该作业的节点
    export MP_SHARED_MEMORY=yes     #让MPI使用共享内存通信
    
    poe a.out          #运行程序
```

#### 类比两种作业调度管理器 ####

功能          | SW-SLURM       | IBM-Loadleveler
-------------|----------------|------------------:
提交作业      | srun           | llrun(禁掉了)
查询作业      | squeue         | llq
取消作业      | scancel        | llcancel 
脚本提交作业   | sbatch         | llsubmit
查询节点资源   | sinfo          | llstatus



### Update Log ###

* 2013-3-8 增加大型机作业调度管理器介绍