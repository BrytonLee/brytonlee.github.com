---
layout: post
title: "[Java] Netty误用导致JVM Crash问题排查"
date: 2014-08-21 10:24
comments: true
categories: 
---

前两天一位同事给我留言，有一个应用在搬机房之后，集群之中有几台机器出现JVM crash的问题。JVM crash之后留下的信息很少，跟我们平常看到的JVM crash现象有较大的区别。 没有hs_err_xxx.log，也没有OOM-killer的日志。

故障现象：
----------
接到这个Case之后，我登陆到机器上去查看相关信息，这个时候JVM进程已经停止，日志文件里面没有OutOfMemoryException, /var/log/kern和/var/log/message也没有OOM-Killer日志。 没有JVM crash log。 只有Jboss日志里面run.sh输出一句xxxpid 已杀死。

通过tsar历史数据可以看到JVM在crash这一刻，系统内存使用接近100%, JVM进程crash之后，系统内存的使用率回落到5%左右。这样的内存使用情况肯定是不正常的。由于JVM进程已经crash，留下的信息又很有限，问题不能定位。所幸的是，这个问题能稳定重现，于是我们给这台机器单独加上监控，在系统内存使用到80%的时候报警，时间的选择我觉得是比较重要的，内存使用80%的时候，基本上能说明问题，但是如果选择90%以上，可用的内存很少，登陆到机器之后，能做的事情也是有限的，而且JVM进程也随时可能挂掉，能处理的时间也很少。 

添加监控之后，启动JVM进程，等待问题重现，大约花了6个小时的系统内存的使用达到80%。于是有了下面的排查过程。

排查过程：
----------
排查的过程，我觉得很重要的是前期信息收集，信息的收集比较全面，对于问题的定位和深入是有好处的。我们需要一把“尺子”去衡量如何搜集，我的观点就是以资源的角度去搜集，系统能够提供给资源有：CPU，内存，网络。登陆到机器之后，我们先看下top的情况：
	
	$top

	top - 13:14:34 up 27 days, 19:39,  3 users,  load average: 1.49, 2.01, 2.04
	Tasks:  32 total,   1 running,  31 sleeping,   0 stopped,   0 zombie
	Cpu(s): 15.2%us,  3.2%sy,  0.0%ni, 80.8%id,  0.0%wa,  0.0%hi,  0.5%si,  0.2%st
	Mem:   8388608k total,  8388516k used,       92k free,        0k buffers
	Swap:  1999864k total,        0k used,  1999864k free,  1411772k cached

   	PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+     COMMAND
	143748 admin     20   0 12.5g 6.2g  12m S 82.1 77.9 203:07.45 java

系统内存8G, CPU 4执行线程(top命令下按下"1"能显示CPU的执行线程数，上图没有显示是因为我知道机器的配置)，Java进程143748物理内存(RES)使用6.2G, 系统的load1是1.49, Java进程CPU使用率是82.1%。这些数据看下来，能得出的结论是Java进程143748消耗的主要是内存，不是CPU，以往碰到的case当中，有些Java程序的Bug,导致了死循环，这个时候的CPU利用率很高，我们的思路是找出什么原因导致的CPU利用率很高。这个case显然不是，所以问题的核心是要找出什么原因导致的内存使用很多，避免了其他方向的查询而浪费了宝贵的故障排查时间和机会。

遇到Java内存使用的问题，最先想到的是要看下Java Heap的配置和gc的情况。JVM的启动参数如下：

	143748 ?        Sl   207:03   /opt/java/bin/java -Dprogram.name=run.sh -server -Xms4g -Xmx4g -XX:PermSize=96m -XX:MaxPermSize=384m -Xmn2g -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSMaxAbortablePrecleanTime=5000 -XX:+CMSClassUnloadingEnabled -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=80 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/home/admin/logs/java.hprof -verbose:gc -Xloggc:/home/admin/logs/gc.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Djava.awt.headless=true -Dsun.net.client.defaultConnectTimeout=10000 -Dsun.net.client.defaultReadTimeout=30000 -Djava.net.preferIPv4Stack=true -Djava.endorsed.dirs=/opt/jboss/lib/endorsed -classpath XXXXX.....

