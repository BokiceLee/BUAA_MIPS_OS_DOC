##1.实验思考题
>Thinking 3.1 为什么我们在构造空闲进程链表时使用了逆序插入的方式？     

其实这里并非真正意义上的逆序，我认为关键看你用的是什么宏，我觉得写指导书的大大应该是使用`LIST_INSERT_HEAD`这个宏，为了使每次从链表中取得的空闲进程控制块是按0到NENV的次序取，显然需要“逆序”，如果使用其他宏的话，这里的“逆序”就不对了。

>Thinking 3.2 思考 env\_setup\_vm 函数：
>
>• 第三点注释中的问题: 为什么我们要执行pgdir[i] = boot_pgdir[i]这个 赋值操作？换种说法，我们为什么要使用boot\_pgdir作为一部分模板？(提 示:mips 虚拟空间布局)
>
>• UTOP 和 ULIM 的含义分别是什么，在 UTOP 到 ULIM 的区域与其他用户区 相比有什么最大的区别？
>
>• (选做) 我们为什么要让pgdir[PDX(UVPT)]=env\_cr3?(提示: 结合系统自映射 机制)

&emsp;首先分析该函数，函数第一步分配一个页表作为进程控制块的页目录，pgdir该页目录的虚拟地址，接下来将页目录低于PDX(UTOP)的每一项初始化为0，而每个进程控制块高于PDX(UTOP)部分的项的值都是相等的，且与boot_pgdir一一对应。这里的boot\_pgdir为页目录。
而VPT和UVPT映射了进程控制块自身的页表，VPT和UVPT分别拥有不同的权限，相对于VPT，UVPT可读可写。     
&emsp;1.对于每个进程，随时都可能陷入内核态，而陷入内核态并并不切换CR3寄存器，即运行的仍然是当前进程，在这种进程下，该进程为了执行特定的指令，必须知道指令的地址，因此需要拷贝一份页目录给pgdir。    
&emsp;2.UTOP是用户态下可读写的最高地址，ULIM是内核地址空间是内核态可读写的最低地址，从UTOP到ULIM区域相比其他区域的最大区别是内核和用户对于地址范围[UTOP, ULIM]有着相同的访问权限，那就是可以读取但是不可以写入。这一个部分的地址空间通常被用于把一些只读的内核数据结构暴露给用户地址空间的代码。    
&emsp;3.根据系统的自映射机制，进程控制块中的页目录当然需要保存着自身页目录的地址啦。

>思考 user_data 这个参数的作用。没有这个参数可不可以？为什么？（如果你能说明哪些应用场景中可能会应用这种设计就更好了。可以举一个实际的库中的例子）   

&emsp;不可以没有该参数，因为load\_icode\_map函数需要使用到该参数。

>Thinking 3.4 思考上面这一段话，并根据自己在lab2 中的理解，回答：
>
>• 我们这里出现的“指令位置” 的概念，你认为该概念是针对虚拟空间，还是物理内存所定义的呢？
>
>• 你觉得entry_point其值对于每个进程是否一样？该如何理解这种统一或不 同？
>
>• 从布局图中找到你认为最有可能是entry_point的值

&emsp;1.这里的指令位置应该是虚拟地址,    
&emsp;2.entry\_point是镜像运行的入口点地址，对于加载到内存中的镜像来说，它们的值应该是相同的且应该为已知值。这样当CPU开始运行时，PC便可从该确定的地址开始读取指令。    
&emsp;3.entry_point的值应该是UTEXT，此处为用户程序的起点。

>Thinking 3.5 思考一下，要保存的进程上下文中的env_tf.pc的值应该设置为多少？ 为什么要这样设置？

应该设置为下一个进程控制块的协处理器中保存的pc值，即cp0_epc，原因是当进程发生切换时，cpu发生了中断，中断处理程序将当前pc值保存到epc中，此处我们将上下文中的pc值设置为epc中保存的值，保证了该进程恢复运行时能够紧接着上一条指令继续被执行。

>Thinking 3.6 思考 TIMESTACK 的含义，并找出相关语句与证明来回答以下关于 TIMESTACK 的问题：  
• 请给出一个你认为合适的 TIMESTACK 的定义  
• 请为你的定义在实验中找出合适的代码段作为证据 (请对代码段进行分析)  
• 思考 TIMESTACK 和第 18 行的 KERNEL_SP 的含义有何不同

