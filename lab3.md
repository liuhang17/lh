#lab3实验报告  
##第一题：给未被映射的地址映射上物理页
###1.1具体实现
实验二已经实现了对虚拟内存的支持，但还没有为其分配具体内存页，这样的话，在访问这些虚拟页的时候就会产生pagefault异常。首先分析一下在ucore操作系统中，如果出现了page fault，应当进行怎样的处理流程。

与通常的中断处理一样，最终page fault的处理也会来到trap_dispatch函数，在该函数中会根据中断号，将page fault的处理交给pgfault_handler函数，进一步交给do_pgfault函数进行处理，因此我们需要修改do_pgfault函数。根据提示修改相应大代码如下：
```
ptep = get_pte(mm->pgdir, addr, 1); // 获取当前发生缺页的虚拟页对应的PTE
if (*ptep == 0) { // 如果需要的物理页是没有分配而不是被换出到外存中
    struct Page* page = pgdir_alloc_page(mm->pgdir, addr, perm); // 分配物理页，并且与对应的虚拟页建立映射关系
}
else {
    // 将物理页从外存换到内存中，练习2中实现的内容
}
```
###1.2问题回答
问题一：
PTE、PDE 每一位的详细用处已于 lab2 report 中写出。
PDE 用于寻找 PTE ，PTE 用于判断物理页面是否存在。如果不存在，则需要给 PDT 分配一个页面，并建立物理地址和逻辑地址之间的联系。
问题二：
将发生错误的线性地址保存在cr2寄存器中;
在中断栈中依次压入EFLAGS，CS, EIP，以及页访问异常码error code，由于ISR一定是运行在内核态下的，因此不需要压入ss和esp以及进行栈的切换；
根据中断描述符表查询到对应页访问异常的ISR，跳转到对应的ISR处执行，接下来将由软件进行处理；

##第二题：补充完成基于FIFO的页面替换算法。
###1.1代码实现
实现的流程如下： 
判断当前是否对交换机制进行了正确的初始化；
将虚拟页对应的物理页从外存中换入内存；
给换入的物理页和虚拟页建立映射关系；
将换入的物理页设置为允许被换出；

对第一题省略代码补充修改
```
ptep = get_pte(mm->pgdir, addr, 1); // 获取当前发生缺页的虚拟页对应的PTE
if (*ptep == 0) { // 如果需要的物理页是没有分配而不是被换出到外存中
    struct Page* page = pgdir_alloc_page(mm->pgdir, addr, perm); // 分配物理页，并且与对应的虚拟页建立映射关系
} else {
        if (swap_init_ok) { // 判断是否当前交换机制正确被初始化
            struct Page *page = NULL;
            swap_in(mm, addr, &page); // 将物理页换入到内存中
            page_insert(mm->pgdir, page, addr, perm); // 将物理页与虚拟页建立映射关系
            swap_map_swappable(mm, addr, page, 1); // 设置当前的物理页为可交换的
            page->pra_vaddr = addr; // 同时在物理页中维护其对应到的虚拟页的信息，这个语句本人觉得最好应当放置在page_insert函数中进行维护，在该建立映射关系的函数外对物理page对应的虚拟地址进行维护显得有些不太合适
        } else {
            cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
            goto failed;
        }
}

```
分析发现当调用swap_in函数的时候，会进一步调用alloc_page函数进行分配物理页，一旦没有足够的物理页，则会使用swap_out函数将当前物理空间的某一页换出到外存，该函数会进一步调用sm（swap manager）中封装的swap_out_victim函数来选择需要换出的物理页，具体对应到了_fifo_swap_out_victim函数，在FIFO算法中，按照物理页面换入到内存中的顺序建立了一个链表，因此链表头处便指向了最早进入的物理页面，也就在在本算法中需要被换出的页面，因此只需要将链表头的物理页面取出，然后删掉对应的链表项即可
```
list_entry_t *head=(list_entry_t*) mm->sm_priv; // 找到链表的入口
assert(head != NULL); // 进行一系列检查
assert(in_tick==0);
list_entry_t *le = list_next(head); // 取出链表头，即最早进入的物理页面
assert(le != head); // 确保链表非空
struct Page *page = le2page(le, pra_page_link); // 找到对应的物理页面的Page结构
list_del(le); // 从链表上删除取出的即将被换出的物理页面
*ptr_page = page;
```
FIFO算法中实现的_fifo_swap_map_swappable函数，这也是在本次练习中需要进行实现的另外一个函数，这个函数的功能比较简单，就是将当前的物理页面插入到FIFO算法中维护的可被交换出去的物理页面链表中的末尾，代码如下：
```
list_entry_t *head=(list_entry_t*) mm->sm_priv; // 找到链表入口
list_entry_t *entry=&(page->pra_page_link); // 找到当前物理页用于组织成链表的list_entry_t
assert(entry != NULL && head != NULL); 
list_add_before(head, entry); // 将当前指定的物理页插入到链表的末尾
```



###1.2问题回答：
如果要在ucore上实现"extended clock页替换算法"请给你的设计方案，现有的swap_manager框架是否足以支持在ucore中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。并需要回答如下问题
1、需要被换出的页的特征是什么？
2、在ucore中如何判断具有这样特征的页？
3、何时进行换入和换出操作？
答：根据上文中提及到的PTE的组成部分可知，PTE中包含了dirty位和访问位，因此可以确定某一个虚拟页是否被访问过以及写过，但是，考虑到在替换算法的时候是将物理页面进行换出，而可能存在着多个虚拟页面映射到同一个物理页面这种情况，也就是说某一个物理页面是否dirty和是否被访问过是有这些所有的虚拟页面共同决定的，而在原先的实验框架中，物理页的描述信息Page结构中默认只包括了一个对应的虚拟页的地址，应当采用链表的方式，在Page中扩充一个成员，把物理页对应的所有虚拟页都给保存下来；而物理页的dirty位和访问位均为只需要某一个对应的虚拟页对应位被置成1即可置成1；
1、访问位（PTE_A）为 0，修改位（PTE_D）为 0。
2、通过 *pte & PTE_A 和 *pte & PTE_D 是否为 0 判断。
3、在缺页的时候换入，在 swap_out_victim 时换出。
