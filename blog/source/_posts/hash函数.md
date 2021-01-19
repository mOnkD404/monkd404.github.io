---
title: hash函数
date: 2021-01-19 15:21:19
tags:
---

Chromium 代码中使用了一个 SuperFastHash 函数当做base库默认的hash函数，相关文档 http://www.azillionmonkeys.com/qed/hash.html , 按文档的benchmark测试结果，该函数的性能远高于常用计算checksum的crc32，fnv等hash函数。
