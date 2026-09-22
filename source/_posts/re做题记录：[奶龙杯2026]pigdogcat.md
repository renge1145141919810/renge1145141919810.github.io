---
title: re做题记录：[奶龙杯2026]pigdogcat
date: 2026-09-22
tags: [CTF,reverse]
categories: 安全
---



# re做题记录：[奶龙杯2026]pigdogcat



**题目链接：https://www.ctfplus.cn/problem-detail/2092535445387743232/description**



**这道题目没有完整的复现环境，比赛时原本的环境是一个完整的minecraft服务器，提供的附件也是一个minecraft的mod**，这道题目和mc结合起来很有意思，很可惜没有现成的复现环境，这里能够演示的图片有的来自比赛时的环境



下载下来的jar包是一个minecraft 1.8.8版本的mod包，进入服务器会先给一个token（游戏演示用图片来着比赛环境）



![QQ图片20260922150742](/images/re做题记录：[奶龙杯2026]pigdogcat/QQ图片20260922150742.jpg)



这个mod包有自己的功能，服务器里面暂停游戏会有一个“super secret settings”按钮，按一下会切换随机一个生物的视角



<img src="/images/re做题记录：[奶龙杯2026]pigdogcat/图片2.png" alt="图片2" style="zoom:50%;" />



这里游戏中的对话框似乎能够输入一些命令，输入/help就可以看见能够使用哪些命令，其中有/flag，这个命令应该就是拿flag的命令了



<img src="/images/re做题记录：[奶龙杯2026]pigdogcat/图片1.png" alt="图片1" style="zoom:50%;" />



游戏里能找到的信息就这些，这时就要分析拿到的mod文件包了。对于java相关的逆向我也没有怎么接触过，一开始也不知道用什么工具，只知道apk的文件可以用jadx打开分析，也不是很会做。拿到的文件格式是jar，一开始也不知道分析哪个文件，**后来才清楚这里需要使用jd-gui对FlagPlugin.class进行分析，这个文件用jadx打开只能看见二进制数据**



分析出来的代码能够看见服务器的一些逻辑，对我来说java代码看起来还是很困难(˃̣̣̥⌓˂̣̣̥ )っ



