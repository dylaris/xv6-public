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
当.text/.data等段加载到对应的地址后，进入到entry中，经过调试发现是在`main.c`中 (b entry)

## main.c
