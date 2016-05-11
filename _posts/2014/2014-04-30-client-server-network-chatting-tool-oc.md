---
layout: single
author_profile: true
comments: true
title: C/S架构的网络聊天工具OC
categories: [Python]
tags: [Python, pyqt, 网络编程]
---
网络编程的一个大作业《编写网络聊天工具》，拿出来和大家分享一下实现思路。
<h1><strong>1.简介</strong></h1>
模仿QQ来写的一个工具，使用Python+PyQt开发，为快速实现网络聊天工具的基本逻辑，服务器端没有实用数据库，客户端有登录窗口、注册窗口、好友列表窗口和对话框。主要功能包括：
<ul>
	<li>注册、登录；</li>
	<li>添加好友；</li>
	<li>支持多会话；</li>
	<li>支持好友上线、下线通知；</li>
	<li>一对一聊天，支持离线消息；</li>
	<li>文件传输，支持离线文件；</li>
</ul>
<h1><strong>2.服务器设计</strong></h1>
<h2><b>2.1.</b> 用户管理模块</h2>
为实现一个网络聊天工具的基本业务逻辑，在基本要求和扩展要求的基础上，添加用户注册和登录验证功能。服务器需要保存用户注册信息，一般会采用数据 库，本实验为确保程序运行环境的灵活性（作为一实验，简洁为上，实际上用数据库更方便），将用户注册信息保存在文件server\user \user.pk中。密码在传输过程中和在服务器保存的形式均为MD5密文，提高了安全性。User.pk文件的内容其实一个字典，如下所示：

	users={user_no_1: [email, nickname, password_md5], user_no_2: [email, nickname, password_md5]….}

user_no_n就是在注册时随机生成的一个6位的整数作为唯一标识OC号（类似QQ号，本实验中用于标识唯一用户的数字串），此处没有添加多余的字段，只有用户邮箱、昵称和密码。

另外一个重要的文件server\user\friends.pk用于保存好友关系，用于字典和列表实现，格式如下：

	friends={user_no_0: [user_no_0,user_no_1, user_no_2,…], user_no_1: [user_no_0,user_no_1, user_no_3,…]…}

列表friends[user_no_n]保存的是用户user_no_n的所有好友的OC号。

至于在线用户，服务器会在内存中存储一个onlineUser列表，保存在线用户的OC号。
<h2><b>2.2. </b>聊天消息存储模块</h2>
服务器在转发聊天消息的同时将聊天记录保存到磁盘上，由于实验中数据比较少，也不采用数据库存储，并且所有的聊天记录保存在一个文件server\msg\history.pk中，每一条聊天记录的格式如下所示：

	send_user_no \t recv_user_no \t cur_time \ t msg_content \n

如果接收方不在线，那么将消息保存到离线消息文件server\msg\offline.pk中，格式如下：

	send_user_no \t recv_user_no \t cur_time \ t msg_content \n

待接收方上线后，将离线消息发送给他，并且将该离线消息从offline.pk文件移动到history.pk文件中。
<h2><b>2.3. </b>文件存储模块</h2>
所有的上传文件保存的server\file\文件夹中，在该文件夹下有一个目录文件file_info.txt，用于保存所有的文件信息，格式如下：

	send_user_no \t recv_user_no \t upload_time \ t file_name\n

在上传成功后，如果接收方在线，服务器向接收方发送提示消息，接收方随时可以下载，如果接收主不在线，等接收方上线后会进行提醒。此外，在两个好友A和B会话期间，对话框右侧的文件列表中会显示所有可下载的文件（准确的说是两人进行传送过的所有历史文件），可以随时下载。
<h2><b>2.4. </b>服务器消息转发</h2>
为实现消息的快速转发，服务器动态维护一个字典sessions，建立会话和关闭会话时，会动态增删。格式如下：

	sessions={user_no_1: {user_no_2: socket,.. }, user_no_2: {user_no_1: socket,.. }…}

sessions[user_no_x][user_no_y]表示user_no_x和服务器建立的一个socket会话，而这个会话的消息是发 送给用户user_no_y的，服务器从sessions[user_no_x][user_no_y]获取消息时，转发给 sessions[user_no_y][user_no_x](如果存在的话)。
<h2>2.5.服务器向客户端推送消息</h2>
服务器大多是在响应客户端的请求，实验中所有的socket都是由客户端发起连接，但是有时候服务器也需要向客户端主动发送消息，所以服务器需要为每个客 户端保存一个socket，用于推送消息（包括新消息提醒、离线，以及上线/下线提醒）。为实现该功能，服务器实时维护一个socket列表 cmd_sock，保存在线用户的接收信息的socket.

<h2>2.6.<b>请求报文的格式</b><b></b></h2>
这是服务器设计中最重要一环，为实现客户端和服务器的通信，双方必须约定一个格式（也就是协议），为了实现简单，容易理解，使用python中的 struct.pack和struct.unpack，每个报文TCP传输的数据（也就是应用层部分）包括四个字段：

