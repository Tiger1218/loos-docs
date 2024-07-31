# core 
## kbd
### struct breakpoint *breakpoint_malloc()
为新断点分配内存；
### struct breakpoint *breakpoint_find(uint64_t addr)
在以分配内存的断点数组中查找与地址 addr 匹配的地址；
### void breakpoint_delete(uint64_t addr)
删除地址为 addr 的断点， 并将该断点的使用位置为 0；
### void breakpoint_add(uint64_t addr) 
在地址 addr 处 添加一个新的断点；
### void breakpoint_clear()
清除所有断点；
### void breakpoint_print()
打印所有已使用的断点信息；
### void cmd_parse(char *cmd)
解析命令行命令 cmd 并将其分割成多个参数；
### int uatoi(const char *str, uint64_t *result)
将字符串 string 转化为 64 位整数 result
### void expr(char *str, uint64_t *result)
解析计算型表达式字符串 str 并计算器结果；
### void cmd_handler() 
命令调试器， 可以输出命令提示符提示用户输入命令， 读取， 解析， 处理用户输入的命令， 若用户输入退出命令， 则结束调试会话；
### void break_handler()
当遇到断点异常时贝雕用的调试器， 获取异常地址和 trapframe ， 打印调试信息并调用命令调试器， 最后删除断；

## proc
### struct proc *my_proc()
获取指向当前进程的指针；
### char* getcwd(char* buf, size_t size)
获取当前工作目录的路径，并将其前 size 位复制到缓冲区 buf 中；
### int get_proc_id()
获取当前进程的 id;
### int set_proc_id(int pid) 
将当前进程的 id 置为 pid；
### int chdir(char *path)
更改当前工作目录为 path ；
### void proc_init()
初始化进程数组 ；
### void clone_proc(struct proc* parent)
通过 parent 进程的结构体复制一个子进程， 并初始化子进程 ；
### void kernel_proc_init() 
初始化一个进程（为进程分配内存和页表）；
### struct proc *proc_alloc()
为一个新的进程结构体分配内存和页表；
### void proc_free(struct proc *proc)
释放一个进程结构体（将该进程的allocs置为UNUSED）；
### int exit(int _)
退出当前进程（记录日志并切换其他进程）；
### int in_kernel_memory(uintptr_t start, uintptr_t end)
判断 start 到 end 地址是否在内核内存空间内， 若在则返回1， 否则返回 0 ；
### void swtch_from_kernel()
确保从内核模式切换到用户模式或其他模式时，内存管理单元（MMU）和相关硬件状态被正确配置 ;
### void swtch_back_to_kernel()
确保从用户模式切换回内核模式时，内存管理单元（MMU）和相关硬件状态被正确重置和配置 ；
### void switch_to(struct proc * before, struct proc *after)
实现从进程  before 到进程 after 之间的切换 ；

