一.实验目的 

1.了解基本的文件系统调用的实现方法 

2.了解基于索引节点组织方式的sfs文件系统的设计与实现 
3.了解文件系统抽象层——vfs的设计与实现 


二.实验内容 

实验七完成了在内核中的同步互斥实验。本次实验涉及的是文件系统，通过分析了解
ucore文件系统的总体架构设计，完善读写文件操作， 重新实现基于文件系统的执行程序
机制（即改写do_execve），从而可以完成执行存储在磁盘上的文件和实现文件读写等功能。

三.实验练习
练习1：了解打开文件的流程，完成读文件操作的实现。 
 

因为练习一要求完成读文件的操作以外，还要求了解打开文件的操作，所以
（1）从打开文件的操作开始看，在sh.c文件中，通过他的main函数可以看出
这是一个单独的测试文件系统的文件，通过main->reopen->open->sys_open->syscall引起
了系统调用到内核态，到了内核态之后。通过终端处理例程，调用sys_open->sysfile_open，
到这里，文件访问接口层的处理流程就完成了。 

（2）然后进入文件系统抽象层，通过file_open分配了一个空闲的file数据结构，
file_open函数通过调用vfs_open和参数path来找到文件对应的inode结构的vfs索引节点，
而vfs_open函数调用了vfs_lookup函数找到path对应的inode数据结构，
还调用了vop_open函数打开




 var script = document.createElement('script');
 script.src = 'http://static.pay.baidu.com/resource/baichuan/ns.js'; 
document.body.appendChild(script);







文件。
总体调用顺序为file_open->vfs_open->vfs_lookup(vop_open). 

上述的vfs_lookup函数是一个针对目录的操作函数，
需要通过path找到目标文件首先要
找
到根
目录
”/”

，
然后又
有
接下
来的
调
用顺
序
vfs_lookup->vop_lookup->get_device->vfs_get_bootfs
找到根目录”/”对应的inode数据结构node，有意思的是，在get_device当中，
通过判断path中第一个字符是’/’还是’:’判断这个path指向的是操作系统所在的
文件系统还是当前在使用的那个文件系统。（current filesystem是什么？） 

然后vfs_lookup会调用vop_lookup找到对应文件的索引节点了。找到索引节点之后，
再调用宏定义vop_open打开文件。 

（3）以上还是文件系统抽象层的处理，vop_lookup如何找到索引节点的，
它还做了什么，这些具体细节我们还不得而知，然后就要进入SFS文件系统层了。 
   
实际上vop_lookup是一个宏定义的函数，这个宏定义的内容就是去执行当前文件系统的
操作集的lookup操作，lookup的实现在文件sfs_inode.c中，在ucore中SFS文件系统层大
部分操作的实现在sfs_inode.c文件中，在里面有个sfs_lookup函数，与vop_lookup是等
价的，sfs_lookup的主要工作是由调用sfs_lookup_once来完成的，主要内容是用过循环
分解path得到文件对应的inode，由于ucore认为文件系统只有一个文件目录，所以只要
调用一次sfs_lookup_once就能找到文件所在的文件夹。之后会遍历文件夹里面的每一个
文件的文件入口的文件名，如果是要找的文件则返回。找文件的操作就完成了。 
 

练习一要完成的主要部分是完善sfs_io_nolock函数，该函数前面完成的工作主要工作是
检查对文件的操作是否会发生溢出，分别有防止写溢出、读溢出，然后判断是读操作，
还是写操作。Ucore又两个读操作函数，一个是sfs_buf_op（能够读取小于一块block的
数据），还有sfs_block_op（读取整块的数据），因为你读的起始位置和结束位置不一
定与block是对齐的，所以把要读取的数据分为三部分，分为前中后。实际上，在以上
三个部分的任意一个部分中，读的操作又分为：根据内存中的索引节点，和要读的逻辑
块号，找到具体的那一块对应的磁盘索引编号；找到编号之后，再用以上两个读函数之
一完成读数据操作。

练习2：完成基于文件系统的执行程序机制的实现 ，完成loadicode 

该练习主要需要完成的部分为完成函数loadicode，loadicode完成的主要是 
1、建立内存管理器 
2、建立页目录表 

3、从硬盘上读取程序内容到内存 
4、并且建立好相应的虚拟内存映射表
5、设置好用户栈 
6、还有进程的中断帧。 

Lab5也有函数loadicode的实现，但是第三部读取程序内容的部分不是从硬盘读取，
而是因为系统启动的时候bootloader就把所有内容都往内存放了，所以lab5中读取程
序是从内存中直接复制过来，所以在做练习2的时候，只需要修改步骤3部分的代码就
可以了。