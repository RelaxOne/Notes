​		本项目旨在学习操作系统内核中关于内存的相关知识，当前操作系统内核源码分析中大部分是基于 Linux 的，但由于作者原来不是做操作系统相关内容的，因此直接上手 Linux 内核源码比较吃力，故而选择一个 开源的微内核操作系统 FreeNOS，其源码下载地址：http://freenos.org/pages/download.html

​		在源码学习中，基于从入门学习计算机编程语言开始就知道要首先知道程序的入口，基于此，对此内核源码分析，首先也找到的是对内核源码中关于程序入口，经过作者对源码的摸索发现，其入口为 /kernel/intel.pc/Main.cpp 或者 /kernel/arm/rasoberry/Main.cpp 中 kernel_main() 方法，而调用 kernel_main()是在/lib/libarch/intel/IntelBoot32.S 或者/kernel/arm/ARMBoots.S文件中，其具体实现由于采用的是汇编代码，作者不懂汇编，因此这部分不清楚。

~~~C++
extern C int kernel_main(CoreInfo *info)
{
    // 初始化内核堆信息，任何对象初始化都必须在内核堆初始化完成之后进行
    coreInfo.heapAddress = MegaByte(3);	// 设置内核堆起始地址
    coreInfo.heapSize    = MegaByte(1);	// 设置内核堆大小
    Kernel::heap(coreInfo.heapAddress, coreInfo.heapSize);

    // 启动内核调试串行控制台
    if (info->coreId == 0){
        IntelSerial *serial = new IntelSerial(0x3f8);
        serial->setMinimumLogLevel(Log::Notice);
    }

    // Run all constructors first
    constructors();

    // Create and run the kernel
    IntelKernel *kernel = new IntelKernel(info);
    return kernel->run();
}
~~~

​		在 kernel_main() 方法中，首先设置内核堆内存的起始地址为 3MB，内核堆内存大小设置为 1MB，然后调用 Kernel::heap() 静态方法初始化内核堆信息，因为任何对象的初始化都必须在内核堆初始化完成之后；在初始化内核堆信息中，设置默认的分配方法为 PoolAllocator，其父分配器为 BubbleAllocator;关于这两种分配器，详细请参照分配器章节内容。

~~~c++
Error Kernel::heap(Address base, Size size)
{
    Allocator *bubble, *pool;
    Size meta = sizeof(BubbleAllocator) + sizeof(PoolAllocator);

    // 设置从 base  开始的 size 个字符为 0
    MemoryBlock::set((void *) base, 0, size);

    // 设置动态堆内存
    bubble = new (base) BubbleAllocator(base + meta, size - meta);
    pool   = new (base + sizeof(BubbleAllocator)) PoolAllocator();
    pool->setParent(bubble);

    // 设置默认分配器
    Allocator::setDefault(pool);
    return 0;
}
~~~

​		在 kernel_main() 方法中，然后会启动内核调试串行控制台（这部分内容未详细探查，但目测是关于调试内核相关配置，后面有时间再深入了解），以及之后的运行所有的构造方法（此部分内容不懂）。接下来就新建一个 Intelkernel 对象，调用其 run() 方法。在新建 Intelkernel 对象时，会初始化一些内核对象；在此过程中会初始化 Kernel 对象，在初始化 Kernel 对象时，会保证其实现的是单例模式，即内存中仅存在一个该对象的实例。

~~~c++
Kernel::Kernel(CoreInfo *info) : Singleton<Kernel>(this), m_interrupts(256)
{
    // Output log banners
    if (Log::instance){
        Log::instance->append(BANNER);
        Log::instance->append(COPYRIGHT "\r\n");
    }

    // 计算内存地址（low & high）
    Memory::Range highMem;
    Arch::MemoryMap map;
    MemoryBlock::set(&highMem, 0, sizeof(highMem));
    highMem.phys = info->memory.phys + map.range(MemoryMap::KernelData).size;

    // Initialize members
    m_alloc  = new SplitAllocator(info->memory, highMem);
    m_procs  = new ProcessManager(new Scheduler());
    m_api    = new API();
    m_coreInfo   = info;
    m_intControl = ZERO;
    m_timer      = ZERO;

    // 标记内核地址为已使用 (物理地址前 4 M)
    for (Size i = 0; i < info->kernel.size; i += PAGESIZE)
        m_alloc->allocate(info->kernel.phys + i);

    // 标记 BootImage 内存为已使用
    for (Size i = 0; i < m_coreInfo->bootImageSize; i += PAGESIZE)
        m_alloc->allocate(m_coreInfo->bootImageAddress + i);

    // 标记 堆内存为已使用
    for (Size i = 0; i < m_coreInfo->heapSize; i += PAGESIZE)
        m_alloc->allocate(m_coreInfo->heapAddress + i);

    // 保留 CoreChannel 内存
    for (Size i = 0; i < m_coreInfo->coreChannelSize; i += PAGESIZE)
        m_alloc->allocate(m_coreInfo->coreChannelAddress + i);

    // 清空中断表
    m_interrupts.fill(ZERO);
}
~~~