```
package com.ctf.flagplugin;

import java.security.MessageDigest;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;
import org.bukkit.ChatColor;
import org.bukkit.Location;
import org.bukkit.Material;
import org.bukkit.World;
import org.bukkit.command.Command;
import org.bukkit.command.CommandSender;
import org.bukkit.entity.Player;
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;
import org.bukkit.event.player.PlayerJoinEvent;
import org.bukkit.plugin.Plugin;
import org.bukkit.plugin.java.JavaPlugin;

public class FlagPlugin extends JavaPlugin {
  private final Map a = new HashMap<Object, Object>();

  private final Set b = new HashSet();

  private static final byte[] c = new byte[] { 
      22, 1, 19, 10, 27, 26, 27, 22, 16, 10, 
      103, 101, 103, 99 };

  private static final byte[] d = new byte[] { 112, 72, 69, 103, 75, 66, 101, 70, 92, 8 };

  private static final int[] e = new int[] { 52222, 80, 48062 };

  public void onEnable() {
    getServer().getPluginManager().registerEvents((Listener)this, (Plugin)this);
    a();
    getLogger().info("FlagPlugin v2.0 enabled.");
  }

  private void a() {
    World world = getServer().getWorlds().get(0);
    int i = e[0] ^ 0xCAFE;
    int j = e[1];
    int k = e[2] ^ 0xBABE;
    world.getBlockAt(i, j, k).setType(Material.SPONGE);
    for (byte b = 1; b <= 3; b++)
      world.getBlockAt(i, j + b, k).setType(Material.AIR); 
    getLogger().info("Ritual stone placed at " + i + "," + j + "," + k);
  }

  @EventHandler
  public void onJoin(PlayerJoinEvent paramPlayerJoinEvent) {
    Player player = paramPlayerJoinEvent.getPlayer();
    String str = a(player.getName());
    this.a.put(player.getUniqueId(), str);
    player.sendMessage(ChatColor.DARK_GRAY + "[" + ChatColor.GRAY + "Auth" + ChatColor.DARK_GRAY + "] " + ChatColor.GRAY + "Token: " + ChatColor.WHITE + str);
  }

  public boolean onCommand(CommandSender paramCommandSender, Command paramCommand, String paramString, String[] paramArrayOfString) {
    if (!(paramCommandSender instanceof Player)) {
      paramCommandSender.sendMessage("Players only.");
      return true;
    } 
    Player player = (Player)paramCommandSender;
    switch (paramCommand.getName().toLowerCase()) {
      case "auth":
        a(player, paramArrayOfString);
        break;
      case "flag":
        a(player);
        break;
    } 
    return true;
  }

  private void a(Player paramPlayer, String[] paramArrayOfString) {
    if (paramArrayOfString.length == 0) {
      paramPlayer.sendMessage(ChatColor.RED + "Usage: /auth <response>");
      return;
    } 
    String str = (String)this.a.get(paramPlayer.getUniqueId());
    if (str == null) {
      paramPlayer.sendMessage(ChatColor.RED + "No active session. Reconnect.");
      return;
    } 
    if (a(str, b(d)).equals(paramArrayOfString[0])) {
      this.b.add(paramPlayer.getUniqueId());
      this.a.remove(paramPlayer.getUniqueId());
      paramPlayer.sendMessage(ChatColor.GREEN + "Authenticated.");
      paramPlayer.sendMessage(ChatColor.DARK_GRAY + "The ancient stone awaits.");
    } else {
      paramPlayer.sendMessage(ChatColor.RED + "Invalid response.");
    } 
  }

  private void a(Player paramPlayer) {
    if (!this.b.contains(paramPlayer.getUniqueId())) {
      paramPlayer.sendMessage(ChatColor.RED + "Admins only. Use /auth first.");
      return;
    } 
    int i = e[0] ^ 0xCAFE;
    int j = e[1];
    int k = e[2] ^ 0xBABE;
    Location location = new Location(paramPlayer.getWorld(), i, j, k);
    if (paramPlayer.getLocation().distanceSquared(location) > 25.0D) {
      paramPlayer.sendMessage(ChatColor.DARK_GRAY + "You sense the flag is near... but not here.");
      return;
    } 
    String str = System.getenv("FLAG");
    if (str == null || str.isEmpty())
      str = "flag{test_placeholder}"; 
    paramPlayer.sendMessage(ChatColor.GOLD + "" + ChatColor.BOLD + str);
  }

  private String a(String paramString) {
    try {
      MessageDigest messageDigest = MessageDigest.getInstance("SHA-1");
      messageDigest.update(paramString.toLowerCase().getBytes("UTF-8"));
      messageDigest.update(a(c));
      byte[] arrayOfByte = messageDigest.digest();
      StringBuilder stringBuilder = new StringBuilder(8);
      for (byte b = 0; b < 4; b++) {
        stringBuilder.append(String.format("%02x", new Object[] { Integer.valueOf(arrayOfByte[b] & 0xFF) }));
      } 
      return stringBuilder.toString();
    } catch (Exception exception) {
      return "00000000";
    } 
  }

  private String a(String paramString, byte[] paramArrayOfbyte) {
    try {
      MessageDigest messageDigest = MessageDigest.getInstance("SHA-1");
      messageDigest.update(paramString.getBytes("UTF-8"));
      messageDigest.update(paramArrayOfbyte);
      byte[] arrayOfByte = messageDigest.digest();
      StringBuilder stringBuilder = new StringBuilder(16);
      for (byte b = 0; b < 8; b++) {
        stringBuilder.append(String.format("%02x", new Object[] { Integer.valueOf(arrayOfByte[b] & 0xFF) }));
      } 
      return stringBuilder.toString();
    } catch (Exception exception) {
      return "";
    } 
  }

  private static byte[] a(byte[] paramArrayOfbyte) {
    byte[] arrayOfByte = new byte[paramArrayOfbyte.length];
    for (byte b = 0; b < paramArrayOfbyte.length; b++)
      arrayOfByte[b] = (byte)(paramArrayOfbyte[b] ^ 0x55); 
    return arrayOfByte;
  }

  private static byte[] b(byte[] paramArrayOfbyte) {
    byte[] arrayOfByte = new byte[paramArrayOfbyte.length];
    for (byte b = 0; b < paramArrayOfbyte.length; b++)
      arrayOfByte[b] = (byte)(paramArrayOfbyte[b] ^ b + 32); 
    return arrayOfByte;
  }
}
```



