---
title: Maven打包Spring-boot
tags:
  - fleet-note
  - build-tools/maven
date: 2024-10-29
time: 18:27
aliases:
---

# 父工程POM：

```xml
	<build>
        <pluginManagement>
            <plugins>
                <plugin>
                    <artifactId>maven-resources-plugin</artifactId>
                    <version>3.3.0</version>
                </plugin>
                <plugin>
                    <artifactId>maven-compiler-plugin</artifactId>
                    <version>3.10.1</version>
                </plugin>
                <plugin>
                    <artifactId>maven-source-plugin</artifactId>
                    <version>3.2.1</version>
                </plugin>
                <plugin>
                    <artifactId>maven-javadoc-plugin</artifactId>
                    <version>3.4.0</version>
                </plugin>
                <plugin>
                    <artifactId>maven-deploy-plugin</artifactId>
                    <version>3.0.0</version>
                </plugin>
                <plugin>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-maven-plugin</artifactId>
                    <version>${spring-boot.version}</version>
                    <configuration>
                        <excludes>
                            <exclude>
                                <groupId>org.projectlombok</groupId>
                                <artifactId>lombok</artifactId>
                            </exclude>
                        </excludes>
                    </configuration>
                </plugin>
            </plugins>
        </pluginManagement>
    </build>
```

# 主工程POM：

```xml
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <mainClass>com.erpang.scaffold.Application</mainClass>
                </configuration>
                <executions>
                    <execution>
                        <id>repackage</id>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

# Repackage 的作用

使用 `repackage` 目标的原因如下：

1. **创建可执行 JAR**：`repackage` 会将应用程序及其所有依赖项打包成一个单一的可执行 JAR 文件，使得应用程序可以轻松运行。

2. **包含依赖**：此目标会将所有项目依赖（包括其他 JAR 文件）嵌入到最终的 JAR 文件中，简化了部署过程。

3. **修改 JAR 结构**：`repackage` 可以调整 JAR 文件的结构，以符合 Spring Boot 的要求，如将 `META-INF` 中的启动类信息更新。

4. **便于运行**：生成的可执行 JAR 可以直接通过 `java -jar` 命令运行，而无需手动设置类路径。

总体而言，`repackage` 使得 Spring Boot 应用的打包和部署变得更加方便和一致。

# 生成的包

在使用 Spring Boot Maven 插件打包应用程序时，生成的 `.jar` 文件和 `.jar.original` 文件的区别如下：

1. **`.jar` 文件**：
   - 这是最终的可执行 JAR 文件，包含了应用程序的代码、资源和所有依赖项。可以通过 `java -jar your-app.jar` 命令直接运行。

2. **`.jar.original` 文件**：
   - 这是在打包过程中生成的原始 JAR 文件，通常包含了项目的原始内容和结构。
   - 在执行 `repackage` 目标时，插件会先创建这个原始 JAR 文件，然后再创建最终的可执行 JAR 文件。这个文件的存在主要是为了保留项目的原始状态，以便在需要时进行参考或调试。
