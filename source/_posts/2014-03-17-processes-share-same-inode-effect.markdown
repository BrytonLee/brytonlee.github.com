---
layout: post
title: "进程间共享inode相互影响简单分析"
date: 2014-03-17 10:03
comments: true
categories: 文件系统 inode 文件清空 进程之间共享 文件恢复
---

起因:
------------
上周我们团队的实习生做了一个rsyslog+logrotate配置的分享，logrotate有一个配置指令create, 意思是在轮转日志的时候先把之前的文件删除，然后重新创建一个新的文件。文件名可以是不同名，也可以是同名，由此我想到一个问题，如果是同名文件，使用create指令的方式能有什么益处？

这篇文章假设的一个前提是文件是同名的，为了做日志轮转，有一种方式就是直接清空现有的文件，然后新的日志还是往这个文件里面写入。但是使用create指令的方式或许能在多进程读写相同文件时避免一些问题，请看下文。

准备工作:
----------
为了模拟多个进程同时读写相同的文件，我写了几个小程序(C语言,目前只会点C，读者见谅)，第一个程序是一个写程序(write.c)：
    
      1 #include <stdio.h>
      2 #include <sys/types.h>
      3 #include <sys/stat.h>
      4 #include <fcntl.h>
      5 #include <unistd.h>
      6 
      7 int main(void) {
      8     int fd;
      9     int i;
     10     ssize_t w_size;
     11 
     12     fd = open("/home/bryton/a",  O_RDWR | O_APPEND);
     13     if ( fd != -1 ) {
     14         for (i = 0; i < 10000; i++) {
     15             w_size = write(fd, "adcdef\n", sizeof("adcdef\n") -1);
     16             if ( w_size == -1 ) {
     17                 printf("write error\n");
     18                 return -1;
     19             }
     20             sleep(1);
     21         }
     22         close(fd);
     23 
     24     } else {
     25         return -1;
     26     }
     27     return 0;
     28 }

这个程序很简单，以追加的方式往/home/bryton/a这个文件里面追加内容,for循环10000次往文件a追加"adcdef\n"字符串，每追加一次，sleep 1s,这样不至于10000次循环很快就跑完了。 

第二个程序是一个读程序(read.c)：

      1 #include <stdio.h>
      2 #include <string.h>
      3 #include <sys/types.h>
      4 #include <sys/stat.h>
      5 #include <fcntl.h>
      6 #include <unistd.h>
      7 
      8 
      9 int main(void) {
     10     int fd;
     11     char buffer[1024];
     12     ssize_t read_size;
     13 
     14     fd = open("/home/bryton/a", O_RDONLY);
     15     if ( fd != -1 ) {
     16         while ( 1 ) {
     17             memset(buffer, 0, 1024);
     18             read_size = read(fd, buffer, 1024);
     19             if ( read_size > 0 ) {
     20                 printf("%s", buffer);
     21             }
     22         }
     23     } else {
     24         return -1;
     25     }
     26     close(fd);
     27     return 0;
     28 }

读程序也是很简单， while( 1 ) { ... } 一直读/home/bryton/a这个文件，每读一次打印一次。

实验
---------
有了这两个程序我们就能做一个小实验。首先通过gcc编译这两个小程序：

    $ gcc -o write write.c
    $ gcc -o read read.c
    
接着在/home/bryton/目录下创建一个名为a文件(笔者自己的HOME目录是/home/bryton,在读者自己的机器测试时可以改成对应的目录，同时程序也要小小的修改下)
    
    $ touch a
    $ tail -f a
    
然后在开启另外一个终端，在另外一个终端里面输入：

    $ ./write
    
这时可以看到tail -f a 有输出：
    
    $ tail -f a
    adcdef
    adcdef
    adcdef
    adcdef
    adcdef

确定write有效之后，按Control - c键退出tail命令，接着执行:

    $ ./read
    adcdef
    adcdef
    adcdef
    adcdef
    
