## 启动流程
```txt
BIOS
  │
  ▼
bootasm.S（汇编）
  │
  │ ① 开启 A20 地址线
  │ ② 加载 GDT
  │ ③ 切换到 32 位保护模式（Protected Mode）
  │ ④ 跳转到 bootmain()
  │
  ▼
bootmain.c（C 语言） ===> bootmain()
  │
  │ ⑤ 从硬盘读取 ELF 格式的内核
  │ ⑥ 跳转到内核入口（entry.S）物理地址
  │
  ▼
entry.S（汇编，内核入口）===> _start
  │
  │ ① 开启 4MB 大页 (PSE)
  │ ② 加载页目录 entrypgdir
  │ ③ 开启分页 (Paging)
  │ ④ 设置栈指针
  │ ⑤ 跳转到 main() (虚拟地址 0x80100000)
  │
  ▼
main.c（内核主程序）===> main()
  │
  │ ⑩ 初始化内存分配器、进程、文件系统...
```

## kernel.ld
在链接脚本中，规划了kernel的内存分布：
- 虚拟地址分配：
```txt
0x80100000  ──┬── .text 开始
              │
0x8010C800    ├── .text 结束 / .rodata 开始
              │
0x8010F000    ├── .rodata 结束 / 页对齐
              │  (ALIGN 0x1000)
0x80110000    ├── .data 开始
              │
0x80111400    ├── .data 结束 / .bss 开始
              │
0x80116400    ├── .bss 结束 (end)
              │
0x80116400  ──┴── 内核结束
```
- 物理地址分配
```txt
0x00100000  ──┬── .text 开始
              │
0x0010C800    ├── .text 结束
              │
0x0010F000    ├── .rodata 开始
              │
0x00112000    ├── .rodata 结束
              │
0x00113000    ├── .data 开始
              │
0x00114400    ├── .data 结束
              │
0x00114400  ──┴── 注意：.bss 在文件中不占空间（物理地址没有）
```
kernel image的VA和PA是一个线性映射的关系：`VA=PA+0x80000000`

## bootasm.S

## bootmain.c
在bootmain()中，下面这段代码
```c
  elf = (struct elfhdr*)0x10000;  // scratch space

  // Read 1st page off disk
  readseg((uchar*)elf, 4096, 0);
```
kernel image存放于第一个扇区，这里是从第一个扇区看是读取4096字节到`0x10000`处，这里只是一个临时存储区，用来存放elf头
```c
  // Load each program segment (ignores ph flags).
  ph = (struct proghdr*)((uchar*)elf + elf->phoff);
  eph = ph + elf->phnum;
  for(; ph < eph; ph++){
    pa = (uchar*)ph->paddr;
    readseg(pa, ph->filesz, ph->off);
    if(ph->memsz > ph->filesz)
      stosb(pa + ph->filesz, 0, ph->memsz - ph->filesz);
  }
```
读取了elf头之后，就知道了program header table，然后将他们加载到对应的物理地址处，然后还需要注意，如果有bss段，还需要清零，并分配空间（磁盘上不存储bss段）
```c
  // Call the entry point from the ELF header.
  // Does not return!
  entry = (void(*)(void))(elf->entry);
  entry();
```
当.text/.data等段加载到对应的地址后，进入到entry中，经过调试发现是在`entry.S:entry`

## entry.S

## main.c
内存管理采用空闲链表的方式(头插法)
```c
struct run {
  struct run *next;
};

struct {
  struct spinlock lock;
  int use_lock;
  struct run *freelist;
} kmem;
```
`kinit1()`初始化低4MB的内存（不包含内核代码，将空闲页加入到空闲链表中），此时无需加锁，只有一个cpu启动
`kinit2()`初始化剩余内存（将空闲页加入到空闲链表中），此时需要加锁，多个cpu已经启动
`kvmalloc()`初始化内核页表，按照`kmap[]`蓝图, `walkpgdir()`是找到虚拟地址对应的页表项，即PTE的地址, `mpapages`将一段物理地址线性映射到一段虚拟地址处（直接映射就是偏移量为0的线性映射）
`seginit()`只初始化`kcode/kdata/ucode/udata`四个段，并且采用平坦模型，所有段的大小都是4GB，只有特权级有区别

## vm.c
虚拟地址 -> 物理地址
虚拟地址高10位是页目录索引，中10位是页表索引，低12位是页内偏移
CR3存储的是页目录表的物理地址
CR3+PDX=>页表物理地址
页表物理地址+PTX=>页物理地址
页物理地址+OFF=>具体物理地址
```txt

                        虚拟地址 0x80100000
                               │
                   ┌───────────┼───────────┐
                   ▼           ▼           ▼
              PDX=0x200   PTX=0x000   OFF=0x000
                   │           │           │
                   ▼           │           │
CR3=0x00000000 → 页目录        │           │
                  PDE[512] ────┘           │
                   │                       │
                   │ 页表地址=0x00000000    │
                   ▼                       │
                 页表                      │
                 PTE[0] ───────────────────┘
                   │
                   │ 物理页地址=0x00000000
                   ▼
              物理地址 = 0x00000000 + 0x000 = 0x00000000
              (PTE_PS 大页模式时实际为 0x00100000)
```

```txt
虚拟地址空间                           物理地址空间
─────────────────────────────────────────────────────────────────
0xFFFFFFFF                             0xFFFFFFFF
    │                                       │
    │  设备内存 (DEVSPACE)                   │  设备内存
    │  0xFE000000 → 0xFE000000 (直接映射)    │
0xFE000000 ──────────────────────────────────── 0xFE000000
    │                                       │
    │  未使用/空洞                            │
    │                                       │
0x80000000 + PHYSTOP ──────────────────────  PHYSTOP (物理内存上限)
    │                                       │
    │  内核数据段 + 自由物理内存                │
    │  data → PHYSTOP                       │
0x80000000 + data ────────────────────────  V2P(data)
    │                                       │
    │  内核代码段 + 只读数据                   │
    │  KERNLINK → data                      │
0x80000000 + KERNLINK ────────────────────  V2P(KERNLINK)
    │                                       │
    │  I/O 空间 (低 1MB)                     │
    │  KERNBASE → KERNBASE+EXTMEM           │
0x80000000 (KERNBASE) ────────────────────  0 (物理地址0)
    │                                       │
    │  用户内存                              │
    │  0 → KERNBASE                         │
0x00000000 ─────────────────────────────────── 自由物理内存
```
