
## 综述

这位实现的[操作系统套件](https://github.com/RobertElderSoftware/recc)个人感觉是由三个部分组成：编译器，模拟器和内核。先按照这几个部分说一下代码的生成过程吧。

### 编译过程

首先必须明确目前这是个和硬件一点关系都没有的工程……虽然应该可以用硬件实现就是了

编译器没有细看，因为好像有好几层代码生成，比较复杂（而且函数起名过于可怕），但是看到比较奇妙的一点是没有做一个真正的链接器，链接操作是写在c里的，也就是要编译某个目标的时候就要给这个编译目标写个c函数来做……实际上就是用个c程序把需要的脚本加载上来（没有细看，看上去是这样的）

举例：内核的编译，build\_kernel.c:

```
void build_tests(void){
	struct build_state * state = create_build_state(); // 初始化编译器
	register_libc_objects(state); // 载入libc等等必要的库
	register_builtin_objects(state);
	register_compiler_objects(state);
	register_kernel_objects(state); // 载入内核文件
	new_register_data_structures_objects(state);
	construct_generated_c_entities(state);

	construct_entity(state, "test/kernel.l0.js"); // 生成目标文件，这里默认是在chrome插件上运行的所以目标是js文件

	destroy_build_state(state);
}

int main(void){
	build_tests();
	return 0;
}

// 这里是加载内核文件的代码：
void register_kernel_objects(struct build_state * state){
    // .............. (摘抄一部分
    register_entity(state, "kernel/filesystem.c", ENTITY_TYPE_C_FILE);
	add_entity_attribute(state, "kernel/filesystem.c", "terminal", "true");
    register_entity(state, "kernel/putchar.c", ENTITY_TYPE_C_FILE);
	add_entity_attribute(state, "kernel/putchar.c", "terminal", "true");
    register_entity(state, "kernel/main.c", ENTITY_TYPE_C_FILE);
	add_entity_attribute(state, "kernel/main.c", "terminal", "true");
    register_entity(state, "kernel/queue.c", ENTITY_TYPE_C_FILE);
	add_entity_attribute(state, "kernel/queue.c", "terminal", "true");
    register_entity(state, "kernel/kernel_state.c", ENTITY_TYPE_C_FILE);
	add_entity_attribute(state, "kernel/kernel_state.c", "terminal", "true");
    register_entity(state, "kernel/kernel_impl.c", ENTITY_TYPE_C_FILE);
	add_entity_attribute(state, "kernel/kernel_impl.c", "terminal", "true");
    // ...............
}
```

编译器的工作原理差不多就是这样。它把原始c代码翻译成一个.l0.c/.l0.js/.l0.py/.l0.java文件而不是二进制文件。然后再把这个文件和模拟器一起编译，这样就可以模拟了。

### 汇编指令

这个[manual](http://recc.robertelder.org/op-cpu-programmer-reference-manual.txt) 里写的很清楚了，piazza上的[帖子](https://piazza.com/class/i5j09fnsl7k5x0?cid=1163)也分析了，不详细写出来。

## 内核分析

内核主要功能基本都在kernel\_impl.c这个文件里，整个文件只有500-行，功能非常简单。

### 中断功能

这个CPU中断的实现非常简单粗暴，没有中断向量表，发生中断时直接从这个内存位置获取中断处理地址：IRQ\_HANDLER，是写死的：

```
#define IRQ_HANDLER      0x00300020u
```

这里存储的就是中断处理函数的起始位置，感觉上也可以说是长度只有1的中断向量表吧…………

在这个位置的处理程序是汇编写的，惯例的保存寄存器、切到内核栈等等，然后按照中断寄存器里的位来判断是何种中断。具体的处理过程在进程调度中说明。

### 进程调度

这个os的进程调度……emmm写的非常粗暴……粘上代码就知道了…………

```
for(i = 0; i < MAX_NUM_PROCESSES; i++)
    pcbs[i].pid = i;

pcbs[0].priority = 5;
pcbs[1].priority = 5;
pcbs[2].priority = 5;
pcbs[3].priority = 2;
pcbs[4].priority = 3;
pcbs[5].priority = 0;
pcbs[6].priority = 1;
pcbs[7].priority = 0;
pcbs[8].priority = 1;
pcbs[9].priority = 3;

pcbs[0].stack_pointer = g_current_sp; /*  Save SP from entering this method so we can exit kernel gracefully */
pcbs[1].stack_pointer = &user_proc_1_stack[STACK_SIZE -1];
pcbs[2].stack_pointer = &user_proc_2_stack[STACK_SIZE -1];
pcbs[3].stack_pointer = &user_proc_3_stack[STACK_SIZE -1];
pcbs[4].stack_pointer = &user_proc_4_stack[STACK_SIZE -1];
pcbs[5].stack_pointer = &user_proc_5_stack[STACK_SIZE -1];
pcbs[6].stack_pointer = &user_proc_6_stack[STACK_SIZE -1];
pcbs[7].stack_pointer = &user_proc_7_stack[STACK_SIZE -1];
pcbs[8].stack_pointer = &user_proc_8_stack[STACK_SIZE -1];
pcbs[9].stack_pointer = &user_proc_9_stack[STACK_SIZE -1];

init_task_stack(&(pcbs[1].stack_pointer), user_proc_1);
init_task_stack(&(pcbs[2].stack_pointer), do_compile);
init_task_stack(&(pcbs[3].stack_pointer), clock_tick_notifier);
init_task_stack(&(pcbs[4].stack_pointer), clock_server);
init_task_stack(&(pcbs[5].stack_pointer), uart1_out_ready_notifier);
init_task_stack(&(pcbs[6].stack_pointer), uart1_out_server);
init_task_stack(&(pcbs[7].stack_pointer), uart1_in_ready_notifier);
init_task_stack(&(pcbs[8].stack_pointer), uart1_in_server);
init_task_stack(&(pcbs[9].stack_pointer), command_server);
```

所以……这个os的所有进程都是在这里写死的，特权级连带初始栈指针也是写死的………………

再看进程优先级的实现：

```
for(i = 1; i < MAX_NUM_PROCESSES; i++)
    add_task_to_ready_queue(&pcbs[i]); // 把各进程加入就绪队列


void add_task_to_ready_queue(struct process_control_block * pcb){ // 重点来了
	pcb->state = READY;
	switch(pcb->priority){
		case 0:{
			task_queue_push_end(&ready_queue_p0, pcb); 
			break;
		}case 1:{
			task_queue_push_end(&ready_queue_p1, pcb); 
			break;
		}case 2:{
			task_queue_push_end(&ready_queue_p2, pcb); 
			break;
		}default:{
			task_queue_push_end(&ready_queue, pcb); 
		}
	}
}

void schedule_next_task(void){ // 当要调度下一个进程的时候：
	struct process_control_block * next_task;
	/*  Get the next task */
	if(task_queue_current_count(&ready_queue_p0)){
		next_task = task_queue_pop_start(&ready_queue_p0); 
	}else if(task_queue_current_count(&ready_queue_p1)){
		next_task = task_queue_pop_start(&ready_queue_p1); 
	}else if(task_queue_current_count(&ready_queue_p2)){
		next_task = task_queue_pop_start(&ready_queue_p2); 
	}else{
		next_task = task_queue_pop_start(&ready_queue); 
	}
	next_task->state = ACTIVE;
	/*  Set its stack pointer */
	g_current_sp = next_task->stack_pointer;
	current_task_id = next_task->pid;
}

```

所以，其实只有4层优先级，实现优先级的方法是如果没有高优先级的进程就绪就调度低优先级的进程……同优先级的用一个队列保存，然后依次调度。

另外，进程状态也被简化了，首先按照这个设计，进程不会被释放，因此进入ZOMBIE态就不会再有任何操作了。

最后关于进程间通信，是可以用message互相通信、并且阻塞等待消息的，发送方发消息之后也会阻塞等待回复。虽然简单不过还勉强可以。当然至于用户进程间权限管理……看这个架构就知道没办法做的吧……

### 分页

页表的实现也很简单：

```
unsigned int level_2_page_table_mappings[MAX_LEVEL_2_PAGE_TABLE_MAPPINGS];
unsigned int level_1_page_table_mappings[MAX_LEVEL_1_PAGE_TABLE_MAPPINGS];
```

这就是页表了……虽然说是二级页表但是好像根本没有起到二级页表应有的作用，因为内存明明还是全都占掉了……

页表的操作也是有启动阶段的，似乎在编译器中做了分段，然后最开始是把内核的每段都设上自映射，然后再打开分页功能。然而问题在于好像现在这个自映射就已经把所有代码的页都分配完了，于是所有东西全都在内存里，没有换入换出也没有缺失，连缺失处理代码都没有……

## 用户态程序

用户态程序在user\_proc.c里……基本上都很简单，只是处理中断而已。但是不是很理解为什么每个处理要分成notifier和server，中断到时是notifier先响应，给server发消息激活server。看上去似乎是想考虑多个中断来源调度的问题。

比较有趣的是有一个do\_compile进程在filesystem.c里，似乎是想要在操作系统里调用编译器(!)编译外部文件来实现加载，这个想法还是不错的不过目前没有启用，不知道为什么。