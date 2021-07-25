# LENS: A Low-level NVRAM Profiler 代码笔记

## len.sh脚本运行基本流程
- mount
    - 编译，得到repfs.ko和latfs.ko
        - latfs.ko：lat.o proc.o misc.o memaccess.o tasks.o chasing.o，主要是命令的解析和执行
  benchmark
        - repfs.ko：rep.o 主要是result record
    - 加载上述内核模块
    - Mount ReportFS and LatencyFS
    
- launch
    - prepare：disable DVFS, disable cache prefetcher, Load and start EMon (需要yum install msr-tools来执行wrmsr和rdmse)
    - 执行具体的test脚本，如pointer_chasing.sh
        - 根据test场景设置具体task参数lens_arg
        - echo $lens_arg >/proc/lens
        - 在lat.c的module_init中，proc_create创建/proc/lens作为跟launch启动脚本交互的虚拟文件，并注册了对应的读写操作函数
            - .open为打印lens_help
            - .write为latencyfs_parse_cmd中对cmd命令进行解析，写入latency_sbi
            - Sanity checks
            - latencyfs_new_threads创建新线程，传入当前task在task.c中对应的执行方法，具体task细节见下
    - 通过解析dmesg来获取执行过程log
    - py脚本parse log获取实验结果
        - 与论文中有出入，需要对应调整

- unmount
        
## pointer_chasing_write_job: Pointer chasing write ONLY task.

1. 解析存储在sbi中的相关参数如region_size, block_size, 工作区范围等

2. 根据block_size确定chasing function，每种block size对应相应load和store function

3. 决定test round times，以GLOBAL_WORKSET / region_size为基准

4. 初始化page table

5. 在reportfs中生成pointer chasing index，即在一个region中不同block的chase次序，存储在cindex中。使用硬件随机数指令**RdRand**

6. 开始benchmark(time measurement略去)

- prepare
    - drop cache：使用**wbinvd**将cache中数据同步到内存中后清空cache，后**mfence**
    - **local_irq_save**, **local_irq_disable** 关中断
    - **kernel_fpu_begin**
- PC_BEFORE_WRITE（主要做的是time measurement，以下不再解释）
    - **rdtscp** 将Time-Stamp Counter和Processor ID读入rdx, rax, rcx寄存器
    - **lfence**
    - "mov %%edx, %[hi]\n\t" "mov %%eax, %[lo]\n\t" 记录时间
- chasing_stnt_##PCBLOCK_SIZE进行写操作，内联汇编代码见下
     - 首先使用**xor** 将相关寄存器置零，其中r8记录当前待写region的offset, r11记录counter，即当前已写过的region个数，当counter达到传入值时终止外层循环
        - 外层循环：
            - 首先使用**lea**，通过start_addr和当前offset得到当前region的起始地址，存放在r9寄存器中
            - 使用**xor** 将相关寄存器置零，其中r10记录当前已经访问过的空间大小，达到region_size后即退出内层循环；r12记录当前block的index
            - 内层循环：
                - 访问cindex，使用**vmovq** 将下一个block的index存入xmm0寄存器中
                - 根据block_size大小，通过**shl**将r12中的index左移一定位数，如block_size = 64则左移6位，得到当前block相对region起始位置的偏移地址
                - 根据block_size的大小进行写操作。
                    - 对于block = 64，使用**vmovntdq** 将zmm0寄存器中的值写入内存，对应地址为(64 * (" #cl_index "))(%%r9, %%r12)\n"，即当前region的起始地址加上当前block的offset；
                    - 对于block = 128，分别进行两次cl_index = 0和1的上述过程，即连续写两个相邻的64block，更大的block_size以此类推。
                - 使用**add**对存放在r10中的当前已访问空间大小加上一个block_size
                - 使用**movq**将存放在xmm0寄存器中的下一个block的index更新入r12
                - **cmp**判断当前已访问空间大小是否达到region_size，若达到则**mfence**，否则继续内层循环
            - 使用**add** 和 **inc** 更新r8存放的当前region的offset以及r11存放的counter
            - **cmp**判断counter是否达到传入值，若达到则任务完成，否则继续外层循环
- PC_BEFORE_READ， PC_AFTER_READ。在本task中没有read任务，无操作执行
- end
    - **kernel_fpu_end**
    - **local_irq_restore**, **local_irq_enable** 开中断
    
