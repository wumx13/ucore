物理内存管理实验报告
计31 2013011421 吴孟轩

练习0：填写已有实验
	书上提示使用“diff/merge”工具来合并lab1和lab2的代码，可是没有找到这款工具，但是被推荐使用meld工具，也能很方便地将不同目录的文件异同比较出来，可以一一手动合并，删除，增加代码，避免了不必要的错误。
	这部分主要合并的文件有kdebug.c、trap.c。

练习1：实现firstfit连续物理内存分配算法。
	这个练习主要就是完善default_alloc_pages和default_free_pages。
	default_alloc_pages(size_t n) {
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }
    list_entry_t *le, *len;
    le = &free_list;

    while((le=list_next(le)) != &free_list) {
      struct Page *p = le2page(le, page_link);
      if(p->property >= n){
        int i;
        for(i=0;i<n;i++){
          len = list_next(le);
          struct Page *pp = le2page(le, page_link);
          SetPageReserved(pp);
          ClearPageProperty(pp);
          list_del(le);
          le = len;
        }
        if(p->property>n){
          (le2page(le,page_link))->property = p->property - n;
        }
        ClearPageProperty(p);
        SetPageReserved(p);
        nr_free -= n;
        return p;
      }
    }
    return NULL;
}

	default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    assert(PageReserved(base));

    list_entry_t *le = &free_list;
    struct Page * p;
    while((le=list_next(le)) != &free_list) {
      p = le2page(le, page_link);
      if(p>base){
        break;
      }
    }
    //list_add_before(le, base->page_link);
    for(p=base;p<base+n;p++){
      list_add_before(le, &(p->page_link));
    }
    base->flags = 0;
    set_page_ref(base, 0);
    ClearPageProperty(base);
    SetPageProperty(base);
    base->property = n;
    
    p = le2page(le,page_link) ;
    if( base+n == p ){
      base->property += p->property;
      p->property = 0;
    }
    le = list_prev(&(base->page_link));
    p = le2page(le, page_link);
    if(le!=&free_list && p==base-1){
      while(le!=&free_list){
        if(p->property){
          p->property += base->property;
          base->property = 0;
          break;
        }
        le = list_prev(le);
        p = le2page(le,page_link);
      }
    }

    nr_free += n;
    return ;
}

练习2、实现寻找虚拟地址对应的页表项
关键的函数以及宏函数：
     *   PDX(la) = 返回虚拟地址la的页目录索引
     *   KADDR(pa) : 返回物理地址pa相关的内核虚拟地址
     *   set_page_ref(page,1) : 设置此页被引用了一次
     *   page2pa(page): 得到page管理的那一页的物理地址
     *   struct Page * alloc_page() : 分配一页出来
     *   memset(void *s, char c, size_t n) : 设置s指向地址的前面n个字节为字节‘c’.

    pde_t *pdep = &pgdir[PDX(la)];//得到页目录项
    if (!(*pdep & PTE_P)) {//假如页目录项不存在
        struct Page *page;
        if (!create || (page = alloc_page()) == NULL) {//假如不需要分配页或者分配页失败
            return NULL;
        }
        set_page_ref(page, 1);//设置该页被引用了一次
        uintptr_t pa = page2pa(page);//得到该页物理地址
        memset(KADDR(pa), 0, PGSIZE);//物理地址转虚拟地址，然后把该页初始化
        *pdep = pa | PTE_U | PTE_W | PTE_P;//设置可读，可写，存在位
    }
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];
    //KADDR(PDE_ADDR(*pdep))：这部分是由页目录项地址得到关联的页表物理地址，再转成虚拟地址
    //PTX(la)：返回虚拟地址la的页表项索引
    //最后返回的是虚拟地址la对应的页表项入口的地址
练习3、释放某虚地址所在的页并取消对应二级页表项的映射
*   struct Page *page pte2page(*ptep): 得到页表项对应的那一页
     *   free_page : 释放一页
     *   page_ref_dec(page) : 减少该页的引用次数，返回剩下引用此时
     *   tlb_invalidate(pde_t *pgdir, uintptr_t la) : 当修改的页表是进程正在使用的那些页表，使TLB的那一页无效

if (*ptep & PTE_P) {//假如页表项存在
	struct Page *page = pte2page(*ptep);//找到页表项的那一页的信息
	if (page_ref_dec(page) == 0) {//假如这一页没有被引用了
		free_page(page);//释放该页
	}
	*ptep = 0;//该页目录项清零
	tlb_invalidate(pgdir, la);//当修改的页表是进程正在使用的那些页表，使TLB的那一页无效
}
