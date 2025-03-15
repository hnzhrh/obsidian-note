---
title: count(*) 和 count(1) 有什么区别？哪个性能最好？
tags:
  - permanent-note
  - middleware/database/mysql
date: 2025-03-14
time: 18:14
aliases: 
done: false
---

结论：
```shell
count(*) = count(1) > count(主键字段) > count(普通字段)
```

# Reference
* [count(\*) 和 count(1) 有什么区别？哪个性能最好？ \| 小林coding](https://xiaolincoding.com/mysql/index/count.html)