## syscall
### struct trapframe *get_trapframe()
获取指向当前陷阱帧的指针
### uint64_t argraw(int n)
获取当前陷阱帧的第 n 个参数；
### int argint(int n, uint64_t *ip) 
获取当前陷阱帧的第 n 个整数参数；
### int argaddr(int n, uint64_t *ip)
获取当前陷阱帧的第 n 个地址参数；
### int argstr(int n, char *buf, int max)
获取当前陷阱帧的第 n 个字符串参数， 将其复制到缓冲区 buf 中， 最多复制 max 位；
### uint64_t sys_openat(void)
实现 openat 系统调用， 打开一个文件；
### uint64_t sys_close(void)
实现 close 系统调用， 关闭一个文件；
### uint64_t sys_read(void)
实现 read 系统调用， 从文件描述符中读取数据；
### uint64_t sys_write(void)
实现 write 系统调用， 向文件描述符中写入数据；
### uint64_t sys_lseek(void)    
实现 sleek 系统调用， 改变文件描述符的偏移量；
### uint64_t sys_fstat()
实现 fstat 系统调用， 获取文件状态；
### uint64_t sys_fcntl()
实现 fcntl 系统调用， 执行文件控制操作；
### uint64_t sys_brk() 
实现 brk 系统调用， 改变进程的堆顶；
### uint64_t sys_mmap()
实现 sys_mmap 系统调用， 映射文件或设备到内存；
### uint64_t sys_writev()
实现 sys_writev 系统调用， 向文件描述符写入多个数据缓冲区；
### uint64_t sys_newuname()
实现 sys_newuname 系统调用， 获取系统信息；
### uint64_t sys_read_link_at()
实现 sys_read_link_at 系统调用， 读取符号链接的内容；
### uint64_t sys_getrandom()
实现 sys_getrandom 系统调用， 填充指定大小的缓冲区，生成随机数；
### uint64_t sys_rt_sigaction()
实现 rt_sigaction 系统调用， 设置或获取信号处理动作;
### uint64_t sys_setitimer()
实现 setitimer 系统调用， 设置定时器， 用于定时触发信号;
### uint64_t sys_exit()
实现 exit 系统调用， 退出当前进程， 并返回一个状态值;
### uint64_t sys_getcwd()
实现 getcwd 系统调用， 获取当前工作目录的路径;
### uint64_t sys_chdir()
实现 chdir 系统调用， 改变当前工作目录;
### uint64_t sys_dup3() 
实现 dup3 系统调用， 复制文件描述符， 并设置特定的标志;
### uint64_t sys_getdents64()
实现 getdents64 系统调用， 读取目录项信息;
### uint64_t sys_mkdirat()
实现 mkdirat 系统调用， 创建一个新目录;
### uint64_t sys_dup()
实现 dup 系统调用， 复制文件描述符;
### uint64_t sys_unlinkat()
实现 unlinkat 系统调用， 删除或解除链接一个文件;
### static uint64_t (*syscalls[])(void)
系统调用总结；
### void clock_handler() 
周期性地检查是否到达了预定的时间点， 并在到达该时间点时执行相应的回调函数；
### void syscall_handler()
处理系统调用请求， 将用户空间的系统调用请求映射到内核空间的相应处理函数， 并在处理完成后将结果返回给用户空间；

# driver
## ahci
### int ahic_probe_port(HBA_MEM *abar)
遍历所有可能的SATA端口，找到并返回第二个活动的端口的索引。 如果只找到一个或没有找到， 则返回 -1 ；
### int find_cmdslot(HBA_MEM *abar, volatile HBA_PORT *port)
在给定的HBA端口中找到一个空闲的 command list 槽位, 如果能找到则返回该索引， 否则返回 -1 ；
### int ahci_read(HBA_MEM *abar, int port_num, uint32_t startl, uint32_t starth, uint32_t count, uint8_t *buf)
通过AHCI接口从SATA设备读取数据, 读取成功则返回 0 ， 否则返回 -1 ；

## disk
### int ramdisk_read(void *buf, int offset, int len)
从 ram 盘偏移量为 offset 处 复制 len 字节数据到缓冲区 buf ；
### int ramdisk_write(void *buf, int offset, int len)
从缓冲区 buf 复制 len 字节数据到 RAM 盘偏移量为 offset 处 ；
### void disk_init()
初始化磁盘， 使用循环读取磁盘上多个扇区的数据， 并将他们复制到 RAM 盘中， 当达到 RAM 盘大小限制或者读取玩磁盘上的可用扇区时， 循环结束 ； 

## shutdown
### void shutdown()
执行系统关机操作 ；

## uart
### void putch(char c)
向串行端口发送单个字符 c ；
### void putstr(char *s)
向串行端口发送字符串 s ；
### int getch()
从串行端口接受单个字符 c ；
### char *getstr()
从串行端口接受字符串 s ；

