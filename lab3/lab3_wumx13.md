虚拟内存管理实验报告
计31 2013011421 吴孟轩

练习1：给未被映射的地址映射上物理页
	ptep=get_pet(mm->dir,addr,1);
	if(ptep == NULL){ //页表项不存在
    	cprintf("get_pte in do_pgfault failed\n");
        goto failed;
}
if (*ptep == 0) { 
	//物理页不在内存之中
	//判断是否可以分配新页
        if (pgdir_alloc_page(mm->pgdir, addr, perm) == NULL) {
            cprintf("pgdir_alloc_page in do_pgfault failed\n");
        goto failed;
       }
 }
else{
	if(swap_init_ok) {
            struct Page *page=NULL;
			ret = swap_in(mm, addr, &page);
			if(ret != 0){ //判断页面可否换入
				cprintf("swap_in in do_pgfault failed\n");
                goto failed;
            }    
			//建立映射
			page_insert(mm->pgdir, page, addr, perm);
            swap_map_swappable(mm, addr, page, 1);
        }
        else {
            cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
            goto failed;
        }
   }
   ret = 0;
failed:
    return ret;
}
练习2：补充完成基于FIFO算法
	_fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in){
		list_entry_t *head=(list_entry_t*) mm->sm_priv;
		list_entry_t *entry=&(page->pra_page_link);

		assert(entry != NULL && head!=NULL);
		list_add(head,entry);
		return 0;
	}
	pra_page_link用来构造按页的第一次访问时间进行排序的一个链表，这个链表的开始表示第一次访问时间最近的页，链表的尾部表示第一次访问时间最远的页。
	sm_priv指向用来连接记录页访问情况的链表头。
	_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
		list_entry_t *head=(list_entry_t*) mm->sm_priv;
		assert(head != NULL);
		assert(in_tick==0);
		//获取最远端的页
		list_entry_t *le = head->prev;
		assert(head != le);
		//取下该页
		struct Page *p = le2page(le,pra_page_link);
		//释放该页
		list_del(le);
		assert(p!=NULL);
		//将该页存入*ptr_page中
		*ptr_page = p;
		return 0;
}	