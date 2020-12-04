---
title: SSH 跳板机和隧道
date: 2019-11-15
tags:
  - 端口转发
  - SSH
  - 代理
  - 跳板机
---

> The Secure Shell (SSH) is a protocol for secure remote login and other secure network services over an insecure network.

SSH 是在不安全网络上建立安全远程登陆和其他安全网络服务的协议。这次我们聊聊SSH一些代理相关的功能和应用场景。

<!--more-->

## 简介

在现实场景中，由于网络上的限制，一些服务往往不能够直接访问。利用SSH安全协议，我们能够安全地突破一些限制。下面是一些常见的应用场景。

场景一：本地机器想通过SSH登陆公司内网的机器，但由于安全策略的限制无法直接登陆。

场景二：本地浏览器想直接访问远程机器的服务程序，但是远程机器上服务程序对应的端口是不开放的。

场景三：我们希望远程机器能够代理我们的请求，访问一些本地机器无法访问的地址。

## SSH跳板机

面对场景一，一般需要代理机器。它允许被外部机器访问，同时又能够访问公司内网的机器。这种场景下的代理机器也被称作跳板机。
手动利用代理机器的方法是就是连续两次SSH登陆。先使用`ssh host1`登陆host1（跳板机）,在使用`ssh host2`从host1登陆host2（目标机器）。当然每次都这么操作会非常麻烦。能不能够直接运行`ssh host2`一个命令就实现上面的功能呢？

如果你使用的是OPENSSH, 那么通过配置文件中的配置项ProxyCommand实现。

在以前的[博客](https://fzp.github.io/2018/05/10/SSH-config.html)中已经介绍过SSH的配置文件了，不了解的可以看一下。

```SSH
Host host1
    User ubuntu
    HostName 192.168.0.1

Host host2
    User ubuntu
    HostName 192.168.0.2
    ProxyCommand ssh host1 -W %h:%p
```

通过上面的配置就能实现我们想要的功能。主要关注上面的ProxyCommand配置，其他的配置都只是在定义机器的IP和用户的名称。

> ProxyCommand: Specifies the command to use to connect to the server.

ProxyCommand用于指定连接服务器的命令。如果不指定ProxyCommand， SSH会自己建立到目标机器的连接。指定后，会使用指定的命令来建立连接，SSH直接使用该连接。所以原则上这条命令只需要能够从标准输入读数据，从标准输出写数据就可以了。

再看看具体的命令。

```bash
ssh host1 -W %h:%p
```

这条命令通过SSH连接到host1,`-W`选项将客户端过来的标准输入和输出转发到指定和host(%h)和port(%p)。 %h和%p是SSH配置文件中的变量。表示当前Host配置下的HostName和Port。在这里host2的ip和端口。

上面的命令理解起来还是挺麻烦的。所以在openSSH 7.3后引入了ProxyJump配置项，直接实现跳转功能。只需要将ProxyCommand换成`ProxyJump host1`.

## SSH隧道

对于场景二和三，这里介绍SSH隧道方法。隧道(Tunneling)是将数据从一个网络移动到另外一个网络的方法。SSH隧道就是用SSH协议形成安全隧道移动网络数据，而不是用于登陆远程服务器。

### 本地端口转发

在场景二中，假设远程机器host3的22端口是开放的，并且host3上部署了一个网页应用，绑定到8080端口。现在我们的目的是在浏览器地址栏输入localhost就能够访问网页应用。

在SSH客户端所在机器下面输入以下命令建立隧道：

```
ssh -NfL 80:localhost:8080 host3
```

`-L`参数命令格式如下。

```
-L [bind_address:]port:host:hostport
```

带`-L`参数的命令称作本地端口转发。它的目的是将对本地端口的访问转发到远程服务器的端口上。所有通过port的流量都会转发到host:hostport上。

注意，在我们具体的命令里，host参数的具体值是localhost。这个localhost是远程服务器的localhost，而我们在浏览器里输入的localhost是本地机器的localhost。两者是不一样的，千万不要混淆。

另外，这条命令中除了端口外还设计了两个地址，bind_address和host。为什么还需要两个地址？不应该默认就是本地机器和远程机器的地址吗？

事实上，这个命令设计到4个机器。浏览器所在的本地机器，SSH客户端，SSH服务器端，还有网页应用所在的远程服务器。bind_address对应的是浏览器所在机器，port是SSH客户端的端口，host和hostport是远程服务器的地址和端口。本地机器和SSH客户端并不一定在同一台机器上。SSH服务器和远程服务器同理。如果本地机器和SSH客户端不是同一台机器。需要指定`-g`参数保证本地机器能够使用SSH客户端的端口进行转发。

`-f`参数是指定SSH客户端以后台方式运行。

`-N`参数是不请求一个shell界面，不执行远程命令以保证SSH服务器的安全。这个参数只是在端口转发时实用。

### 远程端口转发

远程端口转发和本地端口转发时类似的。它们的不同点在于转发数据的方向不同。

本地端口转发是转发SSH客户端的数据到SSH服务器端，远程端口转发则是反过来。

同样一个访问网页应用的例子，不同的是远程机器的22端口是关闭的，而本机的22端口是可访问的。

这时候只能把SSH服务器部署在本地机器(host4)上，SSH客户端则在远程机器上。远程机器上执行如下命令能够能到和本地豆端口转发同样的效果：

```
ssh -NfR 80:localhost:8080 host4
```

注意，无论是本地端口转发还是远程端口转发，都是从SSH客户端发出命令。`-L`和`-R`参数前面端口都是数据转发源，后面的地址和端口都是转发的目的地。

### 动态端口转发

在前面两个例子中，我们都是把数据转发给固定地址和固定端口。这种方式显然不足以满足场景三的需求。那么能不能够取消这种限制，把SSH服务器当作纯粹的代理，访问任意的地址和端口呢？

答案是可以的，假设host5是SSH服务器，执行如下命令：

```
ssh -NfD 8080 host5
```

上述命令表示客户端访问8080端口的请求都转发给了SSH服务器。但是SSH服务器怎么知道客户端要访问的地址和端口呢？事实上，SSH协议只是提供了一条安全的数据传输隧道。代理的工作（包括提供访问地址和端口）是由其他协议提供的。当前OPENSSH提供SOCK4和SOCK5代理协议。要想通过SSH服务器代理我们的请求，发送请求的客户端必须使用SOCKS代理，现在浏览器的自带设置或者插件都能支持SOCKS代理。

## 参考

man ssh_config

man ssh

https://www.ibm.com/developerworks/cn/linux/l-cn-sshforward/index.html

https://www.chenyudong.com/archives/linux-ssh-port-dynamic-forward.html

https://www.jianshu.com/p/f2156c444fd6