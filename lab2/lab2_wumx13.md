�����ڴ����ʵ�鱨��
��31 2013011421 ������

��ϰ0����д����ʵ��
	������ʾʹ�á�diff/merge���������ϲ�lab1��lab2�Ĵ��룬����û���ҵ����ߣ����Ǳ��Ƽ�ʹ��meld���ߣ�Ҳ�ܷܺ���ؽ���ͬĿ¼���ļ���ͬ�Ƚϳ���������һһ�ֶ��ϲ���ɾ�������Ӵ��룬�����˲���Ҫ�Ĵ���
	�ⲿ����Ҫ�ϲ����ļ���kdebug.c��trap.c��

��ϰ1��ʵ��firstfit���������ڴ�����㷨��
	�����ϰ��Ҫ��������default_alloc_pages��default_free_pages��
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

��ϰ2��ʵ��Ѱ�������ַ��Ӧ��ҳ����
�ؼ��ĺ����Լ��꺯����
     *   PDX(la) = ���������ַla��ҳĿ¼����
     *   KADDR(pa) : ���������ַpa��ص��ں������ַ
     *   set_page_ref(page,1) : ���ô�ҳ��������һ��
     *   page2pa(page): �õ�page�������һҳ�������ַ
     *   struct Page * alloc_page() : ����һҳ����
     *   memset(void *s, char c, size_t n) : ����sָ���ַ��ǰ��n���ֽ�Ϊ�ֽڡ�c��.

    pde_t *pdep = &pgdir[PDX(la)];//�õ�ҳĿ¼��
    if (!(*pdep & PTE_P)) {//����ҳĿ¼�����
        struct Page *page;
        if (!create || (page = alloc_page()) == NULL) {//���粻��Ҫ����ҳ���߷���ҳʧ��
            return NULL;
        }
        set_page_ref(page, 1);//���ø�ҳ��������һ��
        uintptr_t pa = page2pa(page);//�õ���ҳ�����ַ
        memset(KADDR(pa), 0, PGSIZE);//�����ַת�����ַ��Ȼ��Ѹ�ҳ��ʼ��
        *pdep = pa | PTE_U | PTE_W | PTE_P;//���ÿɶ�����д������λ
    }
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];
    //KADDR(PDE_ADDR(*pdep))���ⲿ������ҳĿ¼���ַ�õ�������ҳ�������ַ����ת�������ַ
    //PTX(la)�����������ַla��ҳ��������
    //��󷵻ص��������ַla��Ӧ��ҳ������ڵĵ�ַ
��ϰ3���ͷ�ĳ���ַ���ڵ�ҳ��ȡ����Ӧ����ҳ�����ӳ��
*   struct Page *page pte2page(*ptep): �õ�ҳ�����Ӧ����һҳ
     *   free_page : �ͷ�һҳ
     *   page_ref_dec(page) : ���ٸ�ҳ�����ô���������ʣ�����ô�ʱ
     *   tlb_invalidate(pde_t *pgdir, uintptr_t la) : ���޸ĵ�ҳ���ǽ�������ʹ�õ���Щҳ��ʹTLB����һҳ��Ч

if (*ptep & PTE_P) {//����ҳ�������
	struct Page *page = pte2page(*ptep);//�ҵ�ҳ�������һҳ����Ϣ
	if (page_ref_dec(page) == 0) {//������һҳû�б�������
		free_page(page);//�ͷŸ�ҳ
	}
	*ptep = 0;//��ҳĿ¼������
	tlb_invalidate(pgdir, la);//���޸ĵ�ҳ���ǽ�������ʹ�õ���Щҳ��ʹTLB����һҳ��Ч
}
