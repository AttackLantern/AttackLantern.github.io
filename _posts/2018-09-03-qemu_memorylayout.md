---
title: QEMU memory layout
date: 2018-09-03 20:28
---

## 0， Overview
- QEMU 作为 host machine 的进程而存在，启动后利用 mmap 申请内存块来作为 Guest machine 的物理内存
- main() --> cpu_exec() --> tb_gen_code() --> cpu_gen_code() --> tcg_gen_code() --> tcg_out_op() --> cpu_tb_exec() ----> loop cpu_exec
- main() -> [vl.c](https://github.com/qemu/qemu/blob/55f4e79d794d94b2ab22b0dc99c6b05abc628656/vl.c)
- hardware emulator -> [/hw](https://github.com/qemu/qemu/tree/master/hw)
- tcg -> [/tcg](https://github.com/qemu/qemu/tree/master/tcg)

## 1，Memory Manager
###  1.1 Memory Manage Struct
- Virtual memory mapping -> [exec.c](https://github.com/qemu/qemu/blob/55f4e79d794d94b2ab22b0dc99c6b05abc628656/exec.c)
    - host mmap memory manage
    - record RAMlist / RAMBlock
    - RAMBlock 是从 host machine mmap 出来的内存块（拿给 guest machine 当作物理内存用），用 RAMlist 链表串起来记录它
    - RAMBlock struct -> [/include/exec/ram_addr.h](https://github.com/qemu/qemu/blob/55f4e79d794d94b2ab22b0dc99c6b05abc628656/include/exec/ram_addr.h#L26)
        -  *host 用来记录 host memory addr ( host machine virtual address )
        - offset 记录这块 block 相对于 RAMlist 的 offset，在查找特定 block 的时候会按照这个 offset 去找
        - length 记录这块 block 的长度
        - flags 记录这块内存的某些信息，譬如 RAM_RESIZEABLE （block 的大小可以再更改）、RAM_PREALLOC（block 的大小已经被预先设定成不可更改）、RAM_SHARED（是共享内存）
        - idstr[256] 每块 block 都有一个 id 
    - RAMlist 用来串 RAMBlock，并记录有 dirty page bitmap
    - RAMlist struct -> [/include/exec/ramlist.h](https://github.com/qemu/qemu/blob/55f4e79d794d94b2ab22b0dc99c6b05abc628656/include/exec/ramlist.h#L47)
        - *mru_block 记录最近使用的一块 RAMBlock，如果需要遍历链表取出某一块 block，为了效率，会先检查一下这块 block 是否就是最近使用过的 block，若不是再进行遍历
        - QLIST_HEAD(, RAMBlock) blocks 记录 list 的 header
        - dirty_memory，记录 dirty page，像是显示界面时 qemu 会通过这个值来追踪哪一块是 dirty page，然后判断是否需要重绘界面
- Physical memory manage -> [memory.c](https://github.com/qemu/qemu/blob/master/memory.c)
    - guest physical memory manage
    - API -> /include/exec/memory.h
    - RAMBlock 拿到的是待处理的内存 chunks，来源可能是 hvm 或是 /dev，qemu 需要进一步地把这些内存处理成 memory 和 IO ，才能丢给 guest machine 使用 。qemu 使用 AddressSpace 和 MemoryRegion 这两个 struct 进行这一处理的步骤
    - MR 的区域可能存在重叠，譬如在同一段地址空间内，可能存在多个 MR 相互交错
    - AddressSpace 是一棵由 MemoryRegion 构成的树，是 guest machine 的 CPU 所看到的内存，MemoryRegion 是 guest 所直接操纵的，AddreSpace 用一种叫做 FlatView 的树状结构记录下可用的 MemoryRegion 
    - AddressSpace struct ->  [/include/exec/memory.h #425](https://github.com/qemu/qemu/blob/595123df1d54ed8fbab9e1a73d5a58c5bb71058f/include/exec/memory.h#L425)
        - MemoryRegion *root，记录 MemoryRegion Root，这个 MemoryRegion 向下展开，从而形成一棵树，有两个特定的 *root
               - system_memory，记录 guest machine memory address mapping
               -  system_io，记录 guest machine I/O address mapping
        - struct FlatView *current_map，记录构成该 AddressSpace 所有 MR
   - MemoryRegion struct -> [/include/exec/memory.h #339](https://github.com/qemu/qemu/blob/595123df1d54ed8fbab9e1a73d5a58c5bb71058f/include/exec/memory.h#L339)

```c
struct MemoryRegion {
    Object parent_obj; 
    bool romd_mode;
    bool ram; //表明该 MR 是否当作 RAM 使用
    bool subpage;
    bool readonly; /* For RAM regions */   //是否可写
    bool rom_device;
    bool flush_coalesced_mmio;
    bool global_locking;
    uint8_t dirty_log_mask;
    bool is_iommu;
    RAMBlock *ram_block; //这块 MemoryRegion 所指向的 RAMBlock
    Object *owner;

    const MemoryRegionOps *ops; //这块 MR 所能够进行的操作，譬如 read / write 的 callback
    void *opaque;
    MemoryRegion *container; //多个 MR 集合
    Int128 size;
    hwaddr addr;
    void (*destructor)(MemoryRegion *mr);
    uint64_t align;
    bool terminates;
    bool ram_device;
    bool enabled;
    bool warning_printed; /* For reservations */
    uint8_t vga_logging_count;
    MemoryRegion *alias;
    hwaddr alias_offset;
    int32_t priority;
    QTAILQ_HEAD(subregions, MemoryRegion) subregions;
    QTAILQ_ENTRY(MemoryRegion) subregions_link;
    QTAILQ_HEAD(coalesced_ranges, CoalescedMemoryRange) coalesced;
    const char *name;
    unsigned ioeventfd_nb;
    MemoryRegionIoeventfd *ioeventfds;
};
```
- Memory-backend manage -> [/hw/mem/pc-dimm.c](https://github.com/qemu/qemu/blob/master/hw/mem/pc-dimm.c)
    - host machine 认为他所拿到的内存，实际上是由 qemu 以 MemoryRegion 拿给他的假的内存（……）  
    - 对于 host machine 来说是一个热插拔的内存设备
- 整体大概是从 hva 拿到内存块（RAMBlock），用 RAMlist 把这些块串起来。然后 AddressSpace 把这些 blocks 分类为 guest 能看到的 Memory/IO，最后交给 backend。
 
```  
              host machine
            (malloc from host)
                RAMlist                       Block A,B,C......                                                                                                           
                   |              +---------------------------------------------+                                                                                                                     
                   |              |                                             |                               
                   |              |                                             |                              
  +++—————————| ---A--- |   RAMBlock A         ---+                             |                               
  |    ++ ————| ---B--- |   RAMBlock B         <--+ [A + offset]                |
  |     |   +—| ---C--- |   RAMBlock C         <--+ [B + offset]                |
  |     |   |                                                                   |
  |     |   |                                                                   |
  |     |   \---*host pointer C ---> [Host Virual Addr / Guest Physical Addr]   |
  |     |                                                                       |
  |      \---*host pointer B ---> [ HVA / GPA ]                                 |
  |                                                                             |
  \---*host pointer---> [ HVA / GPA ]                                           |
                                                                                |
                                                                                |
       guest machine                                                            |
+----system_memory |---A---|---B---|      <-------------------------------------+
 |   +--system_io  |---C---|
 |   |
 |   |                                            AddressSpace(address_space_io)
 |   |                                                           |
 |   +-------[ram_block ptr C]----MemoryRegion <---[*root ptr]---+
 |   
 |                                    AddressSpace(address_space_memory)
 |                                                     |
 +-------- [ram_block ptr A]----MemoryRegion <---[*root ptr]  
 |                 | (link)
 +---------[ram_block ptr B] <---[MemoryRegion A container]
```

### 1.2 Execute
#### 1.2.1 Alloc RAMBlock function
- qemu 和 kvm 协作进行内存管理（为了增加效率），qemu 所需要做的工作是从宿主机申请内存拿给 guest 用，也就是 guest machine physical address <-> host  virtual address，然后把这个映射关系丢给 kvm，而 vm 会用影子页表记下 guest machine virtual address <-> host machine physical address 
- 分配 Memory 所用到的函数都在 exec.c 里，单位是 RAMBlock：
    - `qemu_ram_alloc_from_file()` 根据创建 qemu 进程时指定的 -mem-path 来拿内存块，大概像是 `/dev/` 之类（？）
    - `qemu_ram_alloc_internal()` 是一个 callback，下面的三个函数都会以不同的参数来 call 这个 function
    - `qemu_ram_alloc_resizeable()` 指定参数 void(*resized)，根据它重设 RAMBlock 的大小
    - `qemu_ram_alloc_from_ptr()`  根据 *host 指向的 address 申请指定 size 的内存块，但这个 size 不能后期更改，该 RAMBlock 的 flag 位设成 RAM_PREALLOC
    - `qemu_ram_alloc()` 比较随便地申请一块 block，既不指定 *host 也不指定起始 ptr 

#### 1.2.2 qemu 内存分配流程
- 1，启动 qemu 时，首先是在 vl.c 的 main() 里处理用户所 input 的参数，然后开始初始化 cpu（用来动态翻译 guest code）。
- 2，初始化 cpu 的工作通过调用 `cpu_exec_init_all()`(exec.c) 进行，此时会注册 memory mapping I/O的 callback。
- 3，转入 `memory_map_init()`(exec.c)，初始化以下四个全局变量：
            - `static MemoryRegion system_memory` 
            - `AddressSpace address_space_memory` Root 存放的是 `system_memory`，name 为 "memory"
            - `static MemoryRegion system_io`
            - `AddressSpace address_space_io` root 存放 `system_io`
            - `memory_map_init()` function:

```c
static void memory_map_init(void)
{
    system_memory = g_malloc(sizeof(*system_memory));
    memory_region_init(system_memory, NULL, "system", UINT64_MAX);
    address_space_init(&address_space_memory, system_memory, "memory");
 
    system_io = g_malloc(sizeof(*system_io));
    memory_region_init_io(system_io, NULL, &unassigned_io_ops, NULL, "io",
                          65536);
    address_space_init(&address_space_io, system_io, "I/O");
}
```
- 4，进行设备初始化，`(qemu_opts_foreach(qemu_find_opts("drive"), drive_init_func, &machine_class->block_default_type, NULL)) `
- 5，然后跳进 `machine_class->init(current_machine)`，这时候会根据对应的 guest machine type 去初始化 guest 的相关信息
- 6，进入 `pc_init1()`(/hw/i386/pc_piix.c) 进行 guest 实际 memory 的分配
    - 判断 guest 内存是否大于 4G 如果大于，则拆分成两个 MR 再操作
    - 调用 `pc_memory_init()` [/hw/i386/pc.c #1324](https://github.com/qemu/qemu/blob/3c825bb7c1b4289ef05f51b5b77ac0967b6a27fa/hw/i386/pc.c#L1324)，以 MR 为单位进行 guest memory 分配（guest machine 将会拿到一整块 MR），并 load bios
    
```
FWCfgState *pc_memory_init(MachineState *machine,
                           MemoryRegion *system_memory,
                           ram_addr_t below_4g_mem_size,
                           ram_addr_t above_4g_mem_size,
                           MemoryRegion *rom_memory,
                           MemoryRegion **ram_memory,
                           PcGuestInfo *guest_info)
{
    int linux_boot, i;
    MemoryRegion *ram, *option_rom_mr;
    MemoryRegion *ram_below_4g, *ram_above_4g;
    FWCfgState *fw_cfg;
    PCMachineState *pcms = PC_MACHINE(machine);

    assert(machine->ram_size == below_4g_mem_size + above_4g_mem_size);

    linux_boot = (machine->kernel_filename != NULL);

    /* Allocate RAM.  We allocate it as a single memory region and use
     * aliases to address portions of it, mostly for backwards compatibility
     * with older qemus that used qemu_ram_alloc().
     */
    ram = g_malloc(sizeof(*ram));
    memory_region_allocate_system_memory(ram, NULL, "pc.ram",
                                         machine->ram_size);
    *ram_memory = ram;
    ram_below_4g = g_malloc(sizeof(*ram_below_4g));
    memory_region_init_alias(ram_below_4g, NULL, "ram-below-4g", ram,
                             0, below_4g_mem_size);
    memory_region_add_subregion(system_memory, 0, ram_below_4g);
    e820_add_entry(0, below_4g_mem_size, E820_RAM);
    if (above_4g_mem_size > 0) {
        ram_above_4g = g_malloc(sizeof(*ram_above_4g));
        memory_region_init_alias(ram_above_4g, NULL, "ram-above-4g", ram,
                                 below_4g_mem_size, above_4g_mem_size);
        memory_region_add_subregion(system_memory, 0x100000000ULL,
                                    ram_above_4g);
        e820_add_entry(0x100000000ULL, above_4g_mem_size, E820_RAM);
    }
```

  - 在拿给 guest 之前，先对这块 MR 进行初始化，进入 `memory_region_init_ram()` [memory.c]，它会调用 `qemu_ram_alloc` 进行实际的内存申请（最新版的 qemu 3.0 需要先进入函数 `memory_region_init_ram_nomigrate()`，表明这块 memory 不是实时迁移来的，2.x 直接调用，不过不管是哪一版本，最终都会用`qemu_ram_alloc()` 进行实际内存的分配）。


## 2，Process
- KVM 模拟出来的 CPU 称之为 vCPU，每个 vCPU 单独一个进程
- availabe devices: qemu -device \?

## 3，qemu debug

```
#!/bin/bash

sudo gdb -n -q --args /home/vm/qemu/bin/debug/native/x86_64-softmmu/qemu-system-x86_64 -enable-kvm -m 512 /home/vm/test/tinycore.img \
-netdev user,id=t0, -device rtl8139,netdev=t0,id=nic0 \
-netdev user,id=t1, -device pcnet,netdev=t1,id=nic1 
```
- gdb script: `/script/qemy-gdb.py` ，gdb 默认用的 python 3，但 qemy-gdb.py 里有用到 long 类型来输出，需要把 long 替换成 int。 
    - 然而只有两个命令可用，基本没用
-  `machine` 记录的是 qemu machine type （可以用  `qemu-system-x86_64 -machine help` 查看）