通过JVM的启动参数，我们知道Java Heap空间使用4G,接着通过jstat命令查看下Java Heap空间中内存使用和GC的情况：
	
	$sudo -u admin /opt/java/bin/jstat -gcutil 143748 1000 100
	  S0     S1     E      O      P     YGC     YGCT    FGC    FGCT     GCT
	18.67   0.00  26.64  37.49  55.88   1102   46.349     8    8.238   54.587
	18.67   0.00  33.75  37.49  55.88   1102   46.349     8    8.238   54.587
	18.67   0.00  38.47  37.49  55.88   1102   46.349     8    8.238   54.587
	18.67   0.00  45.10  37.49  55.88   1102   46.349     8    8.238   54.587
	18.67   0.00  52.19  37.49  55.88   1102   46.349     8    8.238   54.587
	18.67   0.00  57.73  37.49  55.88   1102   46.349     8    8.238   54.587
	18.67   0.00  63.58  37.49  55.88   1102   46.349     8    8.238   54.587
 	18.67   0.00  69.24  37.49  55.88   1102   46.349     8    8.238   54.587
	18.67   0.00  75.43  37.49  55.88   1102   46.349     8    8.238   54.587
	........

Java Heap无论是哪个区的内存使用都没有超过60%，Full GC的次数也很少，只有8次，说明Java Heap本身的内存使用是正常的，4G的Java Heap空间和6.2G的Java进程内存实际占用差了2G+, 这部分的内存很大一部分肯定是不正常的，这部分内存就是俗称的“堆外内存”，其实这种说法是有误的，从进程的角度看，程序分配内存主要的来源是Stack和Heap, Java Heap只是进程Heap中的一部分，如果Java Heap之外分配内存就称之为“堆外内存”是有误的， 我觉得称之为“Java堆外内存”可能比较合适。

Java堆外内存分配很多,google一下查能看到很多Native I/O使用DirectByteBuffer问题，由于DirectByteBuffer内存分配不在Java Heap中，而是在一些Native的方法或者操作系统相关代码分配的，所以这些内存的分配不会统计到Java Heap中。 DirectByteBuffer分配的内存什么时候销毁并不简单。如果DirectByteBuffer的实例已经没有引用(reference),在Hotspot版本小于20.7的JVM上由于Hotspot的Bug回收不掉; 如果DirectByteBuffer实例晋升到了old区，只能等到Full GC才能清理掉，问题是Full GC的触发需要等到Java Heap的内存使用紧张的时候才会触发，DirectByteBuffer实例本身占用Java Heap的内存较少，但是Java Heap之外的内存可能已经申请很大。

第一个问题我在处理的时候没有发现，事后分析才知道还有这点遗漏，所以没有排查。不过我也确认过，线上的JDK没有这个BUG。关于第二个问题，如果是DirectByteBuffer实例晋升到了Old区，Java heap本身比较空闲，不会触发Full GC导致的堆外内存泄漏，如果有办法显式的触发JVM做Full GC,然后我们观察内存的使用情况，就能排除是否与第二个问题有关。jmap这个命令能做到触发JVM Full GC。命令如下：
	
	jmap -histo:live pid


jmap命令执行了几次之后，发现内存的使用并没有下降，说明不是这个问题，虽然如此，依然不能说明和DirectByteBuffer没有关系，因为DirectByteBuffer的实例如果没有晋升到Old区，实例的Reference也不为0，这个时候GC也是不能去掉的。DirectByteBuffer问题似乎没有进展，只能换个角度去查询Java Heap之外内存使用很多的问题。这个时候我使用了jstack命令， 看下JVM里面的线程到底在干些什么事情。 jstack -l pid。执行的结果发现一个可疑点，New I/O client worker这种线程的多得惊人：
	
	$grep 'New I/O client worker' jstack_13_30.log | wc -l 
	4695

