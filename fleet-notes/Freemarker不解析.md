---
title: Freemarker不解析
tags:
  - fleet-note
  - development-framework/freemarker
date: 2024-11-04
time: 23:38
aliases:
---
在 FreeMarker 中，如果你想输出 `${test.version}` 这样的字符串而不被解析，可以使用原始字符串语法（raw string）。在 FreeMarker 中，原始字符串可以通过 `r'...'` 语法实现，它会将字符串中的特殊字符保留原样输出。

### 示例

使用原始字符串来确保 `${test.version}` 不被解析：

```ftl
<version>${r'${test.version}'}</version>
```

这段代码将在生成的 XML 中输出：

```xml
<version>${test.version}</version>
```


# References