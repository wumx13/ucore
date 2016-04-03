�����ڴ����ʵ�鱨��
��31 2013011421 ������

��ϰ1����δ��ӳ��ĵ�ַӳ��������ҳ
	ptep=get_pet(mm->dir,addr,1);
	if(ptep == NULL){ //ҳ�������
    	cprintf("get_pte in do_pgfault failed\n");
        goto failed;
}
if (*ptep == 0) { 
	//����ҳ�����ڴ�֮��
	//�ж��Ƿ���Է�����ҳ
        if (pgdir_alloc_page(mm->pgdir, addr, perm) == NULL) {
            cprintf("pgdir_alloc_page in do_pgfault failed\n");
        goto failed;
       }
 }
else{
	if(swap_init_ok) {
            struct Page *page=NULL;
			ret = swap_in(mm, addr, &page);
			if(ret != 0){ //�ж�ҳ��ɷ���
				cprintf("swap_in in do_pgfault failed\n");
                goto failed;
            }    
			//����ӳ��
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
��ϰ2��������ɻ���FIFO�㷨
	_fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in){
		list_entry_t *head=(list_entry_t*) mm->sm_priv;
		list_entry_t *entry=&(page->pra_page_link);

		assert(entry != NULL && head!=NULL);
		list_add(head,entry);
		return 0;
	}
	pra_page_link�������찴ҳ�ĵ�һ�η���ʱ����������һ�������������Ŀ�ʼ��ʾ��һ�η���ʱ�������ҳ�������β����ʾ��һ�η���ʱ����Զ��ҳ��
	sm_privָ���������Ӽ�¼ҳ�������������ͷ��
	_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
		list_entry_t *head=(list_entry_t*) mm->sm_priv;
		assert(head != NULL);
		assert(in_tick==0);
		//��ȡ��Զ�˵�ҳ
		list_entry_t *le = head->prev;
		assert(head != le);
		//ȡ�¸�ҳ
		struct Page *p = le2page(le,pra_page_link);
		//�ͷŸ�ҳ
		list_del(le);
		assert(p!=NULL);
		//����ҳ����*ptr_page��
		*ptr_page = p;
		return 0;
}	