这里先注意flag出现的地方，有一个函数



```
 private void a(Player paramPlayer) {
    if (!this.b.contains(paramPlayer.getUniqueId())) {
      paramPlayer.sendMessage(ChatColor.RED + "Admins only. Use /auth first.");
      return;
    } 
    int i = e[0] ^ 0xCAFE;
    int j = e[1];
    int k = e[2] ^ 0xBABE;
    Location location = new Location(paramPlayer.getWorld(), i, j, k);
    if (paramPlayer.getLocation().distanceSquared(location) > 25.0D) {
      paramPlayer.sendMessage(ChatColor.DARK_GRAY + "You sense the flag is near... but not here.");
      return;
    } 
    String str = System.getenv("FLAG");
    if (str == null || str.isEmpty())
      str = "flag{test_placeholder}"; 
    paramPlayer.sendMessage(ChatColor.GOLD + "" + ChatColor.BOLD + str);
  }
```



这里给了拿flag的逻辑，先用/auth进行一个校验，用e然后生成一个坐标，在调用paramPlayer.getLocation().distanceSquared()，这应该是一个判断玩家所在坐标和目的坐标距离的函数，这个距离要小于5格（5x5=25），然后使用flag命令拿flag。这里的e前面已经给了，也就是说坐标的逻辑有了可以直接推出来



```
 private static final int[] e = new int[] { 52222, 80, 48062 };
```



前面有一个函数也给了这个坐标的逻辑



```
 private void a() {
    World world = getServer().getWorlds().get(0);
    int i = e[0] ^ 0xCAFE;
    int j = e[1];
    int k = e[2] ^ 0xBABE;
    world.getBlockAt(i, j, k).setType(Material.SPONGE);
    for (byte b = 1; b <= 3; b++)
      world.getBlockAt(i, j + b, k).setType(Material.AIR); 
    getLogger().info("Ritual stone placed at " + i + "," + j + "," + k);
  }
```



用e的数值和两个值异或，然后**调用world.getBlockAt().setType()，参数里有Material.SPONGE，sponge是海绵块的意思，这里大概意思就是在固定坐标生成一个海绵块，要找到也就是这个海绵块了。**加密方法都是硬编码，可以直接解出来是（256，80，256）



<img src="/images/re做题记录：[奶龙杯2026]pigdogcat/QQ截图20260922165149.png" alt="QQ截图20260922165149" style="zoom:50%;" />



校验的部分在下面的函数里



```
  private void a(Player paramPlayer, String[] paramArrayOfString) {
    if (paramArrayOfString.length == 0) {
      paramPlayer.sendMessage(ChatColor.RED + "Usage: /auth <response>");
      return;
    } 
    String str = (String)this.a.get(paramPlayer.getUniqueId());
    if (str == null) {
      paramPlayer.sendMessage(ChatColor.RED + "No active session. Reconnect.");
      return;
    } 
    if (a(str, b(d)).equals(paramArrayOfString[0])) {
      this.b.add(paramPlayer.getUniqueId());
      this.a.remove(paramPlayer.getUniqueId());
      paramPlayer.sendMessage(ChatColor.GREEN + "Authenticated.");
      paramPlayer.sendMessage(ChatColor.DARK_GRAY + "The ancient stone awaits.");
    } else {
      paramPlayer.sendMessage(ChatColor.RED + "Invalid response.");
    } 
  }
```



中间有一段if判断，确定了用/auth进行校验的逻辑，及a(str,b(d)).equals(paramArrayofString[0])



```
    if (a(str, b(d)).equals(paramArrayOfString[0])) {
      this.b.add(paramPlayer.getUniqueId());
      this.a.remove(paramPlayer.getUniqueId());
      paramPlayer.sendMessage(ChatColor.GREEN + "Authenticated.");
      paramPlayer.sendMessage(ChatColor.DARK_GRAY + "The ancient stone awaits.");
    } else {
      paramPlayer.sendMessage(ChatColor.RED + "Invalid response.");
    } 
```



