---
title: re做题记录：[萌新赛]flag白给（ctfshow）
date: 2026-09-15
tags: [CTF,reverse]
categories: 安全
---



# re做题记录：[萌新赛]flag白给（ctfshow）



**题目链接：https://ctf.show/challenges#flag%E7%99%BD%E7%BB%99-127**



平时没有怎么接触过reverse，看见一大段的加密伪代码就让人十分头痛，但是这道题目十分甚至九分的简单，题如其名，没有什么复杂的加密算法，主要还是用来熟悉一下ida的利用和upx脱壳



先把附件下载下来运行一下，题目还做了个窗口，让你输入序列号



<img src="/images/re：flag白给/QQ截图20260915143202.png" alt="QQ截图20260915143202" style="zoom: 80%;" />



题目也已经提示了序列号就是flag，用ida分析一下，发现里面只有一个函数，旁边有upx的提示，看样子是加壳了



<img src="/images/re：flag白给/QQ截图20260915143608.png" alt="QQ截图20260915143608" style="zoom:50%;" />



用exeinfope查一下，果然是加壳了



<img src="/images/re：flag白给/QQ截图20260915143724.png" alt="QQ截图20260915143724" style="zoom: 67%;" />



在kali里面用upx工具可以直接脱壳，进到kali里面脱一下壳，再把脱壳后的文件复制出来



```
upx -d flag.exe
```



![QQ截图20260915144141](/images/re：flag白给/QQ截图20260915144141.png)



拖进ida之后就可以看见全部函数了



<img src="/images/re：flag白给/QQ截图20260915144430.png" alt="QQ截图20260915144430" style="zoom:50%;" />



进入入口start函数，巴拉巴拉的看不懂，不知道是干嘛的，没办法了，从字符串入手吧

字符串里看上去也没什么特别值得注意的，只有一个字符表和一个HackAv的字符串值得看一下

**点击上方view->open subviews->strings查看字符串**



<img src="/images/re：flag白给/QQ截图20260915144841.png" alt="QQ截图20260915144841" style="zoom:50%;" />



双击这个HackAv字符串看看它在内存中的位置，可以发现下面有判定成功和失败的相关字符串



![QQ截图20260915145159](/images/re：flag白给/QQ截图20260915145159.png)



要想看使用判定成功的字符串的相关逻辑，**把鼠标光标移动到字符串地址的位置，按一下x键，ida会给出在哪里调用了这个字符串**



![QQ截图20260915145425](/images/re：flag白给/QQ截图20260915145425.png)



双击进入，f5反汇编，从伪代码可以看出来，调用成功字符串的判断逻辑是if ( Sysutils::CompareStr(v5, &str_HackAv[1]) )，刚才在看字符串的时候可以发现这里的str_HackAv就是HackAv字符串，这里的Sysutils::CompareStr函数应该就是一个字符串比较，如果忘了str_HackAv是什么，双击它就可以看见



<img src="/images/re：flag白给/QQ截图20260915145750.png" alt="QQ截图20260915145750" style="zoom:50%;" />



那逻辑就很清楚了，序列号就是HackAv，用程序验证一下也能过，flag就是flag{HackAv}



<img src="/images/re：flag白给/QQ截图20260915145930.png" alt="QQ截图20260915145930" style="zoom:50%;" />

