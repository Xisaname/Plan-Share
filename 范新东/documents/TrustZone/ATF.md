目录
- [ATF(Arm Trusted Firmware)](#atfarm-trusted-firmware)
- [HighLevel](#highlevel)
	- [ATF是什么](#atf是什么)
	- [ATF由哪些组件构成](#atf由哪些组件构成)
- [BL1](#bl1)
	- [bl1\_entrypoint](#bl1_entrypoint)
	- [el3\_entrypoint\_common](#el3_entrypoint_common)
	- [bl1\_early\_patform\_setup](#bl1_early_patform_setup)
	- [bl\_main](#bl_main)
	- [bl1\_prepare\_next\_image](#bl1_prepare_next_image)
	- [BL1 to BL2](#bl1-to-bl2)
- [BL2](#bl2)
	- [bl2\_entrypoint](#bl2_entrypoint)
	- [bl2\_main](#bl2_main)
	- [bl2\_load\_images](#bl2_load_images)
	- [REGISTER\_BL\_IMAGE\_DESCS(bl2\_mem\_params\_descs)](#register_bl_image_descsbl2_mem_params_descs)
	- [BL2 to BL3](#bl2-to-bl3)
- [BL31](#bl31)
	- [bl31\_entrypoint](#bl31_entrypoint)
	- [bl31\_main](#bl31_main)
	- [runtime\_svc\_init](#runtime_svc_init)
	- [DECLARE\_RT\_SVC](#declare_rt_svc)
	- [BL31 to BL32](#bl31-to-bl32)
		- [OPTEE OS实现](#optee-os实现)
- [BL32](#bl32)
	- [opteed\_setup](#opteed_setup)
	- [opteed\_init](#opteed_init)
	- [opteed\_enter\_sp](#opteed_enter_sp)
	- [BL32 to BL 33](#bl32-to-bl-33)
# ATF(Arm Trusted Firmware)
ATF（ARM Trusted Firmware）是一针对ARM芯片给出的底层的开源固件代码。固件将整个系统分成四种运行等级，分别为：EL0,EL1,EL2,EL3，并规定了每个安全等级中运行的Image名字。

[ATF源代码地址](https://github.com/ARM-software/arm-trusted-firmware)

本文将介绍在ARM芯片冷启动时ATF的具体运行过程以及各层级之间的跳转过程。

# HighLevel
首先先总结ATF是什么？它由哪些组件构成？每个组件的作用又是什么？各个组件之间的关系是怎样的？各个组件的安全性到底如何？

## ATF是什么
在ATF出现之前，一个计算机启动的流程通常是由存储在非易失性存储器中的代码进行最开始的初始化，之后加载并将控制权交给更大的镜像，去负责更复杂功能的初始化，这样一步一步按照顺序最终加载我们熟悉的操作系统以及应用程序。

整个启动过程像是一个信息传递的过程。但是信息在其中是以明文方式存储的，随着信息安全三要素：机密性、完整性以及可用性的提出，传递明文信息的各个引导流程逐渐变得危险起来。

首先，明文信息无法达到机密性的要求。
其次，信息在整个传递过程中甚至是在存储阶段就已经遭到修改，但是整个引导启动过程还是将其当做正常程序执行，显然让恶意程序有机可乘，严重危害系统安全。
最后，在引导过程中，如何对已经启动过的程序进行保护，使得这些程序不受攻击的影响而正常工作，这就是信息的可用性。
显然，普通的启动流程无法满足信息安全三要素。

ATF的作用就是解决上述启动过程中的安全问题。

首先，它将最开始进行初始化的代码存储到芯片中的ROM内，外界难以访问，并且不能修改。
其次，在引导过程中，每一个引导阶段，在加载下一段引导程序时验证其完整性。
最后，在引导加载后，它提供运行时的安全防护，保证在受到攻击时整个计算机也能正常运行。

可以看到ATF重构了计算机启动的流程。但是，上层启动流程并没有改变，因此ATF实现了与之前引导流程一样的API，供上层调用。

## ATF由哪些组件构成




# BL1
BL1是系统上电之后第一批启动的层。在其启动之前，上电后会首先运行SCP boot ROM，这个是存在片上的ROM程序，是信任链的起始点，默认无条件安全。之后会跳转到ATF的BL1中继续执行。

BL1的主要工作是初始化CPU、设定异常向量、将BL2中的image加载到安全的ARM中，然后跳转到BL2中进行执行。

bl1的主要代码存放在bl1目录中， bl1的连接脚本是`bl1/bl1.ld.s`文件，其中可以看到bl1的入口函数是:`bl1_entrypoint`。

## bl1_entrypoint
该函数主要需要执行EL3环境的基本初始化，设定向量表，加载bl2 image并跳转到bl2等操作

```C++
func bl1_entrypoint
	/* ---------------------------------------------------------------------
	 * If the reset address is programmable then bl1_entrypoint() is
	 * executed only on the cold boot path. Therefore, we can skip the warm
	 * boot mailbox mechanism.
	 * ---------------------------------------------------------------------
	 */
/* EL3级别运行环境的初始化，该函数定义在   include/common/aarch64/el3_common_macros.S文件中
*/
	el3_entrypoint_common					\
		_set_endian=1					\
		_warm_boot_mailbox=!PROGRAMMABLE_RESET_ADDRESS	\
		_secondary_cold_boot=!COLD_BOOT_SINGLE_CPU	\
		_init_memory=1					\
		_init_c_runtime=1				\
		_exception_vectors=bl1_exceptions
 
	/* ---------------------------------------------
	 * Architectural init. can be generic e.g.
	 * enabling stack alignment and platform spec-
	 * ific e.g. MMU & page table setup as per the
	 * platform memory map. Perform the latter here
	 * and the former in bl1_main.
	 * ---------------------------------------------
	 */
	bl	bl1_early_platform_setup	//调用bl1_early_platform_setup函数完成底层初始化
	bl	bl1_plat_arch_setup	//调用bl1_plat_arch_setup完成平台初始化
 
	/* --------------------------------------------------
	 * Initialize platform and jump to our c-entry point
	 * for this type of reset.
	 * --------------------------------------------------
	 */
	bl	bl1_main	//调用bl1_main函数，初始化验证模块，加载下一阶段的image到RAM中
 
	/* --------------------------------------------------
	 * Do the transition to next boot image.
	 * --------------------------------------------------
	 */
	b	el3_exit	//调用el3_exit函数，跳转到下一个image(bl2)
endfunc bl1_entrypoint
```

## el3_entrypoint_common
该函数是以宏的形式被定义的，主要完成el3基本设置和向量表注册
```C++
.macro el3_entrypoint_common					\
		_set_endian, _warm_boot_mailbox, _secondary_cold_boot,	\
		_init_memory, _init_c_runtime, _exception_vectors
/* 设定大小端 */
	.if \_set_endian
		/* -------------------------------------------------------------
		 * Set the CPU endianness before doing anything that might
		 * involve memory reads or writes.
		 * -------------------------------------------------------------
		 */
		mrs	x0, sctlr_el3
		bic	x0, x0, #SCTLR_EE_BIT
		msr	sctlr_el3, x0
		isb
	.endif /* _set_endian */
 
/* 判定是否需要调用do_cold_boot流程 */
	.if \_warm_boot_mailbox
		/* -------------------------------------------------------------
		 * This code will be executed for both warm and cold resets.
		 * Now is the time to distinguish between the two.
		 * Query the platform entrypoint address and if it is not zero
		 * then it means it is a warm boot so jump to this address.
		 * -------------------------------------------------------------
		 */
		bl	plat_get_my_entrypoint
		cbz	x0, do_cold_boot
		br	x0
 
	do_cold_boot:
	.endif /* _warm_boot_mailbox */
 
	/* ---------------------------------------------------------------------
	 * It is a cold boot.
	 * Perform any processor specific actions upon reset e.g. cache, TLB
	 * invalidations etc.
	 * ---------------------------------------------------------------------
	 */
	bl	reset_handler		//执行reset handle操作
 
/* 初始化异常向量 */
	el3_arch_init_common \_exception_vectors
 
/* 判定当前CPU是否是主CPU，如果是则做主CPU的初始化 */
	.if \_secondary_cold_boot
		/* -------------------------------------------------------------
		 * Check if this is a primary or secondary CPU cold boot.
		 * The primary CPU will set up the platform while the
		 * secondaries are placed in a platform-specific state until the
		 * primary CPU performs the necessary actions to bring them out
		 * of that state and allows entry into the OS.
		 * -------------------------------------------------------------
		 */
		bl	plat_is_my_cpu_primary
		cbnz	w0, do_primary_cold_boot
 
		/* This is a cold boot on a secondary CPU */
		bl	plat_secondary_cold_boot_setup
		/* plat_secondary_cold_boot_setup() is not supposed to return */
		bl	el3_panic
 
	do_primary_cold_boot:
	.endif /* _secondary_cold_boot */
 
	/* ---------------------------------------------------------------------
	 * Initialize memory now. Secondary CPU initialization won't get to this
	 * point.
	 * ---------------------------------------------------------------------
	 */
/* 初始化memory */
	.if \_init_memory
		bl	platform_mem_init
	.endif /* _init_memory */
 
	/* ---------------------------------------------------------------------
	 * Init C runtime environment:
	 *   - Zero-initialise the NOBITS sections. There are 2 of them:
	 *       - the .bss section;
	 *       - the coherent memory section (if any).
	 *   - Relocate the data section from ROM to RAM, if required.
	 * ---------------------------------------------------------------------
	 */
/* 初始化C语言的运行环境 */
	.if \_init_c_runtime
#ifdef IMAGE_BL31
		/* -------------------------------------------------------------
		 * Invalidate the RW memory used by the BL31 image. This
		 * includes the data and NOBITS sections. This is done to
		 * safeguard against possible corruption of this memory by
		 * dirty cache lines in a system cache as a result of use by
		 * an earlier boot loader stage.
		 * -------------------------------------------------------------
		 */
		adr	x0, __RW_START__
		adr	x1, __RW_END__
		sub	x1, x1, x0
		bl	inv_dcache_range
#endif /* IMAGE_BL31 */
 
		ldr	x0, =__BSS_START__
		ldr	x1, =__BSS_SIZE__
		bl	zeromem
 
#if USE_COHERENT_MEM
		ldr	x0, =__COHERENT_RAM_START__
		ldr	x1, =__COHERENT_RAM_UNALIGNED_SIZE__
		bl	zeromem
#endif
 
#ifdef IMAGE_BL1
		ldr	x0, =__DATA_RAM_START__
		ldr	x1, =__DATA_ROM_START__
		ldr	x2, =__DATA_SIZE__
		bl	memcpy16
#endif
	.endif /* _init_c_runtime */
 
	/* ---------------------------------------------------------------------
	 * Use SP_EL0 for the C runtime stack.
	 * ---------------------------------------------------------------------
	 */
	msr	spsel, #0
 
	/* ---------------------------------------------------------------------
	 * Allocate a stack whose memory will be marked as Normal-IS-WBWA when
	 * the MMU is enabled. There is no risk of reading stale stack memory
	 * after enabling the MMU as only the primary CPU is running at the
	 * moment.
	 * ---------------------------------------------------------------------
	 */
	bl	plat_set_my_stack	//设定堆栈
 
#if STACK_PROTECTOR_ENABLED
	.if \_init_c_runtime
	bl	update_stack_protector_canary
	.endif /* _init_c_runtime */
#endif
	.endm
 
#endif /* __EL3_COMMON_MACROS_S__ */
```
该函数是需要带参数调用，参数说明如下：

+ `_set_endian`：设定大小端
+ `_warm_boot_mailbox`：检查当前是属于冷启动还是热启动(power on or reset)
+ `_secondary_cold_boot`: 确定当前的CPU是主CPU还是从属CPU
+ `_init_memory`：是否需要初始化memory
+ `_init_c_runtime`: 是否需要初始化C语言的执行环境
+ `_exception_vectors`: 异常向量表地址

## bl1_early_patform_setup
该函数用来完成早期的初始化操作，主要包括memory, page table, 所需外围设备的初始化以及相关状态设定等。
```C++
void bl1_early_platform_setup(void)
{
/* 使能看门狗，初始化console,初始化memory */
	arm_bl1_early_platform_setup();
 
	/*
	 * Initialize Interconnect for this cluster during cold boot.
	 * No need for locks as no other CPU is active.
	 */
	plat_arm_interconnect_init();//初始化外围设备
	/*
	 * Enable Interconnect coherency for the primary CPU's cluster.
	 */
	plat_arm_interconnect_enter_coherency();//使能外围设备
}
```

## bl_main
该函数完成bl2 image的加载和运行环境的设置，如果开启了trusted boot，则需要对image进行verify操作
```C++
void bl1_main(void)
{
	unsigned int image_id;
 
	/* Announce our arrival */
	NOTICE(FIRMWARE_WELCOME_STR);
	NOTICE("BL1: %s\n", version_string);
	NOTICE("BL1: %s\n", build_message);
 
	INFO("BL1: RAM %p - %p\n", (void *)BL1_RAM_BASE,
					(void *)BL1_RAM_LIMIT);
 
	print_errata_status();
 
#if DEBUG
	u_register_t val;
	/*
	 * Ensure that MMU/Caches and coherency are turned on
	 */
#ifdef AARCH32
	val = read_sctlr();
#else
	val = read_sctlr_el3();
#endif
	assert(val & SCTLR_M_BIT);
	assert(val & SCTLR_C_BIT);
	assert(val & SCTLR_I_BIT);
	/*
	 * Check that Cache Writeback Granule (CWG) in CTR_EL0 matches the
	 * provided platform value
	 */
	val = (read_ctr_el0() >> CTR_CWG_SHIFT) & CTR_CWG_MASK;
	/*
	 * If CWG is zero, then no CWG information is available but we can
	 * at least check the platform value is less than the architectural
	 * maximum.
	 */
	if (val != 0)
		assert(CACHE_WRITEBACK_GRANULE == SIZE_FROM_LOG2_WORDS(val));
	else
		assert(CACHE_WRITEBACK_GRANULE <= MAX_CACHE_LINE_SIZE);
#endif
 
	/* Perform remaining generic architectural setup from EL3 */
	bl1_arch_setup();	//设置下一个image的EL级别
 
#if TRUSTED_BOARD_BOOT
	/* Initialize authentication module */
	auth_mod_init();	//初始化image的验证模块
#endif /* TRUSTED_BOARD_BOOT */
 
	/* Perform platform setup in BL1. */
	bl1_platform_setup();	//平台相关设置，主要是IO的设置
 
	/* Get the image id of next image to load and run. */
	image_id = bl1_plat_get_next_image_id();	//获取下一个阶段image的ID值。默认返回值为BL2_IMAGE_ID
 
	/*
	 * We currently interpret any image id other than
	 * BL2_IMAGE_ID as the start of firmware update.
	 */
	if (image_id == BL2_IMAGE_ID)
		bl1_load_bl2();		//将bl2 image加载到安全RAM中
	else
		NOTICE("BL1-FWU: *******FWU Process Started*******\n");
 
	bl1_prepare_next_image(image_id);	//获取bl2 image的描述信息，包括名字，ID，entry potin info等，并将这些信息保存到bl1_cpu_context的上下文中
	console_flush();
}
```

## bl1_prepare_next_image
该函数用来获取bl2 image的描述信息，获取bl2的入口地址，这只下个阶段的CPU上下文，以备执行从bl1跳转到bl2的操作使用
```C++
void bl1_prepare_next_image(unsigned int image_id)
{
	unsigned int security_state;
	image_desc_t *image_desc;
	entry_point_info_t *next_bl_ep;
 
#if CTX_INCLUDE_AARCH32_REGS
	/*
	 * Ensure that the build flag to save AArch32 system registers in CPU
	 * context is not set for AArch64-only platforms.
	 */
	if (((read_id_aa64pfr0_el1() >> ID_AA64PFR0_EL1_SHIFT)
			& ID_AA64PFR0_ELX_MASK) == 0x1) {
		ERROR("EL1 supports AArch64-only. Please set build flag "
				"CTX_INCLUDE_AARCH32_REGS = 0");
		panic();
	}
#endif
 
	/* Get the image descriptor. */
/* 获取bl2 image的描述信息，主要包括入口地址，名字等信息 */
	image_desc = bl1_plat_get_image_desc(image_id);
	assert(image_desc);
 
	/* Get the entry point info. */
/* 获取image的入口地址信息 */
	next_bl_ep = &image_desc->ep_info;
 
	/* Get the image security state. */
/* 获取bl2 image的安全状态（判定该image是属于安全态的image的还是非安全态的image） */
	security_state = GET_SECURITY_STATE(next_bl_ep->h.attr);
 
	/* Setup the Secure/Non-Secure context if not done already. */
/* 设定用于存放CPU context的变量 */
	if (!cm_get_context(security_state))
		cm_set_context(&bl1_cpu_context[security_state], security_state);
 
	/* Prepare the SPSR for the next BL image. */
/* 为下个阶段的image准备好SPSR数据 */
	if (security_state == SECURE) {
		next_bl_ep->spsr = SPSR_64(MODE_EL1, MODE_SP_ELX,
				   DISABLE_ALL_EXCEPTIONS);
	} else {
		/* Use EL2 if supported else use EL1. */
		if (read_id_aa64pfr0_el1() &
			(ID_AA64PFR0_ELX_MASK << ID_AA64PFR0_EL2_SHIFT)) {
			next_bl_ep->spsr = SPSR_64(MODE_EL2, MODE_SP_ELX,
				DISABLE_ALL_EXCEPTIONS);
		} else {
			next_bl_ep->spsr = SPSR_64(MODE_EL1, MODE_SP_ELX,
			   DISABLE_ALL_EXCEPTIONS);
		}
	}
	/* Allow platform to make change */
	bl1_plat_set_ep_info(image_id, next_bl_ep);
 
	/* Prepare the context for the next BL image. */
/* 使用获取到的bl2 image的entrypoint info数据来初始化cpu context */
	cm_init_my_context(next_bl_ep);
/* 为进入到下个EL级别做准备 */
	cm_prepare_el3_exit(security_state);
 
	/* Indicate that image is in execution state. */
/* 设定image的执行状态 */
	image_desc->state = IMAGE_STATE_EXECUTED;
 
/* 打印出bl2 image的入口信息 */
	print_entry_point_info(next_bl_ep);
}
```
## BL1 to BL2
在bl1完成了bl2 image加载到RAM中的操作，中断向量表设定以及其他CPU相关设定之后，在`bl1_main`函数中解析出bl2 image的描述信息，获取入口地址，并设定下一个阶段的cpu上下文，完成之后，调用`el3_exit`函数实现bl1到bl2的跳转操作，进入到bl2中执行.

# BL2
BL2 image将会为后续image的加载执行相关的初始化操作。主要是内存，MMU，串口以及EL3软件运行环境的设置，并且加载bl3x的image到RAM中。通过查看`bl2.ld.S`文件就可以发现，bl2 image的入口函数是`bl2_entrypoint`。该函数定义在`bl2/aarch64/bl2_entrypoint.S`文件中。

## bl2_entrypoint
该函数的内容如下，该函数最终会出发smc操作，从bl1中将CPU的控制权转交给bl31:
```C++
func bl2_entrypoint
	/*---------------------------------------------
	 * Save from x1 the extents of the tzram
	 * available to BL2 for future use.
	 * x0 is not currently used.
	 * ---------------------------------------------
	 */
	mov	x20, x1
 
	/* ---------------------------------------------
	 * Set the exception vector to something sane.
	 * ---------------------------------------------
	 */
	adr	x0, early_exceptions	//设定异常向量
	msr	vbar_el1, x0
	isb
 
	/* ---------------------------------------------
	 * Enable the SError interrupt now that the
	 * exception vectors have been setup.
	 * ---------------------------------------------
	 */
	msr	daifclr, #DAIF_ABT_BIT
 
	/* ---------------------------------------------
	 * Enable the instruction cache, stack pointer
	 * and data access alignment checks
	 * ---------------------------------------------
	 */
	mov	x1, #(SCTLR_I_BIT | SCTLR_A_BIT | SCTLR_SA_BIT)
	mrs	x0, sctlr_el1
	orr	x0, x0, x1
	msr	sctlr_el1, x0
	isb
 
	/* ---------------------------------------------
	 * Invalidate the RW memory used by the BL2
	 * image. This includes the data and NOBITS
	 * sections. This is done to safeguard against
	 * possible corruption of this memory by dirty
	 * cache lines in a system cache as a result of
	 * use by an earlier boot loader stage.
	 * ---------------------------------------------
	 */
	adr	x0, __RW_START__
	adr	x1, __RW_END__
	sub	x1, x1, x0
	bl	inv_dcache_range
 
	/* ---------------------------------------------
	 * Zero out NOBITS sections. There are 2 of them:
	 *   - the .bss section;
	 *   - the coherent memory section.
	 * ---------------------------------------------
	 */
	ldr	x0, =__BSS_START__
	ldr	x1, =__BSS_SIZE__
	bl	zeromem
 
#if USE_COHERENT_MEM
	ldr	x0, =__COHERENT_RAM_START__
	ldr	x1, =__COHERENT_RAM_UNALIGNED_SIZE__
	bl	zeromem
#endif
 
	/* --------------------------------------------
	 * Allocate a stack whose memory will be marked
	 * as Normal-IS-WBWA when the MMU is enabled.
	 * There is no risk of reading stale stack
	 * memory after enabling the MMU as only the
	 * primary cpu is running at the moment.
	 * --------------------------------------------
	 */
	bl	plat_set_my_stack	//初始化bl2运行的栈
 
	/* ---------------------------------------------
	 * Initialize the stack protector canary before
	 * any C code is called.
	 * ---------------------------------------------
	 */
#if STACK_PROTECTOR_ENABLED
	bl	update_stack_protector_canary
#endif
 
	/* ---------------------------------------------
	 * Perform early platform setup & platform
	 * specific early arch. setup e.g. mmu setup
	 * ---------------------------------------------
	 */
	mov	x0, x20
	bl	bl2_early_platform_setup	//设置平台相关
	bl	bl2_plat_arch_setup	//设置架构相关
 
	/* ---------------------------------------------
	 * Jump to main function.
	 * ---------------------------------------------
	 */
	bl	bl2_main	//跳转到BL2的主要函数执行，从该函数中跳转到bl31以及bl31,
 
	/* ---------------------------------------------
	 * Should never reach this point.
	 * ---------------------------------------------
	 */
	no_ret	plat_panic_handler
 
endfunc bl2_entrypoint
```

## bl2_main
该函数主要实现将bl3x的image加载RAM中，并通过smc调用执行bl1中指定的smc handle将CPU的全向交给bl31。

```C++
void bl2_main(void)
{
	entry_point_info_t *next_bl_ep_info;
 
	NOTICE("BL2: %s\n", version_string);
	NOTICE("BL2: %s\n", build_message);
 
	/* Perform remaining generic architectural setup in S-EL1 */
	bl2_arch_setup();
 
#if TRUSTED_BOARD_BOOT
	/* Initialize authentication module */
	auth_mod_init();	//初始化image验证模块
#endif /* TRUSTED_BOARD_BOOT */
 
	/* Load the subsequent bootloader images. */
	next_bl_ep_info = bl2_load_images();	//加载bl3x image到RAM中并返回bl31的入口地址
 
#ifdef AARCH32
	/*
	 * For AArch32 state BL1 and BL2 share the MMU setup.
	 * Given that BL2 does not map BL1 regions, MMU needs
	 * to be disabled in order to go back to BL1.
	 */
	disable_mmu_icache_secure();
#endif /* AARCH32 */
 
	console_flush();
 
	/*
	 * Run next BL image via an SMC to BL1. Information on how to pass
	 * control to the BL32 (if present) and BL33 software images will
	 * be passed to next BL image as an argument.
	 */
/* 调用smc指令，触发在bl1中设定的smc异常中断处理函数，跳转到bl31 */
	smc(BL1_SMC_RUN_IMAGE, (unsigned long)next_bl_ep_info, 0, 0, 0, 0, 0, 0);
}
```

## bl2_load_images
该函数用来加载bl3x的image到RAM中，返回一个具有image入口信息的变量。smc handle根据该变量跳转到bl31进行执行。

```C++
entry_point_info_t *bl2_load_images(void)
{
	bl_params_t *bl2_to_next_bl_params;
	bl_load_info_t *bl2_load_info;
	const bl_load_info_node_t *bl2_node_info;
	int plat_setup_done = 0;
	int err;
 
	/*
	 * Get information about the images to load.
	 */
/* 获取bl3x image的加载和入口信息 */
	bl2_load_info = plat_get_bl_image_load_info();
 
/* 检查返回的bl2_load_info中的信息是否正确 */
	assert(bl2_load_info);
	assert(bl2_load_info->head);
	assert(bl2_load_info->h.type == PARAM_BL_LOAD_INFO);
	assert(bl2_load_info->h.version >= VERSION_2);
 
/* 将bl2_load_info中的head变量的值赋值为bl2_node_info，即将bl31 image的入口信息传递給bl2_node_info变量 */
	bl2_node_info = bl2_load_info->head;
 
/* 进入loop循环， */
	while (bl2_node_info) {
		/*
		 * Perform platform setup before loading the image,
		 * if indicated in the image attributes AND if NOT
		 * already done before.
		 */
/* 在加载特定的bl3x image到RAM之前先确定是否需要做平台的初始化 */
		if (bl2_node_info->image_info->h.attr & IMAGE_ATTRIB_PLAT_SETUP) {
			if (plat_setup_done) {
				WARN("BL2: Platform setup already done!!\n");
			} else {
				INFO("BL2: Doing platform setup\n");
				bl2_platform_setup();
				plat_setup_done = 1;
			}
		}
 
/* 对bl3x image进行电子验签，如果通过则执行加载操作 */
		if (!(bl2_node_info->image_info->h.attr & IMAGE_ATTRIB_SKIP_LOADING)) {
			INFO("BL2: Loading image id %d\n", bl2_node_info->image_id);
			err = load_auth_image(bl2_node_info->image_id,
				bl2_node_info->image_info);
			if (err) {
				ERROR("BL2: Failed to load image (%i)\n", err);
				plat_error_handler(err);
			}
		} else {
			INFO("BL2: Skip loading image id %d\n", bl2_node_info->image_id);
		}
 
		/* Allow platform to handle image information. */
/* 可以根据实际需要更改，通过给定image ID来更改image的加载信息 */
		err = bl2_plat_handle_post_image_load(bl2_node_info->image_id);
		if (err) {
			ERROR("BL2: Failure in post image load handling (%i)\n", err);
			plat_error_handler(err);
		}
 
		/* Go to next image */
		bl2_node_info = bl2_node_info->next_load_info;
	}
 
	/*
	 * Get information to pass to the next image.
	 */
/* 获取下一个执行的Image的入口信息，并且将以后会被执行的image的入口信息组合成链表 ,t通过判断image des中的ep_info.h.attr的值是否为（EXECUTABLE|EP_FIRST_EX）来确定接下来第一个被执行的image*/
	bl2_to_next_bl_params = plat_get_next_bl_params();
	assert(bl2_to_next_bl_params);
	assert(bl2_to_next_bl_params->head);
	assert(bl2_to_next_bl_params->h.type == PARAM_BL_PARAMS);
	assert(bl2_to_next_bl_params->h.version >= VERSION_2);
 
	/* Flush the parameters to be passed to next image */
	plat_flush_next_bl_params();
 
/* 返回下一个进入的image的入口信息,即bl31的入口信息 */
	return bl2_to_next_bl_params->head->ep_info;
}
```

## REGISTER_BL_IMAGE_DESCS(bl2_mem_params_descs)
该宏的执行将会初始化组成bl2加载bl3x image的列表使用到的重要全局变量，其中bl2_mem_params_descs变量的定义如下：

```C++
static bl_mem_params_node_t bl2_mem_params_descs[] = {
#ifdef SCP_BL2_BASE
	/* Fill SCP_BL2 related information if it exists */
    {
	    .image_id = SCP_BL2_IMAGE_ID,
 
	    SET_STATIC_PARAM_HEAD(ep_info, PARAM_IMAGE_BINARY,
		    VERSION_2, entry_point_info_t, SECURE | NON_EXECUTABLE),
 
	    SET_STATIC_PARAM_HEAD(image_info, PARAM_IMAGE_BINARY,
		    VERSION_2, image_info_t, 0),
	    .image_info.image_base = SCP_BL2_BASE,
	    .image_info.image_max_size = PLAT_CSS_MAX_SCP_BL2_SIZE,
 
	    .next_handoff_image_id = INVALID_IMAGE_ID,
    },
#endif /* SCP_BL2_BASE */
 
#ifdef EL3_PAYLOAD_BASE
	/* Fill EL3 payload related information (BL31 is EL3 payload)*/
    {
	    .image_id = BL31_IMAGE_ID,
 
	    SET_STATIC_PARAM_HEAD(ep_info, PARAM_EP,
		    VERSION_2, entry_point_info_t,
		    SECURE | EXECUTABLE | EP_FIRST_EXE),
	    .ep_info.pc = EL3_PAYLOAD_BASE,
	    .ep_info.spsr = SPSR_64(MODE_EL3, MODE_SP_ELX,
		    DISABLE_ALL_EXCEPTIONS),
 
	    SET_STATIC_PARAM_HEAD(image_info, PARAM_EP,
		    VERSION_2, image_info_t,
		    IMAGE_ATTRIB_PLAT_SETUP | IMAGE_ATTRIB_SKIP_LOADING),
 
	    .next_handoff_image_id = INVALID_IMAGE_ID,
    },
 
#else /* EL3_PAYLOAD_BASE */
 
	/* Fill BL31 related information */
    {
	    .image_id = BL31_IMAGE_ID,
 
	    SET_STATIC_PARAM_HEAD(ep_info, PARAM_EP,
		    VERSION_2, entry_point_info_t,
		    SECURE | EXECUTABLE | EP_FIRST_EXE),
	    .ep_info.pc = BL31_BASE,
	    .ep_info.spsr = SPSR_64(MODE_EL3, MODE_SP_ELX,
		    DISABLE_ALL_EXCEPTIONS),
#if DEBUG
	    .ep_info.args.arg1 = ARM_BL31_PLAT_PARAM_VAL,
#endif
 
	    SET_STATIC_PARAM_HEAD(image_info, PARAM_EP,
		    VERSION_2, image_info_t, IMAGE_ATTRIB_PLAT_SETUP),
	    .image_info.image_base = BL31_BASE,
	    .image_info.image_max_size = BL31_LIMIT - BL31_BASE,
 
# ifdef BL32_BASE
	    .next_handoff_image_id = BL32_IMAGE_ID,
# else
	    .next_handoff_image_id = BL33_IMAGE_ID,
# endif
    },
 
# ifdef BL32_BASE
	/* Fill BL32 related information */
    {
	    .image_id = BL32_IMAGE_ID,
 
	    SET_STATIC_PARAM_HEAD(ep_info, PARAM_EP,
		    VERSION_2, entry_point_info_t, SECURE | EXECUTABLE),
	    .ep_info.pc = BL32_BASE,
 
	    SET_STATIC_PARAM_HEAD(image_info, PARAM_EP,
		    VERSION_2, image_info_t, 0),
	    .image_info.image_base = BL32_BASE,
	    .image_info.image_max_size = BL32_LIMIT - BL32_BASE,
 
	    .next_handoff_image_id = BL33_IMAGE_ID,
    },
# endif /* BL32_BASE */
 
	/* Fill BL33 related information */
    {
	    .image_id = BL33_IMAGE_ID,
	    SET_STATIC_PARAM_HEAD(ep_info, PARAM_EP,
		    VERSION_2, entry_point_info_t, NON_SECURE | EXECUTABLE),
# ifdef PRELOADED_BL33_BASE
	    .ep_info.pc = PRELOADED_BL33_BASE,
 
	    SET_STATIC_PARAM_HEAD(image_info, PARAM_EP,
		    VERSION_2, image_info_t, IMAGE_ATTRIB_SKIP_LOADING),
# else
	    .ep_info.pc = PLAT_ARM_NS_IMAGE_OFFSET,
 
	    SET_STATIC_PARAM_HEAD(image_info, PARAM_EP,
		    VERSION_2, image_info_t, 0),
	    .image_info.image_base = PLAT_ARM_NS_IMAGE_OFFSET,
	    .image_info.image_max_size = ARM_DRAM1_SIZE,
# endif /* PRELOADED_BL33_BASE */
 
	    .next_handoff_image_id = INVALID_IMAGE_ID,
    }
#endif /* EL3_PAYLOAD_BASE */
};
```
在该变量中规定了SCP_BL2， EL3_payload, bl32, bl33 image的相关信息，例如：
+ image的入口地址信息：ep_info
+ image在RAM中的基地址：image_base
+ image的基本信息：image_info
+ image的ID值：image_id

## BL2 to BL3
在bl2中将会加载bl31, bl32, bl33的image到对应权限的RAM中，并将该三个image的描述信息组成一个链表保存起来，以备bl31启动bl32和bl33使。

在AACH64中，bl31为EL3 runtime software，运行时的主要功能是管理smc指令的处理和中断的主力，运行在secure monitor状态中

bl32一般为TEE OS image，本章节以OP-TEE为例进行说明

bl33为非安全image，例如uboot, linux kernel等，当前该部分为bootloader部分的image，再由bootloader来启动linux kernel.

从bl2跳转到bl31是通过带入bl31的`entry point info`调用smc指令触发在bl1中设定的smc异常来通过cpu将全向交给bl31并跳转到bl31中执行。

# BL31
在bl2中通过调用smc指令后会跳转到bl31中进行执行，bl31最终主要的作用是建立EL3 runtime software，在该阶段会建立各种类型的smc调用注册并完成对应的cortex状态切换。该阶段主要执行在monitor中。

## bl31_entrypoint
通过bl31.ld.S文件可知， bl31的入口函数是：`bl31_entrypoint`函数，该函数的内容如下：

```C++
func bl31_entrypoint
#if !RESET_TO_BL31
	/* ---------------------------------------------------------------
	 * Preceding bootloader has populated x0 with a pointer to a
	 * 'bl31_params' structure & x1 with a pointer to platform
	 * specific structure
	 * ---------------------------------------------------------------
	 */
	mov	x20, x0
	mov	x21, x1
 
	/* ---------------------------------------------------------------------
	 * For !RESET_TO_BL31 systems, only the primary CPU ever reaches
	 * bl31_entrypoint() during the cold boot flow, so the cold/warm boot
	 * and primary/secondary CPU logic should not be executed in this case.
	 *
	 * Also, assume that the previous bootloader has already set up the CPU
	 * endianness and has initialised the memory.
	 * ---------------------------------------------------------------------
	 */
/* el3初始化操作，该el3_entrypoint_common函数在上面已经介绍过，其中runtime_exceptions为el3 runtime software的异常向量表，内容定义在bl31/aarch64/runtime_exceptions.S文件中 */
	el3_entrypoint_common					\
		_set_endian=0					\
		_warm_boot_mailbox=0				\
		_secondary_cold_boot=0				\
		_init_memory=0					\
		_init_c_runtime=1				\
		_exception_vectors=runtime_exceptions
 
	/* ---------------------------------------------------------------------
	 * Relay the previous bootloader's arguments to the platform layer
	 * ---------------------------------------------------------------------
	 */
	mov	x0, x20
	mov	x1, x21
#else
	/* ---------------------------------------------------------------------
	 * For RESET_TO_BL31 systems which have a programmable reset address,
	 * bl31_entrypoint() is executed only on the cold boot path so we can
	 * skip the warm boot mailbox mechanism.
	 * ---------------------------------------------------------------------
	 */
	el3_entrypoint_common					\
		_set_endian=1					\
		_warm_boot_mailbox=!PROGRAMMABLE_RESET_ADDRESS	\
		_secondary_cold_boot=!COLD_BOOT_SINGLE_CPU	\
		_init_memory=1					\
		_init_c_runtime=1				\
		_exception_vectors=runtime_exceptions
 
	/* ---------------------------------------------------------------------
	 * For RESET_TO_BL31 systems, BL31 is the first bootloader to run so
	 * there's no argument to relay from a previous bootloader. Zero the
	 * arguments passed to the platform layer to reflect that.
	 * ---------------------------------------------------------------------
	 */
	mov	x0, 0
	mov	x1, 0
#endif /* RESET_TO_BL31 */
 
	/* ---------------------------------------------
	 * Perform platform specific early arch. setup
	 * ---------------------------------------------
	 */
/* 平台架构相关的初始化设置 */
	bl	bl31_early_platform_setup
	bl	bl31_plat_arch_setup
 
	/* ---------------------------------------------
	 * Jump to main function.
	 * ---------------------------------------------
	 */
	bl	bl31_main	//跳转到bl31_main函数，执行该阶段需要的主要操作
 
	/* -------------------------------------------------------------
	 * Clean the .data & .bss sections to main memory. This ensures
	 * that any global data which was initialised by the primary CPU
	 * is visible to secondary CPUs before they enable their data
	 * caches and participate in coherency.
	 * -------------------------------------------------------------
	 */
	adr	x0, __DATA_START__
	adr	x1, __DATA_END__
	sub	x1, x1, x0
	bl	clean_dcache_range
 
	adr	x0, __BSS_START__
	adr	x1, __BSS_END__
	sub	x1, x1, x0
	bl	clean_dcache_range
 
	b	el3_exit	//执行完成将跳转到bl33中执行，即执行bootloader
endfunc bl31_entrypoint
	//执行完成将跳转到bl33中执行，即执行bootloader
endfunc bl31_entrypoint
```

## bl31_main
该函数主要完成必要初始化操作，配置EL3中的各种smc操作，以便在后续顺利响应在CA和TA中产生的smc操作。
```C++
void bl31_main(void)
{
	NOTICE("BL31: %s\n", version_string);
	NOTICE("BL31: %s\n", build_message);
 
	/* Perform platform setup in BL31 */
	bl31_platform_setup();	//初始化相关驱动，时钟等
 
	/* Initialise helper libraries */
	bl31_lib_init();	//用于执行bl31软件中相关全局变量的初始化
 
	/* Initialize the runtime services e.g. psci. */
	INFO("BL31: Initializing runtime services\n");
	runtime_svc_init();	//初始化el3中的service，通过在编译时指定特定的section来确定哪些service会被作为el3 service
 
	/*
	 * All the cold boot actions on the primary cpu are done. We now need to
	 * decide which is the next image (BL32 or BL33) and how to execute it.
	 * If the SPD runtime service is present, it would want to pass control
	 * to BL32 first in S-EL1. In that case, SPD would have registered a
	 * function to intialize bl32 where it takes responsibility of entering
	 * S-EL1 and returning control back to bl31_main. Once this is done we
	 * can prepare entry into BL33 as normal.
	 */
 
	/*
	 * If SPD had registerd an init hook, invoke it.
	 */
/* 如果注册了TEE OS支持，在调用完成run_service_init之后会使用TEE OS的入口函数初始化bl32_init变量，然后执行对应的Init函数，以OP-TEE为例，bl32_init将会被初始化成opteed_init，到此将会执行 opteed_init函数来进入OP-TEE OS的Image，当OP-TEE image OS执行完了image后，将会产生一个TEESMC_OPTEED_RETURN_ENTRY_DONE的smc来通过bl31已经完成了OP-TEE的初始化*/
	if (bl32_init) {
		INFO("BL31: Initializing BL32\n");
		(*bl32_init)();
	}
	/*
	 * We are ready to enter the next EL. Prepare entry into the image
	 * corresponding to the desired security state after the next ERET.
	 */
	bl31_prepare_next_image_entry();		//准备跳转到bl33，在执行runtime_service的时候会存在一个spd service，该在service的init函数中将会去执行bl32的image完成TEE OS初始化
 
	console_flush();
 
	/*
	 * Perform any platform specific runtime setup prior to cold boot exit
	 * from BL31
	 */
	bl31_plat_runtime_setup();
}
```

## runtime_svc_init
该函数主要用来建立smc索引表并执行EL3中提供的service的初始化操作

```C++
void runtime_svc_init(void)
{
	int rc = 0, index, start_idx, end_idx;
 
	/* Assert the number of descriptors detected are less than maximum indices */
/*判定rt_svc_descs段中的是否超出MAX_RT_SVCS条*/
	assert((RT_SVC_DESCS_END >= RT_SVC_DESCS_START) &&
			(RT_SVC_DECS_NUM < MAX_RT_SVCS));
 
	/* If no runtime services are implemented then simply bail out */
	if (RT_SVC_DECS_NUM == 0)
		return;
 
	/* Initialise internal variables to invalid state */
/* 初始化 t_svc_descs_indices数组中的数据成-1，表示当前所有的service无效*/
	memset(rt_svc_descs_indices, -1, sizeof(rt_svc_descs_indices));
 
/* 获取第一条EL3 service在RAM中的起始地址，通过获取RT_SVC_DESCS_START的值来确定，该值在链接文件中有定义 */
	rt_svc_descs = (rt_svc_desc_t *) RT_SVC_DESCS_START;
 
/* 遍历整个rt_svc_des段，将其call type与rt_svc_descs_indices中的index建立对应关系 */
	for (index = 0; index < RT_SVC_DECS_NUM; index++) {
		rt_svc_desc_t *service = &rt_svc_descs[index];
 
		/*
		 * An invalid descriptor is an error condition since it is
		 * difficult to predict the system behaviour in the absence
		 * of this service.
		 */
/* 判定在编译的时候注册的service是否有效 */
		rc = validate_rt_svc_desc(service);
		if (rc) {
			ERROR("Invalid runtime service descriptor %p\n",
				(void *) service);
			panic();
		}
 
		/*
		 * The runtime service may have separate rt_svc_desc_t
		 * for its fast smc and standard smc. Since the service itself
		 * need to be initialized only once, only one of them will have
		 * an initialisation routine defined. Call the initialisation
		 * routine for this runtime service, if it is defined.
		 */
/* 执行当前service的init的操作 */
		if (service->init) {
			rc = service->init();
			if (rc) {
				ERROR("Error initializing runtime service %s\n",
						service->name);
				continue;
			}
		}
 
		/*
		 * Fill the indices corresponding to the start and end
		 * owning entity numbers with the index of the
		 * descriptor which will handle the SMCs for this owning
		 * entity range.
		 */
/* 根据该service的call type以及start oen来确定一个唯一的index,并且将该service中支持的所有的call type生成的唯一表示映射到同一个index中 */
		start_idx = get_unique_oen(rt_svc_descs[index].start_oen,
				service->call_type);
		assert(start_idx < MAX_RT_SVCS);
		end_idx = get_unique_oen(rt_svc_descs[index].end_oen,
				service->call_type);
		assert(end_idx < MAX_RT_SVCS);
		for (; start_idx <= end_idx; start_idx++)
			rt_svc_descs_indices[start_idx] = index;
	}
}
```

## DECLARE_RT_SVC
该宏用来在编译的时候将EL3中的service编译进rt_svc_descs段中，该宏定义如下：

```C++
#define DECLARE_RT_SVC(_name, _start, _end, _type, _setup, _smch) \
	static const rt_svc_desc_t __svc_desc_ ## _name \
		__section("rt_svc_descs") __used = { \
			.start_oen = _start, \
			.end_oen = _end, \
			.call_type = _type, \
			.name = #_name, \
			.init = _setup, \
			.handle = _smch }
```

+ `start_oen`：该service的起始内部number
+ `end.oen`: 该service的末尾number
+ `call_type`: 调用的smc的类型
+ `name`: 该service的名字
+ `init`: 该service在执行之前需要被执行的初始化操作
+ `handle`: 当触发了call type的调用时调用的handle该请求的函数

## BL31 to BL32
在bl31中会执行runtime_service_inti操作，该函数会调用注册到EL3中所有service的init函数，其中有一个service就是为TEE服务，该service的init函数会将TEE OS的初始化函数赋值给bl32_init变量，当所有的service执行完init后，在bl31中会调用bl32_init执行的函数来跳转到TEE OS的执行。

### OPTEE OS实现
实现从bl31到OP-TEE的跳转是通过执行opteed_setup函数来实现的，该函数在执行runtime_svc_int中对各service做service->init()函数来实现，而OPTEE这个service就是通过DECALARE_RT_SVC被注册到tr_svc_descs段中，代码存在service/spd/opteed/opteed_main.c文件中，内容如下：
```C++
DECLARE_RT_SVC(
    opteed_fast,
    
    OEN_TOS_START,
    OEN_TOS_END,
    SMC_TYPE_FAST,
    opteed_setup,
    opteed_smc_handler
);
```

# BL32
在bl31中的runtime_svc_init函数会初始化OP-TEE对应的service，通过调用该service的init函数来完成OP-TEE的启动。

## opteed_setup
OP-TEE的service通过DECLARE_RT_SVC宏在编译的时候被存放到了rt_svc_des段中。启动该段中的init成员会被初始化成opteed_setup函数，该函数的内容如下：
```C++
int32_t opteed_setup(void)
{
	entry_point_info_t *optee_ep_info;
	uint32_t linear_id;
 
	linear_id = plat_my_core_pos();
 
	/*
	 * Get information about the Secure Payload (BL32) image. Its
	 * absence is a critical failure.  TODO: Add support to
	 * conditionally include the SPD service
	 */
/* 获取bl32(OP-TEE)镜像的描述信息 */
	optee_ep_info = bl31_plat_get_next_image_ep_info(SECURE);
	if (!optee_ep_info) {
		WARN("No OPTEE provided by BL2 boot loader, Booting device"
			" without OPTEE initialization. SMC`s destined for OPTEE"
			" will return SMC_UNK\n");
		return 1;
	}
 
	/*
	 * If there's no valid entry point for SP, we return a non-zero value
	 * signalling failure initializing the service. We bail out without
	 * registering any handlers
	 */
	if (!optee_ep_info->pc)
		return 1;
 
	/*
	 * We could inspect the SP image and determine it's execution
	 * state i.e whether AArch32 or AArch64. Assuming it's AArch32
	 * for the time being.
	 */
	opteed_rw = OPTEE_AARCH64;
/* 初始化CPU的smc上下文 */
	opteed_init_optee_ep_state(optee_ep_info,
				opteed_rw,
				optee_ep_info->pc,
				&opteed_sp_context[linear_id]);
 
	/*
	 * All OPTEED initialization done. Now register our init function with
	 * BL31 for deferred invocation
	 */
/* 使用Opteed_init初始化bl32_init变量，以备在bl31中调用 */
	bl31_register_bl32_init(&opteed_init);
 
	return 0;
}
```

## opteed_init
该函数将会在bl31中的所有service执行完init操作之后，通过执行存放在bl32_init变量中的函数指针来实现调用。该函数调用之后会进入到OP-TEE OS的初始化阶段。

```C++
static int32_t opteed_init(void)
{
	uint32_t linear_id = plat_my_core_pos();
	optee_context_t *optee_ctx = &opteed_sp_context[linear_id];
	entry_point_info_t *optee_entry_point;
	uint64_t rc;
 
	/*
	 * Get information about the OPTEE (BL32) image. Its
	 * absence is a critical failure.
	 */
/* 获取OPTEE image的信息 */
	optee_entry_point = bl31_plat_get_next_image_ep_info(SECURE);
	assert(optee_entry_point);
 
/* 使用optee image的entry point信息初始化cpu的上下文 */
	cm_init_my_context(optee_entry_point);
 
	/*
	 * Arrange for an entry into OPTEE. It will be returned via
	 * OPTEE_ENTRY_DONE case
	 */
/* 开始设置CPU参数，最终会调用opteed_enter_sp函数执行跳转到OPTEE的操作 */
	rc = opteed_synchronous_sp_entry(optee_ctx);
	assert(rc != 0);
 
	return rc;
}
```

## opteed_enter_sp
opteed_entr_sp函数完成跳转到OP-TEE image进行执行的操作，该函数中将会保存一些列的寄存器值，设定好堆栈信息，然后通过调用el3_eixt函数来实现跳转操作

```C++
func opteed_enter_sp
	/* Make space for the registers that we're going to save */
	mov	x3, sp
	str	x3, [x0, #0]
	sub	sp, sp, #OPTEED_C_RT_CTX_SIZE
 
	/* Save callee-saved registers on to the stack */
	stp	x19, x20, [sp, #OPTEED_C_RT_CTX_X19]
	stp	x21, x22, [sp, #OPTEED_C_RT_CTX_X21]
	stp	x23, x24, [sp, #OPTEED_C_RT_CTX_X23]
	stp	x25, x26, [sp, #OPTEED_C_RT_CTX_X25]
	stp	x27, x28, [sp, #OPTEED_C_RT_CTX_X27]
	stp	x29, x30, [sp, #OPTEED_C_RT_CTX_X29]
 
	/* ---------------------------------------------
	 * Everything is setup now. el3_exit() will
	 * use the secure context to restore to the
	 * general purpose and EL3 system registers to
	 * ERET into OPTEE.
	 * ---------------------------------------------
	 */
	b	el3_exit	//使用设定好的安全CPU上下文，退出EL3进入OPTEE
endfunc opteed_enter_sp
```

## BL32 to BL 33
当TEE_OS image启动完成之后会触发一个ID为`TEESMC_OPTEED_RETURN_ENTRY_DONE`的smc调用来告知EL3 TEE OS image已经完成了初始化，然后将CPU的状态恢复到bl31_init的位置继续执行。

bl31通过遍历在bl2中记录的image链表来找到需要执行的bl33的image。然后通过获取到bl33 image的镜像信息，设定下一个阶段的CPU上下文，退出el3然后进入到bl33 image的执行。
