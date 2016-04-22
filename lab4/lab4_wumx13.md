Lab4ʵ�鱨��
��31 ������ 2013011421
��ϰ1�����䲢��ʼ��һ�����̿��ƿ�
proc->state = PROC_UNINIT;
proc->pid = -1;
proc->runs = 0;
proc->kstack = 0;
proc->need_resched = 0;
proc->parent = NULL;
proc->mm = NULL;
memset(&(proc->context), 0, sizeof(struct context));//��ʼ������������
proc->tf = NULL;//��ʼ���ж�֡�����ڼ�¼���̷����ж�ǰ��״̬
proc->cr3 = boot_cr3;//��Ϊ���ں��̣߳�����CR3=boot_cr3
proc->flags = 0;
memset(proc->name, 0, PROC_NAME_LEN);

��ʼ�������ݺ����˳���lisk_link��hash_link������������ݡ�
���Խ�һ������idle���̿��ƿ�����ô���õģ�
idleproc->pid = 0;
    idleproc->state = PROC_RUNNABLE;//����idle״̬Ϊ����״̬
    idleproc->kstack = (uintptr_t)bootstack;//idle�ں�ջ����ʼ��ַ
    idleproc->need_resched = 1;//������Ҫ����
    set_proc_name(idleproc, "idle");//�����ֳ�����
    nr_process ++;
current = idleproc;

��ϰ2��Ϊ�´������ں��̷߳�����Դ
	�ڴ���init�߳�(���helloworld)��ʱ��ֻ������һ��һ�仰��
kernel_thread(init_main, "Hello world!!", 0);//����һ��helloworld�߳�
��1���ڽ���do_fork����֮ǰ���б�Ҫ����kernel_thread()
int
kernel_thread(int (*fn)(void *), void *arg, uint32_t clone_flags) {
    struct trapframe tf;
    memset(&tf, 0, sizeof(struct trapframe));
    tf.tf_cs = KERNEL_CS;
    tf.tf_ds = tf.tf_es = tf.tf_ss = KERNEL_DS;
    tf.tf_regs.reg_ebx = (uint32_t)fn;//�����´�Ҫ�����ĺ���������֮ǰebx�д��к�����ַ
    tf.tf_regs.reg_edx = (uint32_t)arg;//����������֮ǰ�����ĵ�ַ����edx
    tf.tf_eip = (uint32_t)kernel_thread_entry;//�´ν������е�λ��
    return do_fork(clone_flags | CLONE_VM, 0, &tf);
}
	���Ĺ�����Ҫ����ȷ������´������ں��߳�����ʱ��λ�úͻ��������ص���ǣ������´������߳�ʹ�õ����жϻ��ƣ�Ҳ����˵���ڴ������̲߳��������Ͽ�ʼִ�У��ں�������߳̽����ŵ������б����е��ȣ��Ǹ�ʱ�����ִ�е�ʱ��

��2���������������do_fork����������ɾ����ں��̵߳Ĵ�����������������У���Ҫ�����ں��̷߳�����Դ�����Ҹ���ԭ���̵�״̬��

���幤����
	if ((proc = alloc_proc()) == NULL) {//����û���Ϣ��
        goto fork_out;
    }
    proc->parent = current;//���ø�����

    if (setup_kstack(proc) != 0) {//������2ҳ�ں�ջ
        goto bad_fork_cleanup_proc;
    }
    if (copy_mm(clone_flags, proc) != 0) {
        goto bad_fork_cleanup_kstack;
    }
    copy_thread(proc, stack, tf);//�����ж�֡
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        proc->pid = get_pid();
        hash_proc(proc);//���½��̼���hash_list
        list_add(&proc_list, &(proc->list_link));//���½��̼���proc_list
        nr_process ++;
    }
    local_intr_restore(intr_flag);
    wakeup_proc(proc);//���ѽ��̣��ȴ�����
	ret = proc->pid;

��ϰ3�����proc_run�������õĺ��������ɽ����л���
// proc_run - make process "proc" running on cpu
// NOTE: before call switch_to, should load  base addr of "proc"'s new PDT
	void
	proc_run(struct proc_struct *proc) {
    if (proc != current) {
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        local_intr_save(intr_flag);//���ж�
        {
            current = proc;
            load_esp0(next->kstack + KSTACKSIZE);
            lcr3(next->cr3);//ҳĿ¼��ʵû�б䣿����������һ������
            switch_to(&(prev->context), &(next->context));
        }
        local_intr_restore(intr_flag);//���ж�
    }
}
 (3)����switch_to����
�ڿ�switch_to����֮ǰ���ȿ�һ�²���context�Ľṹ
struct context {
    uint32_t eip;
    uint32_t esp;
    uint32_t ebx;
    uint32_t ecx;
    uint32_t edx;
    uint32_t esi;
    uint32_t edi;
    uint32_t ebp;
};
�ٲ鿴�ļ�switch.S
.text
.globl switch_to
switch_to:                      # switch_to(from, to)
    # save from's registers
    movl 4(%esp), %eax          # eax points to from
    popl 0(%eax)                # ���淵�ص�ַ��context->eip
    movl %esp, 4(%eax)
    movl %ebx, 8(%eax)
    movl %ecx, 12(%eax)
    movl %edx, 16(%eax)
    movl %esi, 20(%eax)
    movl %edi, 24(%eax)
    movl %ebp, 28(%eax)
    # restore to's registers
    movl 4(%esp), %eax          # not 8(%esp): popped return address already
                                # eax now points to to
    movl 28(%eax), %ebp
    movl 24(%eax), %edi
    movl 20(%eax), %esi
    movl 16(%eax), %edx
    movl 12(%eax), %ecx
    movl 8(%eax), %ebx
    movl 4(%eax), %esp

    pushl 0(%eax)               # push eip

    ret