<div>
<table border="1" cellspacing="0" cellpadding="0">
<tbody>
<tr>
<td valign="top" width="142">FIELD-1</td>
<td valign="top" width="142">FIELD-2</td>
<td valign="top" width="142">FIELD-3</td>
<td valign="top" width="142">FIELD-4</td>
</tr>
</tbody>
</table>
FIELD-1标识报文类型，FIELD-2~4根据FIELD-1有不同的含义。
<table border="1" cellspacing="0" cellpadding="0">
<tbody>
<tr>
<td valign="top" width="139">FIELD-1</td>
<td valign="top" width="76">FIELD-2</td>
<td valign="top" width="85">FIELD-3</td>
<td valign="top" width="85">FIELD-4</td>
<td valign="top" width="183">含义</td>
</tr>
<tr>
<td valign="top" width="139">SIGN_IN</td>
<td valign="top" width="76">email</td>
<td valign="top" width="85">Nickname</td>
<td valign="top" width="85">Pwd_md5</td>
<td valign="top" width="183">注册新用户</td>
</tr>
<tr>
<td valign="top" width="139">LOGIN</td>
<td valign="top" width="76">User_no</td>
<td valign="top" width="85">Pwd_md5</td>
<td valign="top" width="85"></td>
<td valign="top" width="183">登录</td>
</tr>
<tr>
<td valign="top" width="139">DOWN</td>
<td valign="top" width="76">User_no</td>
<td valign="top" width="85"></td>
<td valign="top" width="85"></td>
<td valign="top" width="183">下线通知</td>
</tr>
<tr>
<td valign="top" width="139">CMD_SOCKET</td>
<td valign="top" width="76">User_no</td>
<td valign="top" width="85"></td>
<td valign="top" width="85"></td>
<td valign="top" width="183">标识此socket用于接收推送消息</td>
</tr>
<tr>
<td valign="top" width="139">GET_FRIENDS</td>
<td valign="top" width="76">User_no</td>
<td valign="top" width="85"></td>
<td valign="top" width="85"></td>
<td valign="top" width="183">获取自己的好友列表</td>
</tr>
<tr>
<td valign="top" width="139">GET_ONLINE</td>
<td valign="top" width="76">User_no</td>
<td valign="top" width="85"></td>
<td valign="top" width="85"></td>
<td valign="top" width="183">获取目前在线的用户</td>
</tr>
<tr>
<td valign="top" width="139">FIND_FRIEND</td>
<td valign="top" width="76">User_no</td>
<td valign="top" width="85"></td>
<td valign="top" width="85"></td>
<td valign="top" width="183">根据oc_no查询一个用户的信息</td>
</tr>
<tr>
<td valign="top" width="139">ADD_FRIEND</td>
<td valign="top" width="76">User_no</td>
<td valign="top" width="85">Friend_no</td>
<td valign="top" width="85"></td>
<td valign="top" width="183">添加好友</td>
</tr>
<tr>
<td valign="top" width="139">OFFLINE_MSG</td>
<td valign="top" width="76">User_no</td>
<td valign="top" width="85"></td>
<td valign="top" width="85"></td>
<td valign="top" width="183">查询有没有自己的离线消息</td>
</tr>
<tr>
<td valign="top" width="139">NEW_SESSION</td>
<td valign="top" width="76">User_no</td>
<td valign="top" width="85">Friend_no</td>
<td valign="top" width="85"></td>
<td valign="top" width="183">建立会话请求</td>
</tr>
<tr>
<td valign="top" width="139">MESG</td>
<td valign="top" width="76">User_no</td>
<td valign="top" width="85">Friend_no</td>
<td valign="top" width="85">Msg</td>
<td valign="top" width="183">发送消息</td>
</tr>
<tr>
<td valign="top" width="139">UPLOAD</td>
<td valign="top" width="76">User_no</td>
<td valign="top" width="85">Friend_no</td>
<td valign="top" width="85">File_name</td>
<td valign="top" width="183">上传文件请求</td>
</tr>
<tr>
<td valign="top" width="139">UPLOAD</td>
<td valign="top" width="76">Data_length</td>
<td valign="top" width="85">File_name</td>
<td valign="top" width="85">File_content</td>
<td valign="top" width="183">上传文件内容</td>
</tr>
<tr>
<td valign="top" width="139">UPLOAD_FINISH</td>
<td valign="top" width="76">User_no</td>
<td valign="top" width="85">Friend_no</td>
<td valign="top" width="85">File_name</td>
<td valign="top" width="183">上传结束</td>
</tr>
<tr>
<td valign="top" width="139">DOWN_FILE_NAME</td>
<td valign="top" width="76">User_no</td>
<td valign="top" width="85">Friend_no</td>
<td valign="top" width="85"></td>
<td valign="top" width="183">获取可下载的文件列表</td>
</tr>
<tr>
<td valign="top" width="139">DOWNLOAD</td>
<td valign="top" width="76">File_name</td>
<td valign="top" width="85">Recv_state</td>
<td valign="top" width="85">Recvd_bytes</td>
<td valign="top" width="183">下载文件</td>
</tr>
</tbody>
</table>
</div>