~~~c++
IntelKernel::IntelKernel(CoreInfo *info) : Kernel(info) {
    IntelMap map;
    IntelCore core;
    IntelPaging memContext(&map, core.readCR3(), m_alloc);

    // Refresh MemoryContext::current()
    memContext.activate();

    // Install interruptRun() callback
    interruptRun = ::executeInterrupt;

    // Setup exception handlers
    for (int i = 0; i < 17; i++){
        hookIntVector(i, exception, 0);
    }
    // Setup IRQ handlers
    for (int i = 17; i < 256; i++){
        // Trap gate
        if (i == 0x90)
            hookIntVector(0x90, trap, 0);
        // Hardware Interrupt
        else
            hookIntVector(i, interrupt, 0);
    }

    // Only core0 uses PIC and PIT.
    if (info->coreId == 0){
        // Set PIT interrupt frequency to 250 hertz
        m_pit.setFrequency(250);
        // Configure the master and slave PICs
        m_pic.initialize();
        m_intControl = &m_pic;
    }
    else
        m_intControl = 0;

    // Try to configure the APIC.
    if (m_apic.initialize() == Timer::Success) {
        NOTICE("Using APIC timer");

        // Enable APIC timer interrupt
        hookIntVector(m_apic.getInterrupt(), clocktick, 0);
        m_timer = &m_apic;
        if (m_coreInfo->timerCounter == 0) {
            m_apic.start(&m_pit);
            m_coreInfo->timerCounter = m_apic.getCounter();
        }
        else
            m_apic.start(m_coreInfo->timerCounter, m_pit.getFrequency());
    }
    // Use PIT as system timer.
    else {
        NOTICE("Using PIT timer");
        m_timer = &m_pit;

        // Install PIT interrupt vector handler
        hookIntVector(m_intControl->getBase() + m_pit.getInterrupt(), clocktick, 0);

        // Enable PIT interrupt
        enableIRQ(m_pit.getInterrupt(), true);
    }

    // Initialize TSS Segment
    Address tssAddr = (Address) &kernelTss;
    gdt[KERNEL_TSS].limitLow    = sizeof(TSS) + (0xfff / 8);
    gdt[KERNEL_TSS].baseLow     = (tssAddr) & 0xffff;
    gdt[KERNEL_TSS].baseMid     = (tssAddr >> 16) & 0xff;
    gdt[KERNEL_TSS].type        = 9;
    gdt[KERNEL_TSS].privilege   = 0;
    gdt[KERNEL_TSS].present     = 1;
    gdt[KERNEL_TSS].limitHigh   = 0;
    gdt[KERNEL_TSS].granularity = 8;
    gdt[KERNEL_TSS].baseHigh    = (tssAddr >> 24) & 0xff;

    // Fill the Task State Segment (TSS).
    MemoryBlock::set(&kernelTss, 0, sizeof(TSS));
    kernelTss.ss0    = KERNEL_DS_SEL;
    kernelTss.esp0   = 0;
    kernelTss.bitmap = sizeof(TSS);
    ltr(KERNEL_TSS_SEL);
}
~~~

​		在初始化为对象后，调用 kernel->run()  方法，在此方法中主要实现了加载 bootImage 和设置任务调度器的定时器。内核进程会一直轮询的选取进程运行。

~~~c++
int Kernel::run() {
    NOTICE("");
    
    loadBootImage();	// 加载 BootImage
    
    m_procs->getScheduler()->setTimer(m_timer);	// 给任务调度器设置定时器
    m_procs->schedule();
    // Never actually returns.
    return 0;
}
~~~