我们能看到和tail -f一样的输出，说明read程序也正常工作了。**接下来要做的这一步很重要，如果把/home/bryton/a文件删除会如何？write和read这两个小程序还能正常工作吗？**读者朋友可以先想一想不要着急回答。

    $ rm -f a 
    
删除之后，只能观察./read是否依然能正常输出，因为./write本来就没有输出。如果还是依然能正常输出，就说明read和write两个小程序依然工作正常，结果是这两个程序依然能正常工作，如下:

    $ ./read
    adcdef
    adcdef
    adcdef
    adcdef
    adcdef
    ......
    ......
    adcdef
    adcdef

我用......表示中间已经输出了很多行，由于不能动画显示，也只能这样了,读者可以自己做实验。

不知道这样的结果读者是否已经提前预知，不管怎么样，到此为止要想想为什么了，rm已经把文件删除，在终端下通过ls命令也无法看到，但是read和write还能正常工作，什么原因使得这两个程序还能正常工作呢。**其实文件一般在文件系统中可以简单的认为由两部分组成，一部分称为inode，inode用于描述文件的基本信息包括文件名，inode号，访问时间，修改时间，权限等信息，inode的数量是在文件系统格式化的时候创建的，另外一部分称为data block, 显然data block用于存放文件的实际数据;而目录对于文件系统其实也是一个文件，与文件不同的是，目录的data block存放的是目录下文件的信息，其中很重要的就是文件的inode号，inode是标识文件的唯一关键信息，通常认为文件名和路径标识一个文件的认识是不全面的。**

有了inode是标识文件唯一的认识之后，继续分析上面的实验，./write和./read通过open系统调用打开的文件/home/bryton/a实际上是把/home/bryton/a的inode从文件系统读入到内存，open系统调用把inode读入到内存之后，通过一个文件描述符(file descriptor)关联到inode,通过文件描述符读取(read)或者写入(write)到文件最终转换成通过inode对文件进行操作，**虽然通过rm命令在文件系统上删除了/home/bryton/a这个文件，但是实际上这个文件只是从/home/bryton目录的data block里面去除了a的inode信息,并没有从文件系统上删除掉**

所以./write还能继续往文件系统里面写， ./read还能继续从文件系统里面读取出来，通过一张图加深下对此刻的理解：

![file_deleted_in_dir_inode_shared_by_processers](/images/file_deleted_in_dir_inode_shared_by_processers.png)

    Tips: 1. *nix系统有一个lsof命令能看到目前进程正在打开的文件有哪些。 通过lsof | grep deleted 能查询到在目录里面删除了，但是依然被进程打开了的文件。
    
    Tips: 2. 如果一个文件被进程打开着，并且由于人为误操作把文件删除了(运维当中偶尔会遇到)，可以通过cp /proc/PID/fd/FD /path/to/save/
    进行恢复，PID表示打开了这个文件的进程ID，FD表示进程打开的文件描述符号。

回到文章开始的事情上，**如果logrotate配置create指令轮转同名文件日志，虽然日志文件在目录下面找不到了，但是对于已经在处理日志的进程来说却不受影响，新创建的文件用于存放新的日志，老的日志进程依然能正确处理。**

如果logrotate不是配置create指令，对于正在处理日志文件的进程到底会有什么样的影响呢? 我们还可以做一个实验，这个实验验证两种情况：

    1.文件被进程以读的方式打开，文件被清空对于正在读取文件的进程有什么影响。
    2.文件被进程以写的方式打开，文件被清空对于正在写的进程有什么影响。
    
