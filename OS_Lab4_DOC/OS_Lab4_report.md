##实验思考题
>Thinking 4.1 思考下面的问题，并对这两个问题谈谈你的理解：
• 子进程完全按照 fork() 之后父进程的代码执行，说明了什么？
• 但是子进程却没有执行 fork() 之前父进程的代码，又说明了什么？

&emsp;子进程获得了与父进程绝大多数的信息状态    
&emsp;子进程的PC值是fork后的指令

>Thinking 4.2 关于 fork 函数的两个返回值，下面说法正确的是： 
A、fork 在父进程中被调用两次，产生两个返回值 B、fork在两个进程中分别被调用一次，产生两个不同的返回值 
C、fork只在父进程中被调用了一次，在两个进程中各产生一个返回值 
D、fork只在子进程中被调用了一次，在两个进程中各产生一个返回值    

&emsp;C是正确的。当切换到父进程的时候，fork的返回值所对应的寄存器保存的值为子进程的id，当切换到子进程的时候，该寄存器的值被设为0，因此两个返回值不同。参考([http://blog.csdn.net/charliewangg12/article/details/38490033](http://blog.csdn.net/charliewangg12/article/details/38490033))

>Thinking 4.3 如果仔细阅读上述这一段话, 你应该可以发现, 我们并不是对所有的用户空间页都使用 duppage 进行了保护。那么究竟哪些用户空间页可以保护，哪些不可以呢，请结合 include/mmu.h 里的内存布局图谈谈你的看法。

&emsp;无效或者不可写的页面无需保护，其中，程序的代码段肯定是不可修改的，因此代码段无需保护，而data段，bss段和堆栈段是可写的，需要进行保护。参考([http://blog.csdn.net/u012317833/article/details/39160219](http://blog.csdn.net/u012317833/article/details/39160219))    
&emsp;同时，为了避免递归的写时拷贝，异常处理栈必须保证不进行保护。

>Thinking 4.4 请结合代码与示意图, 回答以下两个问题： 
1、vpt 和 vpd 宏的作用是什么, 如何使用它们？ 
2、它们出现的背景是什么? 如果让你在 lab2 中要实现同样的功能, 可以怎么写？    

1、答：    
	vpt:
		.globl vpt
	vpt:
		.word UVPT
&emsp;vpt中实际上存着UVPT的首地址，其作用相当于数组，在UVPT中找到对应的页表项即(\*vpt)[i]=UVPT[i]，返回第i个页表项。
	
		.globl vpd
	vpd:
		.word (UVPT+(UVPT>>12)*4)
&emsp;同理，vpd中存着页目录的首地址，在UVPT中查找第i项的页目录项(\*vpd)[i/PTE2PT],(这里与页目录的自映射相关)    
2、    
&emsp;出现背景：    
&emsp;&emsp;需要遍历所有的页表项，进一步判断各页表项及其对应页目录项的操作权限    
&emsp;lab2中实现：    
&emsp;&emsp;对于每个va，应该调用PPN()获取页表项索引，PPN(va),同样使用
(\*vpt)[PPN(va)]可以得到va所对应的页表项,同样使用(\*vpd)[PPD(va)/PTE2PT]可以得到va所对应的页目录项

##实验难点
&emsp;个人认为本次的核心在于理解fork()函数的执行流程，我的理解如下：    
&emsp;&emsp;	1.为父进程设置页缺失错误处理函数    
&emsp;&emsp;	2.通过调用syscall_env_alloc()函数创建子进程，并返回子进程id    
&emsp;&emsp;	3.如果2得到的id为0，则当前进程是子进程，返回0    
&emsp;&emsp;	4.调用duppage，将子进程与父进程的权限位进行映射    
&emsp;&emsp;	5.为子进程注册异常处理栈和页缺失错误处理函数，并且将子进程状态设置为就绪状态，返回子进程id    
&emsp;整体理解该流程之后，函数的具体填写指导书中大部分已经完成了。
##体会与感悟
###难度评价：★★★★☆(四颗星)    
###实验用时：20小时    
###体会与感悟：    
&emsp;&emsp;个人觉得对fork()函数的整体把握对本次实验是最重要的，不过好像指导书中总是缺少这一类整体的叙述而太过着重于函数的具体实现。
