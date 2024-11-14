---
title: Maven-archetype使用
tags:
  - fleet-note
  - build-tools/maven
date: 2024-10-30
time: 13:15
aliases:
---
# 空文件夹问题

如果有空文件夹，在使用插件 `create-from-project` 时不会打入空文件夹，该插件并不支持该功能。
[\[ARCHETYPE-459\] Ability to include empty subdirectories - ASF JIRA](https://issues.apache.org/jira/browse/ARCHETYPE-459)

# 制作流程

## Step1：主工程 `POM` 引入插件

在主工程 `POM` 中引入插件：

```xml
<build>
	<plugins>
		<plugin>
			<groupId>org.apache.maven.plugins</groupId>
			<artifactId>maven-archetype-plugin</artifactId>
			<version>3.2.0</version>
		</plugin>
	</plugins>
</build>
```

之后可以执行 `archetype:create-from-project` 插件生成骨架代码

![image.png](https://images.hnzhrh.com/note/20241030223722.png)

在生成的 `target` 目录中，`generated-sources/archetype/src` 内容为后续要更改的内容。

![image.png](https://images.hnzhrh.com/note/20241030224344.png)

使用 `pom.xml` 1 创建一个新的 `Maven` 工程

## Step 2：修改内容

修改 `pom.xml` 2 号文件，该文件为生成骨架代码的依赖，为了适配用户自定义的 groupId 和 artifactId 需要修改为占位符。
例如：
```xml
<dependency>  
    <groupId>${groupId}</groupId>  
    <artifactId>${rootArtifactId}-adapter</artifactId>  
</dependency>
```

修改 `/src/main/resources/META-INF/maven/archetype-metadata.xml` :

```xml
<module id="${rootArtifactId}-adapter" dir="__rootArtifactId__-adapter" name="${rootArtifactId}-adapter">
```

id 和 name改法一直，目录改为 `__rootArtifactId__` 

同时将对应的目录文件名也改为这样

### Step3: Install

install 后，可在本地仓库的  `archetype-catalog.xml` 文件中看到注册的 archetype，之后就可以i通过该archetype创建项目了。

# References

* [Maven Archetype-CSDN博客](https://blog.csdn.net/luo15242208310/article/details/120411215)
* [Maven Archetype 多 Module 自定义代码脚手架大部分公司都会有一个通用的模板项目，帮助你快速创建一个 - 掘金](https://juejin.cn/post/7052965570660007967)