并且多次执行jstack命令发现New I/O client worker的线程一直在增长。并且线程名里面的是序列号也是连续的，所以New I/O client worker这些线程实际上一直在创建，没有销毁过。 
	
	"New I/O client worker #4695-1" prio=10 tid=0x00007f080e1ec800 nid=0x32047 runnable [0x00007f085e0ed000]
	"New I/O client worker #4694-1" prio=10 tid=0x00007f080e1ec000 nid=0x32044 runnable [0x00007f085e1ee000]
	"New I/O client worker #4693-1" prio=10 tid=0x00007f080e1eb800 nid=0x32033 runnable [0x00007f085e5f2000]
	"New I/O client worker #4692-1" prio=10 tid=0x00007f0925713800 nid=0x3202f runnable [0x00007f085e7f4000]
	"New I/O client worker #4691-1" prio=10 tid=0x00007f0925712800 nid=0x32029 runnable [0x00007f07fd4ff000]
	...
	...
	...
	"New I/O client worker #7-1" prio=10 tid=0x00007f09b5250000 nid=0x233de runnable [0x0000000052e42000]
	"New I/O client worker #6-1" prio=10 tid=0x00007f09b756f800 nid=0x233d5 runnable [0x000000005263a000]
	"New I/O client worker #5-1" prio=10 tid=0x00007f09b4a16800 nid=0x233c7 runnable [0x000000005182c000]
	"New I/O client worker #4-1" prio=10 tid=0x00007f09c45d5000 nid=0x23396 runnable [0x000000004e6fb000]
	"New I/O client worker #3-1" prio=10 tid=0x00007f09b7c4c800 nid=0x2334c runnable [0x000000004d0e5000]
	"New I/O client worker #2-1" prio=10 tid=0x00007f09bc707000 nid=0x2333b runnable [0x000000004c2d7000]
	"New I/O client worker #1-1" prio=10 tid=0x00007f09b7bcb800 nid=0x23271 runnable [0x0000000049bb0000]

查看线程的执行栈，发现New I/O client worker执行的是Netty的代码。
	
	"New I/O client worker #4695-1" prio=10 tid=0x00007f080e1ec800 nid=0x32047 runnable [0x00007f085e0ed000]
	java.lang.Thread.State: RUNNABLE
        at sun.nio.ch.EPollArrayWrapper.epollWait(Native Method)
        at sun.nio.ch.EPollArrayWrapper.poll(EPollArrayWrapper.java:210)
        at sun.nio.ch.EPollSelectorImpl.doSelect(EPollSelectorImpl.java:65)
        at sun.nio.ch.SelectorImpl.lockAndDoSelect(SelectorImpl.java:69)
        - locked <0x00000006eba71db0> (a sun.nio.ch.Util$2)
        - locked <0x00000006eba71da0> (a java.util.Collections$UnmodifiableSet)
        - locked <0x00000006eba71b98> (a sun.nio.ch.EPollSelectorImpl)
        at sun.nio.ch.SelectorImpl.select(SelectorImpl.java:80)
        at org.jboss.netty.channel.socket.nio.SelectorUtil.select(SelectorUtil.java:38)
        at org.jboss.netty.channel.socket.nio.NioWorker.run(NioWorker.java:164)
        at org.jboss.netty.util.ThreadRenamingRunnable.run(ThreadRenamingRunnable.java:108)
        at org.jboss.netty.util.internal.IoWorkerRunnable.run(IoWorkerRunnable.java:46)
        at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:886)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:908)
        at java.lang.Thread.run(Thread.java:662)


锁定问题在Netty之后，我开始google关于Netty的问题，找到一篇毕玄的文章，里面是这么描述的：***对netty熟一些的同学会知道netty创建一个连接后，会将连接交给后端的NioWorker线程，由NioWorker线程来处理具体的I/O读写事件，NioWorker的线程池的线程数默认为cpu * 2，在上面的场景中创建了1w+的NioWorker线程，那只能是更前端让NioWorker产生了更多的线程数。***

