---
layout: single
author_profile: true
comments: true
title: Protocol Buffers 笔记
tagline: 
category: essay
tags : [C/C++, Python]
---

## Protocol Buffers

> Protocol buffers are Google's language-neutral, platform-neutral, extensible mechanism for serializing structured data 
  – think XML, but smaller, faster, and simpler.   
  You define how you want your data to be structured once, then you can use special generated source code to 
  easily write and read your structured data to and from a variety of data streams and using a variety of languages. 

特性解读

- language-neutral: 语言无关的，
- platform-neutral
- serializing `structured data`
- define once, use anywhere
- 目前官方只支持C++, Python, Java三种语言

如果在一个大项目中，使用了多种语言，而且他们之间还涉及通信，这时 `protobuf` 将成为一个利器，
只需定义一次protobuf规范的Message结构，然后生成不同语言的Message类。 所以这里的关键就是抽象出Message.

#### Serialization

> 序列化 (Serialization)将对象的状态信息转换为可以存储或传输的形式的过程。（百度百科）

> In computer science, in the context of data storage, serialization is the process of translating data structures or object state into a format that can be stored (for example, in a file or memory buffer, or transmitted across a network connection link) and reconstructed later in the same or another computer environment. (维基百科)

> Serialization is the process of converting an object into a stream of bytes in order to store the object or transmit it to memory, a database, or a file. Its main purpose is to save the state of an object in order to be able to recreate it when needed. The reverse process is called deserialization. (MSDN)

从定义可以看出，序列化的关键有

- 直观上说，序列化是将结构化数据变成“流”数据的过程，也是存储过程。
- 序列化要保证能够还原，即反序列化。如将C语言的struct结构数据存储到文件，再读出还原成struct对象。
- 序列化，反序列化，都得遵从一个共同的格式标准，即存储对象的结构描述，protobuf的消息格式保存在proto文件中。
- 存储不仅有原始数据，对象的数据成员之间如何分隔？数据本身用不用加上结构描述信息，自描述？这就涉及一关键的存储效率问题。
- XML和JSON语言，有很好的可读性，也算是自我描述吧，protobuf不具有此功能，它需要额外的类文件进行描述（从proto文件编译成不同文件的消息类）。


## Example

这个实例演示了，如何通过Python程序向address_book文件写数据，通过C++程序从address_book文件读取数据，并且还原为原来的结构。

### 1.目录结构

    root@ubuntu:~# pwd 
    /root
    root@ubuntu:~# tree protobuf/
    protobuf/
    ├── addressbook.proto
    ├── reader.cc
    └── writer.py
    
    0 directories, 3 files
    
    
### 2.写文件

编译proto文件，生成相应的Python文件，它包含相关类定义等  

编译命令： `protoc -I=$SRC_DIR --python_out=$DST_DIR $SRC_DIR/addressbook.proto`  

    root@ubuntu:~# SRC_DIR=protobuf/
    root@ubuntu:~# DST_DIR=protobuf/
    root@ubuntu:~# protoc -I=$SRC_DIR --python_out=$DST_DIR $SRC_DIR/addressbook.proto

在Python程序中导入protoc生成的模块，直接执行。

    root@ubuntu:~# python protobuf/writer.py 
    address_book :  person {
      name: "Dave s0"
      id: 0
      email: ""
      phone {
        number: "13233333333"
        type: MOBILE
      }
    }
    ....
  
  并将person信息保存到当前目录的address_book文件中。  
  
    root@ubuntu:~# file address_book 
    address_book: data
  
### 3.读文件

  编译命令： `protoc -I=$SRC_DIR --cpp_out=$DST_DIR $SRC_DIR/addressbook.proto`  
  
    root@ubuntu:~# SRC_DIR=protobuf/
    root@ubuntu:~# DST_DIR=protobuf/
    protoc -I=$SRC_DIR --cpp_out=$DST_DIR $SRC_DIR/addressbook.proto

编译C++程序时需要加上链接选项`-lprotobuf`  

    root@ubuntu:~# g++ $SRC_DIR/reader.cc $SRC_DIR/addressbook.pb.cc -lprotobuf -o $DST_DIR/reader
    root@ubuntu:~# $DST_DIR/reader address_book 
    Person ID: 0
      Name: Dave s0
      E-mail address: 
      Mobile phone #: 13233333333
    ...
 
### 4.目录结构

    root@ubuntu:~# tree protobuf/
    protobuf/
    ├── addressbook_pb2.py
    ├── addressbook_pb2.pyc
    ├── addressbook.pb.cc
    ├── addressbook.pb.h
    ├── addressbook.proto
    ├── reader
    ├── reader.cc
    └── writer.py
    
    0 directories, 8 files

## 参考

参考Google官方文档，包含proto文件，及Python和C++示例程序。

- https://developers.google.com/protocol-buffers/docs/pythontutorial
- https://developers.google.com/protocol-buffers/docs/cpptutorial