&emsp;1.TIMESTACK应该是时钟中断发生时寄存器值的存储区的起始地址   
&emsp;2.当程序发生中断的时候，将中断向量保存到CP0\_CAUSE中并跳转到中断处理函数，时钟中断的处理函数为handle_int函数，在lib/genex.S中有以下这样的一段代码，


	NESTED(handle_int, TF_SIZE, sp)
	.set    noat
	
	//1: j 1b
	nop
	
	SAVE_ALL
	CLI
	.set    at
	mfc0    t0, CP0_CAUSE
	mfc0    t2, CP0_STATUS
	and     t0, t2
	
	andi    t1, t0, STATUSF_IP4
	bnez    t1, timer_irq
	nop
	END(handle_int)
    
&emsp;该代码的关键在于调用了另一个函数，SAVE_ALL，该函数的定义比较长，如下：

	.macro SAVE_ALL
                mfc0    k0,CP0_STATUS                
                sll             k0,3      /* extract cu0 bit */ 
                bltz    k0,1f                            
                nop       
                /*                                       
                 * Called from user mode, new stack      
                 */                                      
                move    k0,sp
                get_sp
                move    k1,sp                     
                subu    sp,k1,TF_SIZE                    
                sw      k0,TF_REG29(sp)                  
                sw      $2,TF_REG2(sp)                   
                mfc0    v0,CP0_STATUS                    
                sw      v0,TF_STATUS(sp)                 
                mfc0    v0,CP0_CAUSE                     
				mfc0    v0,CP0_CAUSE                    
                sw      v0,TF_CAUSE(sp)                  
                mfc0    v0,CP0_EPC                       
                sw      v0,TF_EPC(sp)
                mfc0    v0, CP0_BADVADDR
                sw      v0, TF_BADVADDR(sp)            
                mfhi    v0                             
                sw      v0,TF_HI(sp)                   
                mflo    v0                             
                sw      v0,TF_LO(sp)                   
                sw      $0,TF_REG0(sp)
                sw      $1,TF_REG1(sp)                   
                sw    	$2,TF_REG2(sp)                   
                sw      $3,TF_REG3(sp)                   
                sw      $4,TF_REG4(sp)                   
                sw      $5,TF_REG5(sp)                   
                sw      $6,TF_REG6(sp)                   
                sw      $7,TF_REG7(sp)                   
                sw      $8,TF_REG8(sp)                   
                sw      $9,TF_REG9(sp)                   
                sw      $10,TF_REG10(sp)                 
		 		sw      $11,TF_REG11(sp)                 
                sw      $12,TF_REG12(sp)                 
                sw      $13,TF_REG13(sp)                 
                sw      $14,TF_REG14(sp)                 
                sw      $15,TF_REG15(sp)                 
                sw      $16,TF_REG16(sp)                 
                sw      $17,TF_REG17(sp)                 
                sw      $18,TF_REG18(sp)                 
                sw      $19,TF_REG19(sp)                 
                sw      $20,TF_REG20(sp)                 
                sw      $21,TF_REG21(sp)                 
                sw      $22,TF_REG22(sp)                 
                sw      $24,TF_REG24(sp)                 
                sw      $25,TF_REG25(sp)                 
                sw      $26,TF_REG26(sp)                               
                sw      $27,TF_REG27(sp)                                 
                sw      $28,TF_REG28(sp)                 
                sw      $30,TF_REG30(sp)                 
                sw      $31,TF_REG31(sp)
	.endm

	.macro get_sp
        mfc0    k1, CP0_CAUSE
        andi    k1, 0x107C
        xori    k1, 0x1000
        bnez    k1, 1f
        nop
        li      sp, 0x82000000
        j       2f
        nop
	1:
        bltz    sp, 2f
        nop
        lw      sp, KERNEL_SP
        nop

	2:      nop
上述函数的流程首先是将sp寄存器的值设为0x82000000，接着将32个寄存器的值全部保存到0x82000000起的地址上。这里我们知道0x82000000就是TIMESTACK的值，因此可知，当发生时钟中断时，32个寄存器的值均被保存到以TIMESTACK为起的地址上。  
&emsp;3.TIMESTACK应该是时钟中断发生时寄存器值的存储区，而KERNEL\_SP应该是系统调用后的存储区，KERNEL\_SP只有在系统调用之后才会发生改变。

>Thinking 3.7 思考一下你的调度程序，这种调度方式由于某种不可避免的缺陷而造成对进程的不公平。
• 这种不公平是如何产生的？
• 如果实验确定只运行两个进程，你如何改进可以降低这种不公平？    