# fs
## fs
### int open(const char *pathname, int flags, ...)
打开 pathname 指定路径的文件， 并返回一个文件描述符；
### int openat(int dirfd, const char *pathname, int flags)
打开 pathname 指定路径的文件， 如果文件不存在，则创建该文件， 并返回一个文件描述符；
### int fd_invalid(int fd)
检查文件描述符是否有效， 以及对应的文件能否被打开, 若文件描述符无效或对应文件无法打开， 则返回1， 否则返回 0；
### int close(int fd)
关闭文件描述符 fd；
### int check_offset(FILE *f, int offset)
检查并调整偏移量；
### int lseek(int fd, int offset, int whence)
根据 whence 参数改变文件描述符 fd 的偏移量；
### size_t read(int fd, void *buf, size_t count)
从文件描述符 fd 读取数据到缓冲区 buf， 最多读取 count 字节；
### size_t write(int fd, const void *buf, size_t count)
从缓冲区 buf 写入数据到文件描述符 fd， 最多读取 count 字节；
### int find_inode_by_base(int base)
从 offset_by_inode 数组中找到偏移量等于 base 的 inode， 返回该 inode 在数组中的索引；
### int fstat(int fd, struct stat* buf)
打印获取文件状态信息；
### int fcntl(int fd, int cmd, ...) 
根据 cmd 对文件进行控制操作， 包括文件描述符的复制、锁定和状态获取与设置；
### int dup(int oldfd)
复制文件描述符 oldfd；
### int dup3(int oldfd, int newfd, int flags)
复制文件描述符 oldfd 到文件描述符 newfd， 如果 newfd 文件已打开， 则先关闭 newfd， 再复制 oldfd 和状态；
### int find_curnode_byfd(int fd) 
根据文件描述符查找对应的 inode （从 offset_by_inode 数组中找到偏移量等于 fds[fd].base 的 inode）， 返回该 inode 在数组中的索引；
### int getdents64(int fd, struct linux_dirent64 *dirp, size_t count)
扫描目录并读取目录中的所有目录项；
### int find_inode_by_pathname(const char *pathname)
从 inode 数组中查找匹配路径名 pathname 的 inode， 返回该 inode 在数组中的索引；
### int mkdir(int dirfd, const char *pathname, int mode)
在路径 pathname 下创建新目录， 文件描述符为 dirfd；
### int unlinkat(int dirfd, const char *pathname, int flags)
在路径 pathname 下删除文件描述符为 dirfd 的目录或文件；

# lolibc
## kstruct/list
为了增加灵活性， 内核链表采用循环双向链表。 内核链表最大的特点 ：不是链表包含结构体， 而是结构体包含链表；
### init_list_head(struct list_head *list)
初始化循环双向链表， list 为头结点；
### void list_add(struct list_head *head, struct list_head *new)
将 new 结点用头插法插入以 head 为头结点的链表；
### void list_add_tail(struct list_head *head, struct list_head *new)
将 new 结点用尾插法插入以 head 为头结点的链表；
### void list_del(struct list_head *entry)
从 entry 所在链表中删除 entry 结点， 删除单个结点时可以正常使用， 但要遍历链表并执行删除操作时需搭配宏 list_for_each_entry_safe；
### void list_replace(struct list_head *old, struct list_head *new)
从 old 所在链表中用 new 结点替换 old 结点；
### void list_replace_init(struct list_head *old, struct list_head *new)
从 old 所在链表中用 new 结点替换 old 结点， 并初始化 old 结点；
### void list_move(struct list_head *list, struct list_head *head)
将 list 结点从 list 所在链表中移除， 并用头插法插入到以 head 结点为头结点的链表；
### void list_move_tail(struct list_head *list, struct list_head *head)
将 list 结点从 list 所在链表中移除， 并用尾插法插入到以 head 结点为头结点的链表；
### int list_is_empty(struct list_head *head)
判断链表是否为空（ 内核链表一定存在一个不包含数据的头结点，所以当链表只有头结点时链表即为空 ）， 链表为空则返回 1 ， 否则返回 0 ；
### int list_is_last(struct list_head *list,struct list_head *head)
判断 list 结点是否为以 head 结点为头结点的链表的尾结点， list 为尾结点则返回 1 ， 否则返回 0 ；
### void list_splice(struct list_head *list, struct list_head *head)

## bitmap
### bool bitmap_is_in_use(uint8_t *page_map, size_t map_max, int index)
检查位图中的 index 位是否已使用， 返回 index 位；
### void bitmap_set_status(uint8_t *page_map, size_t map_max, int index, bool status)
将位图中的 index 位置为status；
### size_t bitmap_alloc_one(uint8_t *page_map, size_t map_max)
在位图中分配一位（从位图中找到一个空闲的位， 并将其置为1）， 成功则返回该位， 否则返回-1；
### size_t bitmap_alloc(uint8_t *page_map, size_t map_max, size_t cnt)
在位图中分配连续的 cnt 位， 成功返回第一个分配位的索引， 否则返回-1；
### void bitmap_free(uint8_t *page_map, size_t map_max, size_t index, size_t cnt)
释放位图中从 index 位开始的 cnt 位（将第 index 位到第 （index + cnt -1）位置为 0 ）；