***往上追朔可以看到NioWorker的线程数要创建更多的话，必然是NioClientSocketChannelFactory每次重新创建了，如每次创建连接时，NioClientSocketChannelFactory重新创建，就会直接导致每次创建的这个连接都会扔到一个新的NioWorker线程去处理。***

	
由此可见，只要找到Java代码里面调用NioClientSocketChannelFactory的地方，就能知道对于Netty的使用是否正确。我从应用代码里面搜索NioClientSocketChannelFactory关键字， 但是没有结果。由于我自己使用Java语言较少，于是请教了我们组从事过Java研发的狄卢同学，他写了一个btrace的脚本去抓取什么地方创建的线程，执行的结果能看到线程一直在创建，但是线程的名字并非是New I/O client worker开头。最后请毕玄帮忙排查，毕玄之前排查过类似问题，一眼就知道问题所在，修改下btrace脚本， 改抓取什么地方创建NioClientSocketChannelFactory。
	
	import com.sun.btrace.annotations.*;
	import com.sun.btrace.annotations.OnMethod;
	import com.sun.btrace.annotations.Self;
	
	import static com.sun.btrace.BTraceUtils.*;

	/**
	 * Created with IntelliJ IDEA.
	 * User: dilu.kxq
	 * Date: 14-8-20
	 * To change this template use File | Settings | File Templates.
	 */
	@BTrace
	public class NewThreadStack {
	    @OnMethod(clazz = "org.jboss.netty.channel.socket.nio.NioClientSocketChannelFactory", method = "<init>")
	    public static void func(@Self Object obj) {
	        print("----------");
	        jstack();
	    }
	}

执行打印出的栈如下：
	
	----------org.jboss.netty.channel.socket.nio.NioClientSocketChannelFactory.<init>(NioClientSocketChannelFactory.java:120)
	org.jboss.netty.channel.socket.nio.NioClientSocketChannelFactory.<init>(NioClientSocketChannelFactory.java:105)
	com.XXX.XXX.link.channel.websocket.WebSocketClient.prepareBootstrap(WebSocketClient.java:103)
	com.XXX.XXX.link.channel.websocket.WebSocketClient.prepareAndConnect(WebSocketClient.java:83)
	com.XXX.XXX.link.channel.websocket.WebSocketClient.connect(WebSocketClient.java:48)
	com.XXX.XXX.link.channel.ClientChannelSharedSelector.connect(ClientChannelSharedSelector.java:57)
	com.XXX.XXX.link.channel.ClientChannelSharedSelector.getChannel(ClientChannelSharedSelector.java:43)
	com.XXX.XXX.link.remoting.DynamicProxy.getChannel(DynamicProxy.java:112)
	com.XXX.XXX.link.remoting.DynamicProxy.invoke(DynamicProxy.java:98)
	com.XXX.XXX.link.remoting.DynamicProxy.invoke(DynamicProxy.java:80)
	com.XXX.XXX.link.remoting.DynamicProxy.invoke(DynamicProxy.java:62)
	$Proxy164.pollTask(Unknown Source)
	com.XXX.XXX.workflow.framework.TaskWorkerManager.execute(TaskWorkerManager.java:89)
	com.XXX.XXX.workflow.framework.worker.AbstractWorkerManager$1.run(AbstractWorkerManager.java:164)
	java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:886)
	java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:908)
	java.lang.Thread.run(Thread.java:662)