```
static void chasing_stnt_##PCBLOCK_SIZE(char *start_addr, long size,           \
					long skip, long count,                 \
					uint64_t *cindex)                      \
{                                                                              \
	KERNEL_BEGIN                                                           \
	asm volatile(                                                          \
		"xor    %%r8, %%r8 \n"                 /* r8: access offset  */\
		"xor    %%r11, %%r11 \n"               /* r11: counter       */\
		                                                               \
	"LOOP_CHASING_ST_NT_" #PCBLOCK_SIZE "_OUTER: \n"                       \
		"lea    (%[start_addr], %%r8), %%r9 \n"/* r9: access loc     */\
		"xor    %%r10, %%r10 \n"               /* r10: accessed size */\
		"xor	%%r12, %%r12 \n"               /* r12: chasing index */\
		                                                               \
	"LOOP_CHASING_ST_NT_" #PCBLOCK_SIZE "_INNER: \n"                       \
		"vmovq  (%[cindex], %%r12, 8), %%xmm0\n"                       \
		"shl	$" #PCBLOCK_SHIFT ", %%r12\n"                          \
		CHASING_ST_##PCBLOCK_SIZE(0)                                   \
		"add	$" #PCBLOCK_SIZE ", %%r10\n"                           \
		"movq	%%xmm0, %%r12\n"                                       \
								               \
		"cmp    %[accesssize], %%r10 \n"                               \
		"jl     LOOP_CHASING_ST_NT_" #PCBLOCK_SIZE "_INNER \n"         \
		CHASING_ST_FENCE                                               \
								               \
		"add    %[skip], %%r8 \n"                                      \
		"inc    %%r11 \n"                                              \
		"cmp    %[count], %%r11 \n"                                    \
		"jl     LOOP_CHASING_ST_NT_" #PCBLOCK_SIZE "_OUTER \n"         \
								               \
		:                                                              \
		: [start_addr] "r"(start_addr),                                \
		  [accesssize] "r"(size), [count] "r"(count),                  \
		  [skip] "r"(skip), [cindex] "r"(cindex)                       \
		: "%r12", "%r11", "%r10", "%r9", "%r8"                         \
	);                                                                     \
	KERNEL_END                                                             \
};
```
            
            
## pointer_chasing_read_and_write_job: Pointer chasing read AND write task.
与pointer_chasing_write_job的区别是在所有region写操作结束后以相同次序对regions进行逐block的读操作。具体流程如下（以16KB region_size和64 Byte block_size为例）：
- Record cycle `c_store_start`
- Run `chasing_stnt_64` function on many consecutive 16KB regions (thenumber of total regions is calculated as `count`)
- Run `mfence`
- Record cycle `c_load_start`
- Run `chasing_ldnt_64` function on all the 16KB regions in step 1
- Run `mfence`
- Record cycle `c_load_end`

task job的基本流程和chasing_stnt_##block_size如上介绍，下面单独介绍本例用到的chasing_ldn_##PCBLOCK_SIZE：

- chasing_ldnt_##PCBLOCK_SIZE进行读操作，内联汇编代码见下
    - 首先使用**xor** 将相关寄存器置零，其中r8记录当前待读region的offset, r11记录counter，即当前已读过的region个数，当counter达到传入值时终止外层循环
        - 外层循环：
            - 首先使用**lea**，通过start_addr和当前offset得到当前region的起始地址，存放在r9寄存器中
            - 使用**xor** 将相关寄存器置零，其中r10记录当前已经访问过的空间大小，达到region_size后即退出内层循环；r12记录当前block的index
            - 内层循环：
                - 访问cindex，使用**vmovq** 将下一个block的index存入xmm0寄存器中（实际源代码中是不是少了这一步？）
                - 根据block_size大小，通过**shl**将r12中的index左移一定位数，如block_size = 64则左移6位，得到当前block相对region起始位置的偏移地址
                - 根据block_size的大小进行读操作。
                    - 对于block = 64，使用**vmovntdq** 将内存中对应地址的值读入zmm0寄存器，对应地址为(64 * (" #cl_index "))(%%r9, %%r12)\n"，即当前region的起始地址加上当前block的offset；
                    - 对于block = 128，分别进行两次cl_index = 0和1的上述过程，即连续读两个相邻的64block，更大的block_size以此类推。
                - 使用**add**对存放在r10中的当前已访问空间大小加上一个block_size
                - 使用**movq**将存放在xmm0寄存器中的下一个block的index更新入r12
                - **cmp**判断当前已访问空间大小是否达到region_size，若达到则**mfence**，否则继续内层循环
            - 使用**add** 和 **inc** 更新r8存放的当前region的offset以及r11存放的counter
            - **cmp**判断counter是否达到传入值，若达到则任务完成，否则继续外层循环


