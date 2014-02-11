---
layout: post
title: "C语言宏对于调试的影响"
date: 2014-02-10 20:51
comments: true
categories: C macro 函数 gcc gdb 编译优化
---

最近在写一个nginx的模块，过程中遇到一个小问题，感受到了C语言宏跟函数的区别，记录下来。

问题现象:
---------
先看一段相关代码
    
    
    393    ngx_http_stylecombine_body_filter(ngx_http_request_t *r, ngx_chain_t *in)   
    394	   {                                                                   
    395	    int                   rc;                                       
    396	    ngx_chain_t                    out;
    397	    ngx_http_stylecombine_ctx_t  *ctx;                                      
    398	                                                                       

    399	    ctx = ngx_http_get_module_ctx(r, ngx_http_stylecombine_filter_module);  
    400	                                                                    
    401	    if (ctx == NULL || r->header_only) {               
    402	        return ngx_http_next_body_filter(r, in);                    
    403	    } 

399行通过ngx_http_get_module_ctx获取ctx,401行if语句判断ctx是否为空。在调试的过程中发现402行的ngx_http_next_body_filter总是会执行到，ngx_http_stylecombine_body_filter函数提前返回。

很显然。要么ctx== NULL,要么就是r->header_only不为0,马上启动gdb，下断点在ngx_http_stylecombine_body_filter,运行之。程序停止在393行，单步执行到401行，通过print命令打印出变量ctx和r->header_only, 如下所示：

    (gdb) p ctx
    $29 = (ngx_http_stylecombine_ctx_t *) 0x989680
    (gdb) p r->header_only
    $30 = 0

当天调试得比较晚，比较粗心没发现其中的问题。 乍看之下401行的if语句不应该为真， ctx有值，r->header_only也为0, 那为什么402行还是会被执行呢？

问题分析:
---------
遇到这种问题，我常用的方法是先使用gdb的反汇编功能，然后指令级单步跟踪。(BTW：虽然我的汇编不好，马马虎虎能读反汇编后的结果)下面是ngx_http_stylecombine_body_filter反汇编片段：

    (gdb) disassemble 
    Dump of assembler code for function ngx_http_stylecombine_body_filter:
    0x0000000000487b8b <ngx_http_stylecombine_body_filter+0>:    push   %r15
    0x0000000000487b8d <ngx_http_stylecombine_body_filter+2>:	push   %r14
    0x0000000000487b8f <ngx_http_stylecombine_body_filter+4>:	push   %r13
    0x0000000000487b91 <ngx_http_stylecombine_body_filter+6>:	push   %r12
    0x0000000000487b93 <ngx_http_stylecombine_body_filter+8>:	push   %rbp
    0x0000000000487b94 <ngx_http_stylecombine_body_filter+9>:	push   %rbx
    0x0000000000487b95 <ngx_http_stylecombine_body_filter+10>:	sub    $0x68,%rsp
    0x0000000000487b99 <ngx_http_stylecombine_body_filter+14>:	mov    %rdi,%r14
    0x0000000000487b9c <ngx_http_stylecombine_body_filter+17>:	mov    %rsi,%r13
    0x0000000000487b9f <ngx_http_stylecombine_body_filter+20>:	mov    0x10(%rdi),%rdx
    0x0000000000487ba3 <ngx_http_stylecombine_body_filter+24>:	mov       0x234c76(%rip),%rax        # 0x6bc820 <ngx_http_stylecombine_filter_module>
    0x0000000000487baa <ngx_http_stylecombine_body_filter+31>:	mov    (%rdx,%rax,8),%rax
    0x0000000000487bae <ngx_http_stylecombine_body_filter+35>:	test   %rax,%rax
    0x0000000000487bb1 <ngx_http_stylecombine_body_filter+38>:	je     0x487bbc <ngx_http_stylecombine_body_filter+49>
    0x0000000000487bb3 <ngx_http_stylecombine_body_filter+40>:	testb  $0x2,0x468(%rdi)
    0x0000000000487bba <ngx_http_stylecombine_body_filter+47>:	je     0x487bd0 <ngx_http_stylecombine_body_filter+69>
    0x0000000000487bbc <ngx_http_stylecombine_body_filter+49>:	mov    %r13,%rsi
    0x0000000000487bbf <ngx_http_stylecombine_body_filter+52>:	mov    %r14,%rdi
    0x0000000000487bc2 <ngx_http_stylecombine_body_filter+55>:	callq  *0x24f140(%rip)        # 0x6d6d08 <ngx_http_next_body_filter>
    0x0000000000487bc8 <ngx_http_stylecombine_body_filter+61>:	mov    %rax,%rdx
    0x0000000000487bcb <ngx_http_stylecombine_body_filter+64>:	jmpq   0x488175 <ngx_http_stylecombine_body_filter+1514>
    0x0000000000487bd0 <ngx_http_stylecombine_body_filter+69>:	mov    %rax,(%rsp)
    0x0000000000487bd4 <ngx_http_stylecombine_body_filter+73>:	mov    0x18(%rax),%rax
    0x0000000000487bd8 <ngx_http_stylecombine_body_filter+77>:	cmp    $0x1,%rax
    0x0000000000487bdc <ngx_http_stylecombine_body_filter+81>:	je     0x487c49 <ngx_http_stylecombine_body_filter+190>
    0x0000000000487bde <ngx_http_stylecombine_body_filter+83>:	cmp    $0x1,%rax
    0x0000000000487be2 <ngx_http_stylecombine_body_filter+87>:	jb     0x487bfd <ngx_http_stylecombine_body_filter+114>
    0x0000000000487be4 <ngx_http_stylecombine_body_filter+89>:	cmp    $0x2,%rax
    0x0000000000487be8 <ngx_http_stylecombine_body_filter+93>:	je     0x487d71 <ngx_http_stylecombine_body_filter+486>
    0x0000000000487bee <ngx_http_stylecombine_body_filter+99>:	cmp    $0x3,%rax
    0x0000000000487bf2 <ngx_http_stylecombine_body_filter+103>:	jne    0x4880c5 <ngx_http_stylecombine_body_filter+1338>
    0x0000000000487bf8 <ngx_http_stylecombine_body_filter+109>:	jmpq   0x4880b4 <ngx_http_stylecombine_body_filter+1321>
    ---Type <return> to continue, or q <return> to quit---quit
    
地址0x0000000000487bbc \<ngx_http_stylecombine_body_filter+49\>开始到0x0000000000487bc2 \<ngx_http_stylecombine_body_filter+55\>之间有3条指令，前两条指令是给第3条调用指令传递的参数。x86_64的ABI规定低于6个参数的函数默认使用寄存器传参，寄存器传参顺序是rdi,rsi,rdx,rcx,r8和r9。也就是rdi是ngx_http_next_body_filter的第一个参数r, rsi是ngx_http_next_body_filter的第二个参数in。 r13,r14两个寄存器的内容来源于0x0000000000487b99和0x0000000000487b9c处的两条指令。ngx_http_stylecombine_body_filter和ngx_http_next_body_filter是一样的参数形式，回头去看0x0000000000487b99和0x0000000000487b9c处的两条指令，很容易发现ngx_http_stylecombine_body_filter函数在使用r和in之前各保存到了一份在r14和r13两个寄存器中。

从上面反汇编的结果可以看出，如果要执行到ngx_http_next_body_filter退出，程序的控制流必定是会流经ngx_http_stylecombine_body_filter+49处，上面反汇编的代码只有一处就是ngx_http_stylecombine_body_filter+38，而ngx_http_stylecombine_body_filter+38处的je跳转指令取决于前一条指令test   %rax,%rax对于EFLAG寄存器Z标志位的影响，如果rax为0,Z标志位为0,je指令跳转到ngx_http_stylecombine_body_filter+49。 所以只要单步执行下来到ngx_http_stylecombine_body_filter+35处看下rax的值，就能确定是否跳转。通过gdb指令单步跟踪发现rax确实为0:
    
    (gdb) p $rax
    $28 = 0
这就解释了为什么ngx_http_stylecombine_body_filter为什么总是提前执行ngx_http_next_body_filter退出。问题的重点是**rax代表什么?为什么会是0?**

问题的本质:
----------
要追究问题的本质需要知道C语句

    ctx = ngx_http_get_module_ctx(r, ngx_http_stylecombine_filter_module);
经过gcc编译之后到底翻译成了什么？
C语言函数ngx_http_stylecombine_body_filter反汇编之后的第一条指令到ngx_http_stylecombine_body_filter+35处指令并不多，我们可以一一对照下分析下，

    0x0000000000487b8b <ngx_http_stylecombine_body_filter+0>:    push   %r15
    0x0000000000487b8d <ngx_http_stylecombine_body_filter+2>:	push   %r14
    0x0000000000487b8f <ngx_http_stylecombine_body_filter+4>:	push   %r13
    0x0000000000487b91 <ngx_http_stylecombine_body_filter+6>:	push   %r12
    0x0000000000487b93 <ngx_http_stylecombine_body_filter+8>:	push   %rbp
    0x0000000000487b94 <ngx_http_stylecombine_body_filter+9>:	push   %rbx
    ;以上是保存寄存器的值，这些寄存器在本次函数调用中有可能被使用到，
    ;需要保护起来，在函数退出的时候通过pop操作恢复回去。
    0x0000000000487b95 <ngx_http_stylecombine_body_filter+10>:	sub    $0x68,%rsp
    ;这条指令开辟栈上空间用于保存函数的局部变量。
    ;也就是对于到这三条C语句 （rc， out和ctx指针占用的字节数在比0x68要小，
    ;		实际上gcc会可能会对齐局部变量加快CPU对于变量的访问速度，
    ;		另外对于整个函数使用的局部变量都会一次性分配好，ngx_http_stylecombine_body_filter有些局部变量在后面声明。）
    ; 395        int                   rc;                                       
    ; 396	     ngx_chain_t                    out;
    ; 397	     ngx_http_stylecombine_ctx_t  *ctx; 
    
    0x0000000000487b99 <ngx_http_stylecombine_body_filter+14>:	mov    %rdi,%r14
    0x0000000000487b9c <ngx_http_stylecombine_body_filter+17>:	mov    %rsi,%r13
    ;前文说到，这两条指令保存r和in到r14和r13寄存器。

结合前面的分析，也就是剩下三条指令和一句C语句需要分析。
    
    0x0000000000487b9f <ngx_http_stylecombine_body_filter+20>:    mov    0x10(%rdi),%rdx
    0x0000000000487ba3 <ngx_http_stylecombine_body_filter+24>:	mov    0x234c76(%rip),%rax        # 0x6bc820 <ngx_http_stylecombine_filter_module>
    0x0000000000487baa <ngx_http_stylecombine_body_filter+31>:	mov    (%rdx,%rax,8),%rax;
很显然这三条指令是对应到ctx = ngx_http_get_module_ctx(r, ngx_http_stylecombine_filter_module);(后面可以看到这么说不全对)

当我调试到此处的时候，发现ngx_http_get_module_ctx其实是一个宏，这个宏的定义如下：

    #define ngx_http_get_module_ctx(r, module)  (r)->ctx[module.ctx_index]
了解C语言的同学一定知道ctx是r的一个成员，对于汇编来说其实就是一个地址，

    mov    0x10(%rdi),%rdx   
    ;rdi寄存器保存的是r的地址，0x10(%rdi)表示r的地址开始加16(0x10)
    ;由此可知ctx成员在r结构体中的偏移是16.
 module.ctx_index用于在ctx中做索引，通过ngx_module_t的定义，我们知道ctx_index是ngx_module_t的首成员，
    
    struct ngx_module_s {
    ngx_uint_t            ctx_index;
    ngx_uint_t            index;
    .....

所以module和module.ctx_index的地址是相同的，这也是为什么gdb提示的是# 0x6bc820 <ngx_http_stylecombine_filter_module>

    mov    0x234c76(%rip),%rax        # 0x6bc820 <ngx_http_stylecombine_filter_module>
    
有了r->ctx的地址(保存在rdx)，和索引值module.ctx_index(保存在rax)，接下来就是去取索引的内容：

    mov    (%rdx,%rax,8),%rax
    ;rdx + rax * 8
    ;x86_64的地址长度为8个字节。

到此为止知道ngx_http_stylecombine_body_filter+35指令之前rax寄存器保存内容的来历。就是说\(r\)->ctx[module.ctx_index]确实是为0, if语句的跳转没有出错。

等等，貌似少了什么东西，**为什么在前面通过gdb打印ctx的值不为0，按照C语言的语句ctx = ngx_http_get_module_ctx(r, ngx_http_stylecombine_filter_module);ctx应该要是0才对!?** 我认为这是由于gcc对C语言的宏进行展开之后，通过数据流分析和优化能发现if语句里面要判断的值其实是\(r\)->ctx[module.ctx_index],不是局部变量ctx，从反汇编的结果也可以看出在ngx_http_stylecombine_body_filter+35处的test语句之前并没有ctx的赋值操作,如果ngx_http_get_module_ctx是一个函数而不是一个宏，ctx应该会是0(前提是编译器没有过度优化)。

这就能解释ctx为什么不为0,gdb打印出来ctx的值实际上是一个未初始化的栈上的值，也就是一个野值(0x989680)，当天调试较晚困了没有发现。同时也解释了我前面说从ngx_http_stylecombine_body_filter+20开始的三条mov指令对应到ctx = ngx_http_get_module_ctx(r, ngx_http_stylecombine_filter_module)的说法其实是不全对的，因为ctx的赋值操作没有在这3条指令中。(*ps:欢迎大家和我交流^-^*)