## struct/list
### struct list *list_init()
初始化一个空链表；
### struct list_element *new_list_elem(void *data)
初始化一个新节点；
### bool inline list_is_empty(struct list *list)
判断链表是否为空， 为空则返回1， 否则返回0；
### void list_push(struct list *list, void *data)
创建一个新结点， 数据为 data， 并用头插法将该结点插入链表中；
### void list_append(struct list *list, void *data)
创建一个新结点， 数据为 data， 并用尾插法将该结点插入链表中；
### void list_remove(struct list *list, void *data)
从链表中删除所有数据为 data 的结点；
### void list_print(struct list *list)
打印链表；

## bits
### bool get_bit(uint64_t x, int l)
查询 x 中第 l 位的值；
### set_bit(uint64_t x, int l, bool status)
将 x 中第 l 位置为 status；
### get_bits_range(uint64_t x, int s, int len)
返回 x 中从第 s 位开始的 len 个连续位的值；
### void hexdump(uint8_t *buf, int len)
将缓冲区中的数据 buf 以 16 进制的形式输出；

## stdio
### int get_fmt_flag(const char *fmt, int *flag)
解析格式化字符串 fmt 中的前导标志（flag），并设置相应的标志变量；
### int copy_fmt_str(char *dst, const char *buf, int flag, int num)
根据指定的对齐标志 flag 和长度 num 将字符串 buf 复制到 dst;
<!-- ### int vsprintf(char *out, const char *fmt, va_list ap)
### int sprintf(char *out, const char *fmt, ...)
### int printf(const char *fmt, ...) -->

## stdlib
### int rand(void)
产生一个随机数；
### void srand(unsigned int seed)
输入随机数种子 seed ；
### int abs(int x)
计算 x 的绝对值；
### int atoi(const char* nptr)
将数字构成的字符串 nptr 转化位 int 型；
### void *malloc(size_t size) 
分配一个 size 大小的内存；
### void free(void *ptr) 
释放内存；
### int min(int a, int b)
返回 a, b 中的最小值；
### int max(int a, int b)
返回 a, b 中的最大值；

## string
### size_t strlen(const char *s)
计算 s 指向的字符串长度；
### char *strcpy(char *dst, const char *src)
将 src 指向的字符串复制到 dst 指向的字符串中, 返回 dst 的起始地址；
### char *strncpy(char *dst, const char *src, size_t n)
将 src 指向的字符串中最多 n 位复制到 dst 指向的字符串中， 返回 dst 的起始地址；
### char *strcat(char *dst, const char *src)
将 src 指向的字符串连接到 dst 的末尾， 返回 dst 的起始地址；
### int strcmp(const char *s1, const char *s2)
比较字符串 s1 和 s2 的大小， 返回 s1 - s2;
### int strncmp(const char *s1, const char *s2, size_t n)
比较字符串 s1 和 s2 前 n 位的大小；
### void *memset(void *s, int c, size_t n)
将 s 的前 n 个 bit 的值置为 c；
### void *memcpy(void *out, const void *in, size_t n)

# mm
## buddy
### void check_pages(struct page * start_page, uint64_t num)
检查从 start_page 页开始共 num 页的分配状态， 确保伙伴系统的正确性；
### void dump_pages(struct page * start_page, uint64_t num)
打印从 start_page 页开始共 num 页的分配状态 和每阶的空闲列表状态；
### void kfree_pages_buddy(struct page * page)
释放内存页 page ， 并将其合并到空闲列表；
### void kinit_pages_buddy(struct page * start_page, uint64_t num)
初始化从 start_page 开始共 num 页的内存页， 将其分配状态置为 1 ；
### struct page * kalloc_pages_buddy(uint64_t page_num)
找到一个能够满足 page_num 请求的最小阶数的内存块，并分配这些内存页;