```
static void chasing_ldnt_##PCBLOCK_SIZE(char *start_addr, long size,           \
					long skip, long count,                 \
					uint64_t *cindex)                      \
{                                                                              \
	KERNEL_BEGIN                                                           \
	asm volatile(                                                          \
		"xor    %%r8, %%r8 \n"                 /* r8: access offset  */\
		"xor    %%r11, %%r11 \n"               /* r11: counter       */\
		                                                               \
	"LOOP_CHASING_LD_NT_" #PCBLOCK_SIZE "_OUTER: \n"                       \
		"lea    (%[start_addr], %%r8), %%r9 \n"/* r9: access loc     */\
		"xor    %%r10, %%r10 \n"               /* r10: accessed size */\
		"xor	%%r12, %%r12 \n"               /* r12: chasing index */\
		                                                               \
	"LOOP_CHASING_LD_NT_" #PCBLOCK_SIZE "_INNER: \n"                       \
		"shl	$" #PCBLOCK_SHIFT ", %%r12\n"                          \
		CHASING_LD_##PCBLOCK_SIZE(0)                                   \
		"add	$" #PCBLOCK_SIZE ", %%r10\n"                           \
		"movq	%%xmm0, %%r12\n"                                       \
								               \
		"cmp    %[accesssize], %%r10 \n"                               \
		"jl     LOOP_CHASING_LD_NT_" #PCBLOCK_SIZE "_INNER \n"         \
		CHASING_LD_FENCE                                               \
								               \
		"add    %[skip], %%r8 \n"                                      \
		"inc    %%r11 \n"                                              \
		"cmp    %[count], %%r11 \n"                                    \
								               \
		"jl     LOOP_CHASING_LD_NT_" #PCBLOCK_SIZE "_OUTER \n"         \
								               \
		:                                                              \
		: [start_addr] "r"(start_addr),                                \
		  [accesssize] "r"(size), [count] "r"(count),                  \
		  [skip] "r"(skip), [cindex] "r"(cindex)                       \
		: "%r12", "%r11", "%r10", "%r9", "%r8"                         \
	);                                                                     \
	KERNEL_END                                                             \
}
 
```

## pointer_chasing_read_after_write_job：Pointer chasing read AFTER write task
与pointer_chasing_read_and_write_job的区别是在对一个特定的region完成逐block写之后接着对这个region再进行逐block读操作。具体流程如下（以16KB region_size和64 Byte block_size为例）：
- Record cycle `c_store_start`
- Run `chasing_stnt_64` function on ONE 16KB region
- Run `mfence`
- Run `chasing_ldnt_64` function on the 16KB region in step 1
- Run `mfence`
- Repeat step 1-4 for `count` rounds, on different 16KB regions
- Record cycle `c_load_start`
- Record cycle `c_load_end`

下面单独介绍本例用到的chasing_read_after_write_##PCBLOCK_SIZE：

-  chasing_read_after_write_##PCBLOCK_SIZE进行写后读操作，内联汇编代码见下
    - 首先使用**xor** 将相关寄存器置零，其中r8记录当前region的offset, r11记录counter，即当前已操作过的region个数，当counter达到传入值时终止外层循环
        - 外层循环：
            - 首先使用**lea**，通过start_addr和当前offset得到当前region的起始地址，存放在r9寄存器中
            - 使用**xor** 将相关寄存器置零，其中r10记录当前已经访问过的空间大小，达到region_size后即退出内层循环；r12记录当前block的index
            - 内层写循环：
                - 访问cindex，使用**vmovq** 将下一个待写block的index存入xmm0寄存器中
                - 根据block_size大小，通过**shl**将r12中的index左移一定位数，如block_size = 64则左移6位，得到当前block相对region起始位置的偏移地址
                - 根据block_size的大小进行写操作，具体方法和上述task相同位置一致。
                - 使用**add**对存放在r10中的当前已访问空间大小加上一个block_size
                - 使用**movq**将存放在xmm0寄存器中的下一个block的index更新入r12
                - **cmp**判断当前已写空间大小是否达到region_size，若达到则**mfence**进入内层读循环，否则继续内层写循环
            - 内层读循环
                - 访问cindex，使用**vmovq** 将下一个待读block的index存入xmm0寄存器中
                - 根据block_size大小，通过**shl**将r12中的index左移一定位数，如block_size = 64则左移6位，得到当前block相对region起始位置的偏移地址
                - 根据block_size的大小进行读操作，具体方法和上述task相同位置一致。
                - 使用**add**对存放在r10中的当前已访问空间大小加上一个block_size
                - 使用**movq**将存放在xmm0寄存器中的下一个block的index更新入r12
                - **cmp**判断当前已读空间大小是否达到region_size，若达到则**mfence**，否则继续内层读循环
            - 使用**add** 和 **inc** 更新r8存放的当前region的offset以及r11存放的counter
            - **cmp**判断counter是否达到传入值，若达到则任务完成，否则继续外层循环

