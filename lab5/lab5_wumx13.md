# 实验五：用户进程管理

## 实验目的
- 了解第一个用户进程创建过程
- 了解系统调用框架的实现机制
- 了解ucore如何实现系统调用sys_fork/sys_exec/sys_exit/sys_wait来进行进程管理

## 实验内容
实验5将创建用户进程，让用户进程在用户态执行，且在需要ucore支持时，可通过系统调用来让ucore提供服务。为此需要构造出第一个用户进程，并通过系统调用sys-fork/sys-exec/sys-exit/sys-wait来支持运行不同的应用程序，完成对用户进程的执行过程的基本管理。

### 练习1: 加载应用程序并执行
do-execv函数调用load-icode来加载并解析一个处于内存中的ELF执行文件格式的应用程序，建立相应的用户内存空间来放置应用程序的代码段、数据段等，且要设置好proc-struct结构中的成员变量trapframe中的内容，确保在执行此进程后，能够从应用程序设定的起始执行地址开始执行。需设置正确的trapframe内容。  

修改load_icode函数：

    tf->tf_cs = USER_CS;
    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
    tf->tf_esp = USTACKTOP;
    tf->tf_eip = elf->e_entry;
    tf->tf_eflags = FL_IF;
	
do_execv函数中的内容：
	int
	do_execve(const char *name, size_t len, unsigned char *binary, size_t size) {
	    struct mm_struct *mm = current->mm;
	    if (!user_mem_check(mm, (uintptr_t)name, len, 0)) {
	        return -E_INVAL;
	    }
	    if (len > PROC_NAME_LEN) {
	        len = PROC_NAME_LEN;
	    }
	
	    char local_name[PROC_NAME_LEN + 1];
	    memset(local_name, 0, sizeof(local_name));
	    memcpy(local_name, name, len);
	
	    if (mm != NULL) {
	        lcr3(boot_cr3);
	        if (mm_count_dec(mm) == 0) {
	            exit_mmap(mm);
	            put_pgdir(mm);
	            mm_destroy(mm);
	        }
	        current->mm = NULL;
	    }
	    int ret;
	    if ((ret = load_icode(binary, size)) != 0) {
	        goto execve_exit;
	    }
	    set_proc_name(current, local_name);
	    return 0;
	
	execve_exit:
	    do_exit(ret);
	    panic("already exit: %e.\n", ret);
	}


调用exit_mmap（mm）和put_pgdir（mm）回收当前进程的内存空间
调用load_icode设置新的存储空间通过二进制程序



### 练习2: 父进程复制自己的内存空间给子进程
创建子进程的函数do-fork在执行中将拷贝当前进程（即父进程）的用户内存地址空间中的合法内容到新进程中（子进程），完成内存资源的复制。具体是通过copy-range函数实现的，补充copy-range的实现，确保能够正确执行。


copy_range函数：

	void * kva_src = page2kva(page);
    void * kva_dst = page2kva(npage);

    memcpy(kva_dst, kva_src, PGSIZE);

    ret = page_insert(to, npage, start, perm);
    assert(ret == 0);

1.调用alloc_proc,首先获得一块用户信息块。

2.为进程分配一个内核栈。

3.复制原进程的内存管理信息到新进程

4.复制原进程上下文到新进程

5.将新进程添加到进程列表

6.唤醒新进程

7.返回新进程号	
	

### 练习3: 阅读分析源代码，理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现

fork/exec/wait/exit的实现：

fork/wait/exit分别调用了libs/unistd.h中的1,2,3号系统调用，SYS_exit,SYS_fork,SYS_wait;

实验中并没有提供exec函数供用户态进程使用。与此相关的是kernel_execve函数，这个函数在创建用户进程的过程中被使用，通过调用4号系统调用SYS_exec实现。

系统调用的实现：

由idt_init函数初始化中断描述符表，建立系统调用的用户库准备，简化应用程序访问系统调用的复杂性，系统调用的执行过程：

首先保存执行系统调用前的现场，然后根据中断描述符表，进入内核态，跳转到对应位置，执行系统调用。完成调用后，根据保存的信息恢复现场