## memblock
### void kfree(void *pa)
释放内存页 pa（将内存页 pa 标记为空闲，并将其添加到内存管理子系统的空闲列表中）;
### void freerange(void *pa_start, void *pa_end)
释放从 pa_start 到 pa_end 的所有内存页；
### void* kalloc(void)
从内存管理子系统中分配内存;
### void* kmalloc(int size)
从内存管理子系统中分配大小为 size 的内存；

## mm
### uint64_t get_dmw_data(uint32_t base, uint8_t mat, uint8_t plv)
将输入的基地址 base , 内存属性 mat 和权限级别 plv 组合成一个可以写入特定控制寄存器的值;
### void dmw_init() 
设置内存访问控制,  确保不同域的软件按照既定的权限级别访问内存;
### void dump_dmw() 
打印当前的域访问控制寄存器 dmw 的值
### void tlb_init()
初始化 tlb；
### void clean_dmw()
清除当前 dmw 的值；
### void pmm_init()
初始化物理内存管理系统；
### void mm_init()
输出化内存管理(dmw 和 pmm)；

## vm
### void vm_init(void)
初始化虚拟内存系统。设置内核页表并配置相关的控制寄存器；
### void kvmmap(pagetable_t kpgtbl, uint64_t va, uint64_t pa, uint64_t sz, int perm)
在内核页表中映射虚拟地址（起始地址为 va , 大小为 sz ）到物理地址（ pa ），设置相应的权限 perm ;
### pte_t * walk(pagetable_t pagetable, uint64_t va, int alloc)
遍历页表，查找给定虚拟地址对应的PTE，如果该页表项不存在，会分配新的页表页， 返回指向该 pte 的指针； 
### uint64_t walkaddr(pagetable_t pagetable, uint64_t va)
根据 pagetable 和 va 查找物理地址， 返回该物理地址；
### int mappages(pagetable_t pagetable, uint64_t va, uint64_t size, uint64_t pa, uint64_t perm)
为一系列虚拟地址(起始地址为 va 大小为 sz)创建PTEs， 将它们映射到相应的物理地址；
### int map_single_page(pagetable_t pagetable, uint64_t va, uint64_t pa, uint64_t perm)
为单个虚拟地址创建PTE， 将它映射到相应的物理地址；
### void uvmunmap(pagetable_t pagetable, uint64_t va, uint64_t npages, int do_free)
从页表中移除页面映射， 可选地释放物理内存；
### pagetable_t uvmcreate()
创建一个新的用户虚拟内存页表， 返回该页表地址；
### void uvminit(pagetable_t pagetable, unsigned char *src, unsigned int sz)
初始化一个用户级虚拟内存环境（将初始化代码 src 加载到 新进程的地址空间）；
### uint64_t uvmalloc(pagetable_t pagetable, uint64_t oldsz, uint64_t newsz)
在用户虚拟内存空间中分配新内存， 若 oldsz >= newsz 则无需分配， 否则将 oldsz 大小的内存增加到 newsz ，返回新内存大小 newsz ；
### uint64_t uvmdealloc(pagetable_t pagetable, uint64_t oldsz, uint64_t newsz)
缩减用户进程的地址空间，若 oldsz <>= newsz 则无需分配， 否则将 oldsz 大小的内存缩减到 newsz ，返回新内存大小 newsz ；
### void freewalk(pagetable_t pagetable)
释放页表；
### void uvmfree(pagetable_t pagetable, uint64_t sz)
释放用户进程的页表；
### int uvmcopy(pagetable_t old, pagetable_t new, uint64_t sz)
复制父进程的页表和物理内存到子进程；
### void uvmclear(pagetable_t pagetable, uint64_t va)
清除页表 pagetable 中虚拟地址 va 的权限；
### int copyout(pagetable_t pagetable, uint64_t dstva, char *src, uint64_t len)
从内核空间 src 复制长度为 len 的数据到数据空间 dstva ；
### int copyin(pagetable_t pagetable, char *dst, uint64_t srcva, uint64_t len)
从用户空间 srcva 复制长度为 len 的数据到数据空间 dst ；
### int copyinstr(pagetable_t pagetable, char *dst, uint64_t srcva, uint64_t max)
从用户空间 srcva 复制字符串到内核空间 dst ， 最大为 max 字节；