```
static void chasing_read_after_write_##PCBLOCK_SIZE(char *start_addr,          \
						    long size,                 \
						    long skip,                 \
						    long count,	               \
						    uint64_t *cindex)          \
{                                                                              \
	KERNEL_BEGIN                                                           \
	asm volatile(                                                          \
		"xor    %%r8, %%r8 \n"                 /* r8: access offset  */\
		"xor    %%r11, %%r11 \n"               /* r11: counter       */\
		                                                               \
	"LOOP_CHASING_RAW_NT_" #PCBLOCK_SIZE "_OUTER: \n"                      \
		"lea    (%[start_addr], %%r8), %%r9 \n"/* r9: access loc     */\
		"xor    %%r10, %%r10 \n"               /* r10: accessed size */\
		"xor	%%r12, %%r12 \n"               /* r12: chasing index */\
		                                                               \
	"LOOP_CHASING_RAW_NT_WRITE_" #PCBLOCK_SIZE "_INNER: \n"                \
		"vmovq  (%[cindex], %%r12, 8), %%xmm0\n"                       \
		"shl	$" #PCBLOCK_SHIFT ", %%r12\n"                          \
		CHASING_ST_##PCBLOCK_SIZE(0)                                   \
		"add	$" #PCBLOCK_SIZE ", %%r10\n"                           \
		"movq	%%xmm0, %%r12\n"                                       \
								               \
		"cmp    %[accesssize], %%r10 \n"                               \
		"jl     LOOP_CHASING_RAW_NT_WRITE_" #PCBLOCK_SIZE "_INNER \n"  \
		CHASING_MFENCE                                                 \
								               \
		"xor    %%r10, %%r10 \n"               /* r10: accessed size */\
		"xor	%%r12, %%r12 \n"               /* r12: chasing index */\
	"LOOP_CHASING_RAW_NT_READ_" #PCBLOCK_SIZE "_INNER: \n"                 \
		"shl	$" #PCBLOCK_SHIFT ", %%r12\n"                          \
		CHASING_LD_##PCBLOCK_SIZE(0)                                   \
		"add	$" #PCBLOCK_SIZE ", %%r10\n"                           \
		"movq	%%xmm0, %%r12\n"                                       \
								               \
		"cmp    %[accesssize], %%r10 \n"                               \
		"jl     LOOP_CHASING_RAW_NT_READ_" #PCBLOCK_SIZE "_INNER \n"   \
		CHASING_MFENCE                                                 \
									       \
		"add    %[skip], %%r8 \n"                                      \
		"inc    %%r11 \n"                                              \
		"cmp    %[count], %%r11 \n"                                    \
								               \
		"jl     LOOP_CHASING_RAW_NT_" #PCBLOCK_SIZE "_OUTER \n"        \
								               \
		:                                                              \
		: [start_addr] "r"(start_addr),                                \
		  [accesssize] "r"(size), [count] "r"(count),                  \
		  [skip] "r"(skip), [cindex] "r"(cindex)                       \
		: "%r12", "%r11", "%r10", "%r9", "%r8"                         \
	);                                                                     \
	KERNEL_END                                                             \
}

```
## Task与Hardware Behavior对应关系

pointer_chasing_read_and_write_job 对应 Buffer overflow和R/W amplification

pointer_chasing_read_after_write_job 对应 Data fast-forwarding



  
