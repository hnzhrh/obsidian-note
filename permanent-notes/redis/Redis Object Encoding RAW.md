---
title: Redis Object Encoding RAW
tags:
  - permanent-note
  - middleware/redis/data-object-encoding
date: 2025-03-04
time: 15:59
aliases:
---
当 `EMBSTR` 编码的字符串被修改、大于 44 字节的字符串且不是整数（在最大范围之内），则会使用 `RAW` 编码。
![image.png](https://images.hnzhrh.com/note/20241212141849.png)
# Reference