实验2
------
为了验证第一种进程以读方式打开文件，文件被清空的情况，需要一个新的read程序--read2.c, read2.c只读取两次，在读取完第一次并打印之后，进程睡眠10s,这样有10s钟的操作时间去清空文件，清空文件的操作没有重新去写一个新的清空程序，而是使用bash的'>'操作。

      1 #include <stdio.h>
      2 #include <sys/types.h>
      3 #include <sys/stat.h>
      4 #include <fcntl.h>
      5 #include <unistd.h>
      6 #include <string.h>
      7 
      8 int main(void) {
      9     int fd;
     10     char buffer[128];
     11     ssize_t read_size;
     12 
     13     fd = open("/home/bryton/b", O_RDONLY);
     14     if ( fd != -1 ) {
     15         memset(buffer, 0, 128);
     16         read_size = read(fd, buffer, 127);
     17         if ( read_size > 0 ) {
     18             printf("%s", buffer);
     19         }
     20         printf("\nsleep 10s...\n");
     21         sleep(10);
     22         memset(buffer, 0, 128);
     23         read_size = read(fd, buffer, 127);
     24         if ( read_size > 0) {
     25             printf("%s", buffer);
     26         }
     27     } else {
     28         return -1;
     29     }
     30     return 0;
     31 }


验证过程如下：
首先在HOME目录下创建一个b文件，内容如下：

      1 aaaaaaaaaaaaaaaaaa
      2 aaaaaaaaaaaaaaaaaa
      3 aaaaaaaaaaaaaaaaaa
      4 aaaaaaaaaaaaaaaaaa
      5 aaaaaaaaaaaaaaaaaa
      6 aaaaaaaaaaaaaaaaaa
      7 aaaaaaaaaaaaaaaaaa
      8 bbbbbbbbbbbbbbbbbb
      9 bbbbbbbbbbbbbbbbbb
     10 bbbbbbbbbbbbbbbbbb
     11 bbbbbbbbbbbbbbbbbb
     12 bbbbbbbbbbbbbbbbbb
     13 bbbbbbbbbbbbbbbbbb
     14 bbbbbbbbbbbbbbbbbb

编译并执行read2.c

    $ gcc -o read2 read2.c
    $ ./read2
    aaaaaaaaaaaaaaaaaa
    aaaaaaaaaaaaaaaaaa
    aaaaaaaaaaaaaaaaaa
    aaaaaaaaaaaaaaaaaa
    aaaaaaaaaaaaaaaaaa
    aaaaaaaaaaaaaaaaaa
    aaaaaaaaaaaaa
    sleep 10s...
 
当read2打印出sleep 10s的时候立即在另外一个终端执行清空文件b的操作：

    $ >b
10s之后，read2读第二次，但是没有任何输出，**由此可见当清空一个正在被进程读的文件，进程将不能读取到文件后面的内容**，如果是一个日志处理程序，在logrotate清空文件这一刻，如果文件内容没有处理完，内容将会丢失。

接着验证第二种情况。当一个进程以写的方式打开一个文件时，文件被清空会有什么影响。写日志文件复用write.c程序，清空操作还是使用bash的'>'。首先编译并执行write.c

    $ gcc -o write write.c
    $ ./write
    $
     
接着在另外一个终端先cat a这个文件

    $ cat a
    adcdef
    adcdef
    adcdef
    adcdef
    adcdef
    adcdef
    adcdef
    adcdef
    adcdef
    $ 

然后执行清空操作并立即cat a查看文件的变化

    $ >a
    $ cat a
    adcdef
    adcdef
    $
此时能够看到文件的内容看起来像是被截断了一样，其实文件已经被清空，新增的两行是write程序在清空之后新写入的(write程序始终写入的是adcdef字符串，所以不太好区分旧的输出和新的写入)。

利用这个特性，在运维高可用的应用时有一个非常好的用途，高可用的程序往往不能重启，程序以追加的方式写日志文件。当磁盘空间不足时，必须要清空磁盘空间才能使程序继续运行。如果直接删除日志文件而不重启进程会发现磁盘空间依然没有得到释放，du命令查看目录下的文件占用空间是变少了，但是df查看整个磁盘的空间确依然没不减。如果日志并不重要，这个时候可以使用清空的方式，**不需要重启进程，也能释放磁盘空间。**