prepareBootstrap方法属于一个二方jar包，下载二方jar的源码查看实现果然如毕玄的文章所说一个链接创建了一个线程：
	
	private static ClientBootstrap prepareBootstrap(Logger logger,
		WebSocketClientUpstreamHandler wsHandler,
		SslHandler sslHandler,
		int connectTimeoutMillis) {
		ClientBootstrap bootstrap = new ClientBootstrap(new NioClientSocketChannelFactory(
			Executors.newCachedThreadPool(),
			Executors.newCachedThreadPool()));

		bootstrap.setOption("tcpNoDelay", true);
		bootstrap.setOption("reuseAddress", true);
		bootstrap.setOption("connectTimeoutMillis", connectTimeoutMillis);
		bootstrap.setOption("writeBufferHighWaterMark", 10 * 1024 * 1024);

NioClientSocketChannelFactory不应该每次都是创建，链接应该复用NioClientSocketChannelFactory这个类的实例。和二方包的作者确认过问题之后，得知新版本已经修复了这个问题，通知应用方的研发升级版本，虽然最终应用方的研发没有升级代码，而是去掉了对于这块业务的调用，问题得到解决。


总结：
---------
虽然应用方去掉了对于Netty的调用使得问题得以修复，但是，如果你细心去计算堆外内存的使用，你会发现其实线程的创建可能并不会导致Java堆外内存的泄漏很严重，因为JVM中Heap空间是所有Java线程共用的，Java线程最直接的开销就是Java线程的执行栈，HotSpot虚拟机默认的Java线程栈的大小是1MB的虚拟地址空间，实际上Java线程一般使用不到1MB的栈空间，运行了的Java线程最少使用一个页(4k)的栈空间，如果JVM crash的时候有6000个线程，按照最少的栈使用大概需要24MB内存就足够了。我不知道一般线程运行时使用的栈空间大小，所以没办法计算准确的数值。

Netty高性I/O能框架，核心功能就是使用了NIO, NIO使用的是epoll系统调用和DirectByteBuffer,所以又回到了DirectByteBuffer,最早的分析我们没有发现DirectByteBuffer有问题，但是也没有排除，事后，我在想有没有办法知道DirectByteBuffer Java堆外内存的使用，果然有大拿做了个Java程序能分析出Java堆外DirectByteBuffer的内存使用,DirectMemorySize.java, 这个程序直接分析Java进程，也能分析core文件，我在排查过程中使用gdb生成了一个core文件，所以我直接分析core文件，输出如下：
	
	$java -cp .:$JAVA_HOME/lib/sa-jdi.jar DirectMemorySize $JAVA_HOME/bin/java core.143748
	Attaching to core core.143748 from executable /opt/java//bin/java, please wait...
	Debugger attached successfully.
	Server compiler detected.
	JVM version is 20.0-b12-internal
	NIO direct memory: (in bytes)
	  reserved size = 2276.261724 MB (2386833414 bytes)
	  max size      = 3891.250000 MB (4080271360 bytes)

可以看到DirectByteBuffer最大Java对外内存使用到达3G+, 目前在使用的也有2G+, 2GJava堆外内存加上4GJava堆内存，跟top命令输出java进程占用6.2G的物理内存基本一致。
如果Netty线程创建很多不是直接导致堆外内存使用很多的原因，那么即使使用新版本的二方包能否解决Java堆外内存很多的问题呢，这要看Netty的实现，我没有去细看代码，应用方也没有升级到新版本的二方包去测试。有文档介绍Netty是使用内存池管理内存，如果一个NioClientSocketChannelFactory实例创建一个内存池，那么减少NioClientSocketChannelFactory实例，也就是减少Netty线程的创建是有效果的，如果DirectByteBuffer用于每个单独的链接，每个链接都有自己的内存，那么可能减少线程的创建不能解决Java内存堆外内存使用过多的问题。

有兴趣的读者可以再去了解下NIO的实现方式epoll的模型，其实了解epoll系统调用的同学可能对于NIO的实现方式秒懂，为什么不是一个链接一个线程，而是可以很多链接一个线程处理。故障排查归根结底，还是毕玄那句：知其因，惟手熟尔。原理性的东西理解得越多就越不需要上层代码是如何编写，底层的东西共性的实在太多，从下往上查询问题，不需要知道应用的业务代码如何实现，最重要的还是多去实践。

参考链接:
---------
1.http://hellojava.info/?p=184</br>
2.http://www.ibm.com/developerworks/cn/java/j-lo-javaio/</br>
3.http://www.infoq.com/cn/articles/netty-high-performance</br>
4.https://gist.github.com/rednaxelafx/1593521</br>
5.http://hellojava.info/?p=56</br>

