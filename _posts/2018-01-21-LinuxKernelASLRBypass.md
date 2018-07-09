---
title: CVE-2016-3672 Linux ASLR Disable 漏洞分析
date: 2018-01-21 21:41:49
---
 
CVE-2016-3672，这个漏洞的利用难度非常低，危害性不大，但是思路蛮有趣的，也许对一些 CTF 比赛会有一些帮助。

## 0x00 特殊情况下的 ASLR 
利用漏洞需要几个条件：
- 1，Linux kernel < 4.5.2
- 2，攻击者能够远程登录主机
- 3，漏洞程序为 x86 且存在 Overflow

在使用 mmap 将数据映射至内存中时，系统会以 mmap 函数的第一个参数作为依据进行 mapping，若 addr 为 NULL，则由系统自行随机分配地址完成，mmap() 原型为：
```
 void *mmap(void *addr, size_t length, int prot, int flags,
                  int fd, off_t offset);
```

编译
```c
#include <stdio.h>
#include <sys/mman.h>
#include <sys/types.h>
#include <fcntl.h>

int main(){
       int fd, i;
       fd = open("/etc/shadow", O_RDONLY);
       
       void *mapping = mmap(NULL, 4096, PROT_READ, MAP_PRIVATE, 
                                fd, 0);
    printf("%p", mapping);
    return 0;
}
```

当 mmap 失败时，会返回地址 0xffffffff。将栈的大小由默认值 8192 改为 unlimited （`ulimit -s unlimited`）后，mmap 的地址将固定，ASLR 将失效。

ASLR 的影响主要涉及 stack、heap、libc、VDSO 等等，当开启正常级别 ASLR（即 kernel.randomize_va_space = 1）时，stack 和 libc 开始随机化。
在某些情况下，即便开启了 ASLR，系统也不会对程序的 libc 进行随机。这个漏洞就是通过将 stack 资源设置为无限大，从而触发这个特殊情况，使得 libc 位置固定下来。

## 0x01 虚拟内存布局
系统使用 vm_area_struct 进行虚拟内存管理分配线性区，其源码位于mm_types.h ，这份源码中有一结构体 mm_struct ，其中的内容便包括了mmap 进行内存映射的信息。
```c
struct mm_struct {
       struct vm_area_struct *mmap;		/* list of VMAs */
       struct rb_root mm_rb;
       u32 vmacache_seqnum;                   /* per-thread vmacache */
#ifdef CONFIG_MMU
        unsigned long (*get_unmapped_area) (struct file *filp,
                             unsigned long addr, unsigned long len,
                             unsigned long pgoff, unsigned long flags);
#endif
       unsigned long mmap_base;		/* base of mmap area */
        unsigned long mmap_legacy_base;         /* base of mmap area
        in bottom-up allocations */
#ifdef CONFIG_HAVE_ARCH_COMPAT_MMAP_BASES
       /* Base adresses for compatible mmap() */
       unsigned long mmap_compat_base;
       unsigned long mmap_compat_legacy_base;
#endif
       unsigned long task_size;		/* size of task vm space */
       unsigned long highest_vm_end;		/* highest vma end address */
......
```

在这当中有两个重要地址，即 mmap_base 和 mmap_legacy_base。mmap_base 即为所分配的 mmap 基地址；后者的存在意义是，支持旧的 x86 程序的 unmapped_area 布局，当采用该种布局时，系统会将  mmap_legacy_base 的值赋给 mmap_base，从而完成 mmap 区域的划分。漏洞就出在 mmap_legacy_base 的问题上。
当调用 mmap 时，系统需要在内存空间中寻找一块足够大的区域来分配给它，有两种方法进行分配。
1，unmapped_area
x86 原始的分配方法，mmap 向栈方向生长。
```
______________
|   STACK   |       down
|　         |
|           |            
|    mmap   |        up
|           |
|           |
|    heap   |        up
_____________
```

2，unmapped_area_topdown
mmap 加上一个随机值进行偏移。此时对栈使用一个固定值，它的方向向堆方向生长。这种方法需要确切的 STACK 大小，也就是为 STACK 预先分好一个固定的大小，这样才能在其下方开一块 mmap 区域，使得 heap 和 mmap 中间的空当最大化，尽可能的利用空间。这个方法受制于栈的大小。
```
____________________
|   STACK          |       down
|　                |
|                  |            
|    mmap+random   |        down
|                  |
|                  |
|    heap          |        up
_____________________
```

/mm/mmap.c 中对使用何种布局使用函数 arch_pick_mmap_layout() 进行判断：
```c
void arch_pick_mmap_layout(struct mm_struct *mm)
{
       unsigned long random_factor = 0UL;

       if (current->flags & PF_RANDOMIZE)
                  random_factor = arch_mmap_rnd();

        mm->mmap_legacy_base = mmap_legacy_base(random_factor);

         if (mmap_is_legacy()) {
              mm->mmap_base = mm->mmap_legacy_base;
              mm->get_unmapped_area = arch_get_unmapped_area;
          } else {
              mm->mmap_base = mmap_base(random_factor);
              mm->get_unmapped_area = arch_get_unmapped_area_topdown;
          }
}

```
当 mmap_is_legacy() 为真值时，系统将会采用第一种布局进行分配，即 `mm->get_unmapped_area = arch_get_unmapped_area;`，此时 mmap 的基地址由 mm->mmap_legacy_base 确定。
进一步地，mmap_legacy_base 的地址又是根据 mmap_is_ia32() 的返回值是否为真而得来：
[/v4.5.1/x86/mmap.c](http://elixir.free-electrons.com/linux/v4.5.1/source/arch/x86/mm/mmap.c)

```c
static unsigned long mmap_legacy_base(unsigned long rnd)
{
       if (mmap_is_ia32())
               return TASK_UNMAPPED_BASE;
        else
               return TASK_UNMAPPED_BASE + rnd;
}
```
这就存在一个漏洞利用点，如果能进入 return TASK_UNMAPPED_BASE; 这一分支，则 mmap 区域的基地址不会进行偏移，只要使 mmap_is_ia32() 返回真。
到这里，可以看出来，只需要满足：1，程序是 32 位；2，mmap_is_legacy() 返回真。
/mm/mmap.c :
```c
static int mmap_is_legacy(void)
{
       if (current->personality & ADDR_COMPAT_LAYOUT)
              return 1;

        if (rlimit(RLIMIT_STACK) == RLIM_INFINITY)
               return 1;

         return sysctl_legacy_va_layout;
}
```
 
当 stack 大小为 RLIM_INFINITY 时，即可满足。而在 resource.h 中，它的默认值是 0x7fffffff，即 unlimited。
```
#define RLIM_INFINITY		0x7fffffff
``` 
综上，只要设置 `ulimit -s unlimited` 就可以了。

## 0x02 总结
这个漏洞非常好理解：
1，在某些特殊情况下可以使得 libc 不进行随机化
特殊情况：程序为 32 位，mmap 基地址不会偏移，且系统采用 unmapped_area 布局布置 mmap 空间
2，如何触发该情况
将 stack 设置为无限大，导致系统无法以固定大小限定 stack 下界，不能通过 unmapped_area_topdown 布局来分配 mmap。 

这个漏洞也许在一些 CTF 中能用……吧……