这里调用了一个a函数和一个b函数，a函数在这里有好几个，不是很清楚具体调用的是哪个。b函数下面有



```
  private static byte[] b(byte[] paramArrayOfbyte) {
    byte[] arrayOfByte = new byte[paramArrayOfbyte.length];
    for (byte b = 0; b < paramArrayOfbyte.length; b++)
      arrayOfByte[b] = (byte)(paramArrayOfbyte[b] ^ b + 32); 
    return arrayOfByte;
  }
```



b函数里面就是一个加密操作，先对索引异或然后加32。前面if判断里面b函数的参数是d，d在前面也有



```
  private static final byte[] d = new byte[] { 112, 72, 69, 103, 75, 66, 101, 70, 92, 8 };
```



可以先把b（d）解出来看看，照着加密算法算一次就行



<img src="/images/re做题记录：[奶龙杯2026]pigdogcat/QQ截图20260922170803.png" alt="QQ截图20260922170803" style="zoom:80%;" />



而前面a函数的参数str则是根据玩家id生成的一个值，这个值在生成token时也出现了



```
  public void onJoin(PlayerJoinEvent paramPlayerJoinEvent) {
    Player player = paramPlayerJoinEvent.getPlayer();
    String str = a(player.getName());
    this.a.put(player.getUniqueId(), str);
    player.sendMessage(ChatColor.DARK_GRAY + "[" + ChatColor.GRAY + "Auth" + ChatColor.DARK_GRAY + "] " + ChatColor.GRAY + "Token: " + ChatColor.WHITE + str);
  }
```



这里似乎不需要知道token是怎么来的，具体的token生成算法和前面一段硬编码的数据c有关系，但是这里没有用到，下文进行校验的str就是进入服务器时给玩家的token，这是只需要知道a函数逻辑就行了。



经过参数对比，发现的a函数如下



```
  private String a(String paramString, byte[] paramArrayOfbyte) {
    try {
      MessageDigest messageDigest = MessageDigest.getInstance("SHA-1");
      messageDigest.update(paramString.getBytes("UTF-8"));
      messageDigest.update(paramArrayOfbyte);
      byte[] arrayOfByte = messageDigest.digest();
      StringBuilder stringBuilder = new StringBuilder(16);
      for (byte b = 0; b < 8; b++) {
        stringBuilder.append(String.format("%02x", new Object[] { Integer.valueOf(arrayOfByte[b] & 0xFF) }));
      } 
      return stringBuilder.toString();
    } catch (Exception exception) {
      return "";
    } 
  }
```



这里逻辑就是**先将str进行utf-8编码后将两个字符串合并，然后进行一个sha-1操作得到一个哈希值。注意这里将字符串合并调用的是update，合并的字符串有严格的顺序，前面是token，后面是之前解出来的PigDogCat!。**有了这个逻辑，就可以直接解出来验证密码了



![QQ截图20260922172711](/images/re做题记录：[奶龙杯2026]pigdogcat/QQ截图20260922172711.png)



现在所有需要的东西都有了，解题流程就是**先进入游戏拿到token，利用token计算验证值用/auth命令进行验证，然后去坐标(256,80,256)找一个海绵块，在海绵块周围5格处使用/flag命令拿flag。**照着这个流程玩一下就行



先把校验值输进去校验



<img src="/images/re做题记录：[奶龙杯2026]pigdogcat/图片3.png" alt="图片3" style="zoom: 67%;" />



跑到（256，80，256）附近找海绵块



<img src="/images/re做题记录：[奶龙杯2026]pigdogcat/图片4.png" alt="图片4" style="zoom:67%;" />



海绵块生成到天上了，搭方块上去，在对话框输入/flag就给flag了



<img src="/images/re做题记录：[奶龙杯2026]pigdogcat/图片5.png" alt="图片5" style="zoom:67%;" />



本来还在想有这样的题目那之后复现岂不是开一个环境就有一个免费的mc联机服务器了(*ˊᗜˋ*)可惜了这种还挺有意思的题目无法找到复现的完整环境，似乎配置起来也十分困难(╥﹏╥)