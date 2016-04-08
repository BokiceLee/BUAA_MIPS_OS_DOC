
> Thinking 3.1 我们注意到我们把宏函数的函数体写成了 do { // ... } while(0) 的形式，而不是仅仅写成形如 {// ... } 的语句块，这样的写法好处是什么？    

####1、避免引用宏时的出错。    
&emsp;在C语言的代码风格中习惯在每个语句后加上分号，若用将宏写成形如 {// ... } 的语句块，则显然在预编译展开后的代码是有语法错误的，如下:    

	{
		func1();    
	};
而写成了 do { // ... } while(0) 的形式的代码块如果加上分号，在预编译展开后的代码如下:

	do
	{
		func1();
	} while(0);
显然没有语法错误。    
####2、可以通过在do { // ... } while(0)代码块中添加`break`语句类似实现`goto`语句的功能。而在简单的{// ... } 的语句块中添加`break`语句将产生错误。    
####3、避免空宏引起的warning。    
##1.alloc函数
&emsp;本人认为需要完成的第一个函数是`alloc`函数。    
####函数头部    
	static void * alloc(u_int n, u_int align, int clear)
####函数作用 
&emsp;由函数头部及指导书的说明可知，函数的作用如下:

1. 负责分配`n`个字节大小的内存，
2. 根据参数`align`对所需要进行分配的内存进行对齐
3. 根据`clear`是否为1对内存进行清零。    
#####分配字节
&emsp;该函数的实现首先需要确定空闲内存的起始地址，我们在`scse0_3.lds`文件中已经用`end`标签标记处已分配内存的结束地址，则该地址为空闲内存的起始地址。`pmap.c`中定义了一个静态变量`freemem`作为内存空闲地址的起始标志位，由于静态变量未初始化，所以在`alloc`函数被调用之前`freemem`的值应该为0，在该函数中我们首先将`freemem`的值设置为`end`的值，以后每次分配都以`freemem`的值为起始地址，分配结束之后将`freemem`增大到相应的值，使`freemem`保持为空闲内存的起始地址。显然在分配之前和分配之后的`freemem`之间的地址即为我们分配出来的内存。同时我们定义一个新的变量`alloc_mem`作为我们分配的内存的起始地址，这是函数的返回值。    
#####对齐内存
&emsp;在`include/types.h`中定义了这样的两个宏,    

	/* Rounding; only works for n = power of two */
	\#define ROUND(a, n)     (((((u_long)(a))+(n)-1)) & ~((n)-1))
	\#define ROUNDDOWN(a, n) (((u_long)(a)) & ~((n)-1))
&emsp;这里以第一个宏`ROUND(a,n)`进行分析，该宏接受`a`,`n`两个参数，因为`n = power of two`，设其指数为`m`，则`(n)-1`为`m`个`1`组成的二进制数，`~((n)-1)`为`m`个`0`组成的二进制数，当`a`的低`m`位不全为0时，显然`((((u_long)(a))+(n)-1))`在第`m+1`位增加1，则`(((((u_long)(a))+(n)-1)) & ~((n)-1))`的结果是`a`的第`m+1`位增加1，且低`m`位全为0，即__上对齐__。    
&emsp;同理可知宏`ROUNDDOWN(a,n)`的作用为__下对齐__。    
&emsp;显然我们在`alloc`函数中应该将内存地址上对齐，否则将导致数据的混淆。因此引用宏`ROUND(a,n)`。
#####清零
&emsp;就如上述的`ROUND`宏一样，实验代码也为我们提供了一个清零的函数，在`init/init.c`中定义了函数:    
	
	void bzero(void *b, size_t len)
	{
        void *max;

        max = b + len;

        //printf("init.c:\tzero from %x to %x\n",(int)b,(int)max);

        // zero machine words while possible

        while (b + 3 < max)
        {
                *(int *)b = 0;
                b+=4;
        }
        
        // finish remaining 0-3 bytes
        while (b < max)
        {
                *(char *)b++ = 0;
        }
        
	}

&emsp;该函数相对来说比较简单，借助注释可以看出其作用为将以指针`b`所指向的内存地址起的`len`个字节的内存初始化为0。    
&emsp;至此我们的`alloc`函数就算完成了。贴出完整代码如下:    

	static void * alloc(u_int n, u_int align, int clear)	
	{
	        extern char end[];
	
	        u_long alloced_mem;
	
	        // initialize freemem if this is the first time
	
	        if(freemem==0){
	        	freemem=(u_long)end;
	        }
            // Your code here:
	        //      Step 1: round freemem up to be aligned properly

	        	freemem=ROUND(freemem,align);

    	    //      Step 2: save current value of freemem as allocated chunk

	        	alloced_mem=freemem;

        	//      Step 3: increase freemem to record allocation

	        	freemem=freemem+n;

        	//      Step 4: clear allocated chunk if necessary

	        	if(clear){
	                bzero(alloced_mem,n);
	        	}
        	//      Step 5: return allocated chunk

        return (void *)alloced_mem;

	}

既然我们已经能够分配指定字节数的内存了，就在这个基础上编写粒度更大的分配函数，即以页(4k)为单位分配内存。
##2、page\_init函数
首先是page\_init函数，这也是Exercise2.2的要求。
> Exercise 2.2 完成 page\_init 函数，使用 include/queue.h 中定义的宏函数将未分配的物理页加入到空闲链表 page\_free\_list 中去。思考如何区分已分配的内存块和未分配的内存块，并注意内核可用的物理内存上限    

&emsp;虽然该函数的注释里面写了一大堆，但是感觉最重要的只有两点，    
&emsp;1. 将所有的页面添加到链表中去；    
&emsp;2. 标记第一个页面为已分配，其他所有页面为未分配。    
&emsp;首先我们的链表在哪里？在`mm/pmap.c`即本文件的开头定义了这样一个变量，

	static struct Page_list page_free_list; /* Free list of physical pages */
&emsp;从注释可以看出这个链表就是我们的页面链表，接下来我们的操作将在其上面进行。所以可以先看一下类型`Page_list`的定义：    

	struct Page_list { struct Page *lh_first; };
&emsp;再看一下结构体`Page`的定义：  

	struct Page {
 		Page_LIST_entry_t pp_link;
 		u_short pp_ref;
	};
&emsp;再看看`Page_LIST_entry_t`:

	typedef struct { struct Page *le_next; struct Page **le_prev; } Page_LIST_entry_t;

#(未完带续，先攀岩去==







 
 
 
 
 
 












>Exercise 2.3 完成 mm/pmap.c 中的 page\_alloc 和 page\_free 函数，基于空闲内存链表page\_free\_list ，以页为单位进行物理内存的管理。    

&emsp;