<h2><b>2.7.</b>服务器的I/O模型</h2>
采用select模型实现I/O多路复用，使用队列Queue实现消息发送，具体参见server\server.py，主体框架代码并不多。
<h1><strong>3.客户端设计</strong></h1>
上一节已经详细介绍了请求的各种类型，客户端如果想得到某些信息，只需要按相应的数据报文类型格式进行请求，并解析服务器响应即可。这时客户端只剩下界面设计部分。
<h2>3.1. 流程图</h2>
<img src="/assets/images//client-framework.jpg">  

<h2>3.2. 界面设计</h2>
使用QT Designer设计了四个界面login.ui, register.ui, main.ui, chat.ui ,使用pyuic4命令生成对应的py文件，稍微修改。Client\main.py是主要的界面和逻辑实现部分，client\net.py是 main.py用到的一些辅助的函数。<b> </b>

<img src="/assets/images/oc_login.jpg">    
<img src="/assets/images/oc_register.jpg">   
<img src="/assets/images//mainWindow.jpg">    
<img src="/assets/images/chat_dialog.jpg">   

<h2>3.3. 聊天对话框</h2>
启动聊天对话框后，首先从服务器获取可下载的文件列表，如果有离线消息的话，也是从主程序传过来的，在此不用查询。
<h2>3.4.文件上传与下载</h2>
服务器设计的相应报文格式：
<div>
<table border="1" cellspacing="0" cellpadding="0">
<tbody>
<tr>
<td valign="top" width="38">1</td>
<td valign="top" width="139">UPLOAD</td>
<td valign="top" width="76">User_no</td>
<td valign="top" width="85">Friend_no</td>
<td valign="top" width="85">File_name</td>
<td valign="top" width="151">上传文件请求</td>
</tr>
<tr>
<td valign="top" width="38">2</td>
<td valign="top" width="139">UPLOAD</td>
<td valign="top" width="76">Data_length</td>
<td valign="top" width="85">File_name</td>
<td valign="top" width="85">File_content</td>
<td valign="top" width="151">上传文件内容</td>
</tr>
<tr>
<td valign="top" width="38">3</td>
<td valign="top" width="139">UPLOAD_FINISH</td>
<td valign="top" width="76">User_no</td>
<td valign="top" width="85">Friend_no</td>
<td valign="top" width="85">File_name</td>
<td valign="top" width="151">上传结束</td>
</tr>
<tr>
<td valign="top" width="38">4</td>
<td valign="top" width="139">DOWN_FILE_NAME</td>
<td valign="top" width="76">User_no</td>
<td valign="top" width="85">Friend_no</td>
<td valign="top" width="85"></td>
<td valign="top" width="151">获取可下载的文件列表</td>
</tr>
<tr>
<td valign="top" width="38">5</td>
<td valign="top" width="139">DOWNLOAD</td>
<td valign="top" width="76">File_name</td>
<td valign="top" width="85">Recv_state</td>
<td valign="top" width="85">Recvd_bytes</td>
<td valign="top" width="151">下载文件</td>
</tr>
</tbody>
</table>
</div>

报文1是通知服务器有文件即将上传，服务器如果收到就返回一个“RECV_NOTICE”，客户端收到之后才开始传送文件内容； 

报文2是用来传送文件内容的，它告诉服务器传送的文件名称和传送的字节数，服务器正确收到后响应一个“RECV_OK”；  

报文3是在客户端上传结束时，发送该报文通知服务器结束；  

报文4是客户端告诉服务器要下载的文件名；  

报文5是从服务器下载文件时，服务器每发送一次文件内容，客户端响应一个该报文，它标识了下载的文件名，已经接收的字节数，服务器根据已经接收的字节数 ，可以直接定位到文件的准确位置，避免了在传送过程中一直打开文件，或者一次性将文件内容读入内存。  

<h1><strong>4.总结</strong></h1>
实现了一个简易逻辑完整的网络聊天工具，聊天室部分由于时间关系没有实现，不过实现思路和两人聊天相同，界面需要重新设计一个，还有共享文件等。本工具的消息提示方式不够人性化，是弹窗方式，不知道为什么，PyQt的系统托盘消息提示在win7下不好使。

附：

完整代码见: <a title="网络聊天工具OC" href="https://github.com/onestraw/OC" target="_blank">https://github.com/onestraw/OC</a>
