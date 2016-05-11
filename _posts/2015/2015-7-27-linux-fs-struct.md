---
layout: single
author_profile: true
comments: true
title: Linux文件系统数据结构
tagline: 
category: Linux
tags : [Linux]
---

下面是Linux fs 相关数据结构的一些笔记，列出的是我开发中经常用到的一些结构成员，以linux 3.12为例。

#dentry

      struct dentry{
      	struct dentry 	*d_parent;	      /* parent directory */
      	struct qstr 	d_name;		
      	struct inode 	*d_inode;	      /* Where the name belongs to */
      	unsigned char d_iname[DNAME_INLINE_LEN];        /* small names */
      	...
      };

dentry: directory entry   

detail: http://goo.gl/Lisgeg

#inode

      struct inode{
      	unsigned long           		i_ino;
      	umode_t                 		i_mode;
      	struct super_block              *i_sb;
      	const struct file_operations    *i_fop;
      	void                   			*i_security;
      	...
      };
      
  detail: http://goo.gl/TphYHp   
  
`inode->i_mode` 是 16位整数，下面是8进制表示

      #define S_IFMT  00170000
      #define S_IFSOCK 0140000
      #define S_IFLNK	 0120000
      #define S_IFREG  0100000
      #define S_IFBLK  0060000
      #define S_IFDIR  0040000
      #define S_IFCHR  0020000
      #define S_IFIFO  0010000
      #define S_ISUID  0004000
      #define S_ISGID  0002000
      #define S_ISVTX  0001000
      
      #define S_ISLNK(m)	(((m) & S_IFMT) == S_IFLNK)
      #define S_ISREG(m)	(((m) & S_IFMT) == S_IFREG)
      #define S_ISDIR(m)	(((m) & S_IFMT) == S_IFDIR)
      #define S_ISCHR(m)	(((m) & S_IFMT) == S_IFCHR)
      #define S_ISBLK(m)	(((m) & S_IFMT) == S_IFBLK)
      #define S_ISFIFO(m)	(((m) & S_IFMT) == S_IFIFO)
      #define S_ISSOCK(m)	(((m) & S_IFMT) == S_IFSOCK)
      
      #define S_IRWXU 00700
      #define S_IRUSR 00400
      #define S_IWUSR 00200
      #define S_IXUSR 00100
      
      #define S_IRWXG 00070
      #define S_IRGRP 00040
      #define S_IWGRP 00020
      #define S_IXGRP 00010
      
      #define S_IRWXO 00007
      #define S_IROTH 00004
      #define S_IWOTH 00002
      #define S_IXOTH 00001

举例

    root@ubuntu:~# ls -il temp.txt 
    681278 -rw-r--r-- 1 root root 56844 Jul  9 08:46 temp.txt

i_ino = 681278   
i_imode = 00644   

#super_block

      struct super_block {
            unsigned long               s_magic;
            void                        *s_security;
            const struct xattr_handler  **s_xattr;
      };

`s_magic`: http://goo.gl/jOhHRM  

`s_xattr` 文件扩展属性


    struct xattr_handler{
        size_t (*list)(...);
        int (*get)(...);
        int (*set)(...);
    };
    struct xattr {
        const char *name;
        void *value;
        size_t value_len;
    };


用户空间使用attr示例  
    

          root@ubuntu:~# apt-get install attr
          root@ubuntu:~# touch hello.c
          root@ubuntu:~# setfattr -n user.author -v geeksword -h hello.c 
          root@ubuntu:~# getfattr -n user.author hello.c 
          # file: hello.c
          user.author="geeksword"
          root@ubuntu:~# 


涉及的系统调用是

        setxattr()
        getxattr()

#file

      struct file{
      	struct path             		f_path;
      	unsigned int            		f_flags;	/* 文件属性 */
      	fmode_t                 		f_mode;	/* 文件打开模式 */
      	struct inode            	    *f_inode;	
      	const struct file_operations    *f_op;
      	const struct cred       		*f_cred;
      	void                    		*f_security;
      };
  
  detail: http://goo.gl/56ouH4   

位于源码/include/linux/fs.h头部




file->f_flags:  http://goo.gl/YCkhZq   


    #define O_ACCMODE	00000003
    #define O_RDONLY	00000000
    #define O_WRONLY	00000001
    #define O_RDWR		00000002
    #define O_CREAT		00000100	/* not fcntl */
    #define O_EXCL		00000200	/* not fcntl */
    #define O_NOCTTY	00000400	/* not fcntl */
    #define O_TRUNC		00001000	/* not fcntl */
    #define O_APPEND	00002000
    #define O_NONBLOCK	00004000
    #define O_DSYNC		00010000	/* used to be O_SYNC, see below */
    #define FASYNC		00020000	/* fcntl, for BSD compatibility */
    #define O_DIRECT	00040000	/* direct disk access hint */
    #define O_LARGEFILE	00100000
    #define O_DIRECTORY	00200000	/* must be a directory */
    #define O_NOFOLLOW	00400000	/* don't follow links */
    #define O_NOATIME	01000000
    #define O_CLOEXEC	02000000	/* set close_on_exec */


`file->f_mode`

       /*
       * flags in file.f_mode.  Note that FMODE_READ and FMODE_WRITE must correspond
       * to O_WRONLY and O_RDWR via the strange trick in __dentry_open()
       */
      
      /* file is open for reading */
      #define FMODE_READ		((__force fmode_t)0x1)
      /* file is open for writing */
      #define FMODE_WRITE		((__force fmode_t)0x2)
      /* file is seekable */
      #define FMODE_LSEEK		((__force fmode_t)0x4)
      /* file can be accessed using pread */
      #define FMODE_PREAD		((__force fmode_t)0x8)
      /* file can be accessed using pwrite */
      #define FMODE_PWRITE		((__force fmode_t)0x10)
      /* File is opened for execution with sys_execve / sys_uselib */
      #define FMODE_EXEC		((__force fmode_t)0x20)
      /* File is opened with O_NDELAY (only set for block devices) */
      #define FMODE_NDELAY		((__force fmode_t)0x40)
      /* File is opened with O_EXCL (only set for block devices) */
      #define FMODE_EXCL		((__force fmode_t)0x80)


一些文件操作相关函数参数mask的掩码，标识访问文件的方式

      #define MAY_EXEC		0x00000001
      #define MAY_WRITE		0x00000002
      #define MAY_READ		0x00000004
      #define MAY_APPEND	0x00000008
      #define MAY_ACCESS	0x00000010
      #define MAY_OPEN		0x00000020
      #define MAY_CHDIR		0x00000040




#file_operations

      struct file_operations{
      	int (*open) (struct inode *, struct file *);
      	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
      	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
      	...
      };

details: http://goo.gl/oMQuKj   

#path

      struct path {
      	struct vfsmount *mnt;
      	struct dentry *dentry;
      };


#more

- [open系统调用入口](http://lxr.free-electrons.com/source/fs/open.c?v=3.12#L983)
- [linux 2.x read 系统调用分析](https://www.ibm.com/developerworks/cn/linux/l-cn-read/)
