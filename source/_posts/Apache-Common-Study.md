---
title: Apache Commons类库 
date: 2017-07-20 19:39:41
tags: - tools
      - apache
categories: Java
---

[Apache common](http://commons.apache.org/)提供了很多强大的工具集，简化了Java开发人员的开发。下面是我个人一些使用心得。
<!--more-->
### 介绍下**commons-lang3** jar类库下的一些常用工具集
### *RandomStringUtils*
> 生成随机串

```java
//生成随机指定长度的字符串
RandomStringUtils.random(4);
//生成指定字符指定长度的字符串
RandomStringUtils.random(4,new char[]{'a', 'b', 'c', 'd'});
//生成指定长度的数字字符串
RandomStringUtils.randomNumeric(4);
//生成自定长度的Alpha字母串（a-z,A-Z）
RandomStringUtils.randomAlphabetic(4);
//生成指定长度的Alpha字母或数字串（a-z,A-z,0-9）
RandomStringUtils.randomAlphanumeric(4);
//获取指定长度的Ascii值在（32-126）的字符串
RandomStringUtils.randomAscii(4);
```
### *StringUtils*

> 非空判断

```
//判断是否为null或""
StringUtils.isNotEmpty("");
//判断是否为null或者""(去空格)
StringUtils.isNotBlank(" ");
//将null或" "转换为""(空串)
StringUtils.trimToEmpty("  ");
//将null或""转换为null
StringUtils.trimToNull("");
```

#### StringUtils分装的方法很多，这里就不一一列举，有兴趣可以[查看文档](http://commons.apache.org/)