&emsp;1.当进程由阻塞的状态变为就绪的状态的顺序跟进程控制块envs的索引顺序不一致时，很有可能产生不公平，例如，当前有3个进程，envs[1],envs[2],envs[3]，正在运行的进程是envs[3]，并且envs[1],envs[2]均为阻塞状态，在envs[3]的时间片结束之前，envs[2],envs[1]的状态相继由阻塞变为就绪，由于进程调度通过索引进行遍历，因此两个进程的执行顺序为envs[1],envs[2]，产生了不公平。该调度方式并非严格意义上的先来先服务策略。   
&emsp;2.在函数中添加一个静态变量m，初始值为0，当进程i从阻塞状态切换为就绪状态是，如果m的值为0，则将m的值设为i，当进程j从阻塞状态切换为就绪状态是，如果m的值为0，则将m的值设为j，调度进程运行的时候，首先调度m所对应的进程，同时将m值设为0，当该进程执行完毕之后，再调度另一进程。如此可使进程的调度按照先来先服务原则进行。

##2.实验难点
&emsp;我认为镜像加载是本次实验最大的难点，其主要涉及load\_elf()函数，该函数主要通过对elf文件的分析得出每个segment的大小和地址，并调用load\_icode\_mapper()函数将对应的segment加载到相应的内存地址

####load\_icode\_mapper()执行过程分析：
&emsp;这里是本实验最大的坑点，在前置条件中说了

	* Pre-Condition:
	*   va aligned 4KB and bin can't be NULL.    


&emsp;我们跳转到调用这个函数的load_elf()中看一看，

	int load_elf(u_char *binary, int size, u_long *entry_point, void *user_data,int (*map)(u_long va, u_int32_t sgsize,u_char *bin, u_int32_t bin_size, void *user_data))
	{
        Elf32_Ehdr *ehdr = (Elf32_Ehdr *)binary;
        Elf32_Phdr *phdr = NULL;
        /* As a loader, we just care about segment,
         * so we just parse program headers.
         */
        u_char *ptr_ph_table = NULL;
        Elf32_Half ph_entry_count;
        Elf32_Half ph_entry_size;
        int r;

        // check whether `binary` is a ELF file.
        if (size < 4 || !is_elf_format(binary)) {
                return -1;
        }

        ptr_ph_table = binary + ehdr->e_phoff;
		 ph_entry_count = ehdr->e_phnum;
        ph_entry_size = ehdr->e_phentsize;

        while (ph_entry_count--) {
                phdr = (Elf32_Phdr *)ptr_ph_table;

                if (phdr->p_type == PT_LOAD) {
				/*
				 *如果检测到可加载的段(text/data/bss/...)，则调用map函数进行加载
				 *这里的map函数应是我们写的load_icode_mapper()函数
				 */	
                        r = map(phdr->p_vaddr, phdr->p_memsz,binary + phdr->p_offset, phdr->p_filesz, user_data);
                        if (r < 0) {
                                return r;
                        }
                }
                ptr_ph_table += ph_entry_size;
        }
        *entry_point = ehdr->e_entry;
        return 0;
}

&emsp;可以看出，但是实际上传到map函数中的phdr->p\_vaddr并没有进行任何处理，即load\_icode\_mapper()函数接收到的va值并不能保证4kB对齐。    
在理想的情况下，即va是4kB对齐的时候，加载镜像到内存的过程应该如下：

<img src=".\offset0.bmp" width="300" height="250" alt="offset0">    

&emsp;若va未对齐，如果照常加载，加载镜像的过程同上：

<img src=".\offset1.bmp" width="300" height="250" alt="offset0">

&emsp;当进程开始运行的时候，程序的入口所对应的并不是我们所想象的入口，因为此时对应着图中offset的下部分，则将产生不可预料的结果。
经过与各位大神交流，认为比较科学的处理方法应该如下，即先分配一个页面将前面未对齐的部分加载进来，然后后面的部分就可以当做对其处理了。(如下图，图中一个矩形代表一个页面，下图只表示处理未对齐部分的过程)

<img src=".\offset2.bmp" width="300" height="250" alt="offset0">

##3.体会与感悟
####难度评价: ★★★★☆(四颗星)    
####实验用时:大约30个小时    
####体会与感悟:
&emsp;lab3前半部分跟lab2比较相似，写起来还是感觉没有什么难度的，但是后面关于进程创建和运行特别是镜像的加载这一部分明显就难多了。