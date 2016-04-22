Lab4实验报告
计31 吴孟轩 2013011421
练习1、分配并初始化一个进程控制块
proc->state = PROC_UNINIT;
proc->pid = -1;
proc->runs = 0;
proc->kstack = 0;
proc->need_resched = 0;
proc->parent = NULL;
proc->mm = NULL;
memset(&(proc->context), 0, sizeof(struct context));//初始化进程上下文
proc->tf = NULL;//初始化中断帧，用于记录进程发生中断前的状态
proc->cr3 = boot_cr3;//因为是内核线程，所以CR3=boot_cr3
proc->flags = 0;
memset(proc->name, 0, PROC_NAME_LEN);

初始化的内容涵盖了除了lisk_link、hash_link以外的所有内容。
可以进一步看看idle进程控制块是怎么设置的：
idleproc->pid = 0;
    idleproc->state = PROC_RUNNABLE;//设置idle状态为运行状态
    idleproc->kstack = (uintptr_t)bootstack;//idle内核栈的起始地址
    idleproc->need_resched = 1;//设置需要调度
    set_proc_name(idleproc, "idle");//设置现成名字
    nr_process ++;
current = idleproc;

练习2、为新创建的内核线程分配资源
	在创建init线程(输出helloworld)的时候，只是用了一下一句话：
kernel_thread(init_main, "Hello world!!", 0);//生成一个helloworld线程
（1）在讲解do_fork函数之前，有必要讲解kernel_thread()
int
kernel_thread(int (*fn)(void *), void *arg, uint32_t clone_flags) {
    struct trapframe tf;
    memset(&tf, 0, sizeof(struct trapframe));
    tf.tf_cs = KERNEL_CS;
    tf.tf_ds = tf.tf_es = tf.tf_ss = KERNEL_DS;
    tf.tf_regs.reg_ebx = (uint32_t)fn;//设置下次要启动的函数，调度之前ebx中存有函数地址
    tf.tf_regs.reg_edx = (uint32_t)arg;//参数，调度之前参数的地址存于edx
    tf.tf_eip = (uint32_t)kernel_thread_entry;//下次进程运行的位置
    return do_fork(clone_flags | CLONE_VM, 0, &tf);
}
	它的功能主要是在确定这个新创建的内核线程启动时的位置和环境，有特点的是，启动新创建的线程使用的是中断机制，也就是说现在创建的线程并不会马上开始执行，在后面这个线程将被放到进程列表，进行调度，那个时候才是执行的时候。

（2）这个函数调用了do_fork（）函数完成具体内核线程的创建工作，这个过程中，需要给新内核线程分配资源，并且复制原进程的状态。

具体工作：
	if ((proc = alloc_proc()) == NULL) {//获得用户信息块
        goto fork_out;
    }
    proc->parent = current;//设置父进程

    if (setup_kstack(proc) != 0) {//分配了2页内核栈
        goto bad_fork_cleanup_proc;
    }
    if (copy_mm(clone_flags, proc) != 0) {
        goto bad_fork_cleanup_kstack;
    }
    copy_thread(proc, stack, tf);//设置中断帧
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        proc->pid = get_pid();
        hash_proc(proc);//将新进程加入hash_list
        list_add(&proc_list, &(proc->list_link));//将新进程加入proc_list
        nr_process ++;
    }
    local_intr_restore(intr_flag);
    wakeup_proc(proc);//唤醒进程，等待调度
	ret = proc->pid;

练习3、理解proc_run和它调用的函数如何完成进程切换的
// proc_run - make process "proc" running on cpu
// NOTE: before call switch_to, should load  base addr of "proc"'s new PDT
	void
	proc_run(struct proc_struct *proc) {
    if (proc != current) {
        bool intr_flag;
        struct proc_struct *prev = current, *next = proc;
        local_intr_save(intr_flag);//关中断
        {
            current = proc;
            load_esp0(next->kstack + KSTACKSIZE);
            lcr3(next->cr3);//页目录其实没有变？体现了这样一个过程
            switch_to(&(prev->context), &(next->context));
        }
        local_intr_restore(intr_flag);//开中断
    }
}
 (3)深入switch_to函数
在看switch_to函数之前，先看一下参数context的结构
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
再查看文件switch.S
.text
.globl switch_to
switch_to:                      # switch_to(from, to)
    # save from's registers
    movl 4(%esp), %eax          # eax points to from
    popl 0(%eax)                # 保存返回地址到context->eip
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