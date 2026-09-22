---
title: web做题记录：[奶龙杯2026]lets_goooooo
date: 2026-09-22
tags: [CTF,web]
categories: 安全
---



# web做题记录：[奶龙杯2026]lets_goooooo



**题目链接：https://www.ctfplus.cn/problem-detail/2092535504032501760/description**



这是一道简单的web源码审计题目和防火墙绕过题目，但是网站使用go语言写的。头一次接触go语言，感觉这个东西挺新奇，这个题不难但好多东西还是得结合ai之类的工具搜一下才知道



题目给了源码，可以直接下载审计，里面只有一个main.go文件



```
package main

import (
	"fmt"
	"html/template"
	"log"
	"net/http"
	"os/exec"
	"strings"
	"time"
)

var page = template.Must(template.New("index").Parse(`<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width,initial-scale=1">
  <title>Let's Goooooo!</title>
  <style>
    :root{color-scheme:dark;--bg:#07130d;--card:#102119;--green:#64ff9b;--muted:#91aa9a}
    *{box-sizing:border-box}body{margin:0;min-height:100vh;display:grid;place-items:center;background:radial-gradient(circle at 50% 20%,#173d27,var(--bg) 55%);font:16px/1.6 ui-monospace,SFMono-Regular,Consolas,monospace;color:#eafff0}
    main{width:min(760px,92vw);padding:38px;border:1px solid #356348;border-radius:18px;background:rgba(16,33,25,.94);box-shadow:0 25px 80px #0008}
    h1{margin:0;color:var(--green);font-size:clamp(34px,8vw,70px);letter-spacing:-4px}p{color:var(--muted)}
    form{display:flex;gap:10px;margin:28px 0}input{flex:1;min-width:0;padding:14px 16px;border:1px solid #3d6d4e;border-radius:8px;background:#08120d;color:#fff;font:inherit;outline:none}input:focus{border-color:var(--green)}button{padding:14px 22px;border:0;border-radius:8px;background:var(--green);color:#06200f;font:700 15px inherit;cursor:pointer}
    pre{min-height:130px;max-height:340px;overflow:auto;padding:18px;border-radius:10px;background:#040a06;border:1px solid #263d2e;color:#b8ffd0;white-space:pre-wrap}.tag{display:inline-block;padding:2px 8px;border:1px solid #426b50;border-radius:999px;color:#a5d6b4;font-size:12px}
  </style>
</head>
<body><main><span class="tag">GO NETWORK TOOL v1.0</span><h1>LET'S GOOOOOO!</h1><p>输入一个 IP 地址，服务器会帮你检测目标是否在线。</p>
<form action="/ping" method="get"><input name="host" placeholder="127.0.0.1" autocomplete="off" required><button type="submit">PING!</button></form>
{{if .Output}}<pre>{{.Output}}</pre>{{else}}<pre>$ waiting for target...</pre>{{end}}
</main></body></html>`))

type viewData struct{ Output string }

func index(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "text/html; charset=utf-8")
	_ = page.Execute(w, viewData{})
}

func ping(w http.ResponseWriter, r *http.Request) {
	host := r.URL.Query().Get("host")
	if host == "" || len(host) > 80 {
		http.Error(w, "invalid host", http.StatusBadRequest)
		return
	}

	if strings.ContainsAny(host, " ;|&$`(){}[]<>\\\"") {
		w.Header().Set("Content-Type", "text/html; charset=utf-8")
		_ = page.Execute(w, viewData{Output: "blocked: suspicious character detected"})
		return
	}

	ctx := r.Context()
	cmd := exec.CommandContext(ctx, "/bin/sh", "-c", fmt.Sprintf("ping -c 1 -W 1 %s", host))
	result := make(chan []byte, 1)
	go func() {
		out, _ := cmd.CombinedOutput()
		result <- out
	}()

	var out []byte
	select {
	case out = <-result:
	case <-time.After(3 * time.Second):
		_ = cmd.Process.Kill()
		out = []byte("timeout")
	}
	if len(out) > 4096 {
		out = out[:4096]
	}
	w.Header().Set("Content-Type", "text/html; charset=utf-8")
	_ = page.Execute(w, viewData{Output: string(out)})
}

func main() {
	http.HandleFunc("/", index)
	http.HandleFunc("/ping", ping)
	log.Println("listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}

```



中间写的前端代码和进入环境后的网页显示似乎这是一个用于检查网络联通性的ping测试网站，对于这种网站常见的利用方法就是命令注入进行rce



<img src="/images/web做题记录：[奶龙杯2026]lets_goooooo/QQ截图20260922141845.png" alt="QQ截图20260922141845" style="zoom:50%;" />



这里直接使用一些常规的命令注入手法肯定是不行了，前面都给了源码，这个源码不难看懂，即使是头一次见go语言也能看懂个大概。其中有一段很明显的网站后端逻辑函数



```
func ping(w http.ResponseWriter, r *http.Request) {
	host := r.URL.Query().Get("host")
	if host == "" || len(host) > 80 {
		http.Error(w, "invalid host", http.StatusBadRequest)
		return
	}

	if strings.ContainsAny(host, " ;|&$`(){}[]<>\\\"") {
		w.Header().Set("Content-Type", "text/html; charset=utf-8")
		_ = page.Execute(w, viewData{Output: "blocked: suspicious character detected"})
		return
	}

	ctx := r.Context()
	cmd := exec.CommandContext(ctx, "/bin/sh", "-c", fmt.Sprintf("ping -c 1 -W 1 %s", host))
	result := make(chan []byte, 1)
	go func() {
		out, _ := cmd.CombinedOutput()
		result <- out
	}()

	var out []byte
	select {
	case out = <-result:
	case <-time.After(3 * time.Second):
		_ = cmd.Process.Kill()
		out = []byte("timeout")
	}
	if len(out) > 4096 {
		out = out[:4096]
	}
	w.Header().Set("Content-Type", "text/html; charset=utf-8")
	_ = page.Execute(w, viewData{Output: string(out)})
}
```



这也是整个源码中占大头的处理函数，其实只看其中的一个判断部分就行



```
	if strings.ContainsAny(host, " ;|&$`(){}[]<>\\\"") {
		w.Header().Set("Content-Type", "text/html; charset=utf-8")
		_ = page.Execute(w, viewData{Output: "blocked: suspicious character detected"})
		return
	}

	ctx := r.Context()
	cmd := exec.CommandContext(ctx, "/bin/sh", "-c", fmt.Sprintf("ping -c 1 -W 1 %s", host))
	result := make(chan []byte, 1)
```



if语句里看上去就是比较标准的防火墙写法了，**strings.ContainsAny函数就是检查一个字符串中是否有另一个字符串中的字符**，if语句里面就是一个报错提示，就说明我们输入的host不能出现if判断里面指定的字符。绕过if判断之后有一个exec.CommandContext函数，这个函数就是执行命令，也就是直接执行了后面的ping命令



有了防火墙逻辑之后就是绕过的思路问题了，判断里禁了分号，各种括号，反斜杠，美元符之类的，但是没有禁换行符，也就是\n，但是这里\被禁了，也就不能直接用\n来利用。这里则可以考虑使用url编码，换行符的url编码是%0a，使用%0a就可以成功运行ls命令（为了url编码能够被识别，这里不能直接在输入框里注入命令，而是改一下url里面的get参数host）



```
?host=127.0.0.1%0Als
```



<img src="/images/web做题记录：[奶龙杯2026]lets_goooooo/QQ截图20260922143944.png" alt="QQ截图20260922143944" style="zoom:50%;" />



有了flag路径，接下来要执行的命令就是cat flag，但是这里测试的空格和加号都不能直接使用，这里就要用到一个东西叫制表符，也就是Tab键输入的符号。linux中的一些编辑器可以将制表符识别成空格。制表符的url编码是%09，使用%09代替空格就能cat flag



```
?host=127.0.0.1%0Acat%09flag
```



<img src="/images/web做题记录：[奶龙杯2026]lets_goooooo/QQ截图20260922145008.png" alt="QQ截图20260922145008" style="zoom:50%;" />