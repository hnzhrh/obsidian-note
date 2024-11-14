---
title: Swagger注解使用
tags:
  - fleet-note
  - development-framework/swagger
date: 2024-10-29
time: 21:38
aliases:
---
# annotations

* `@Tag(name = "")` ，用在 `Controller` 类上
* `@Operation(summary = "接口Summary", description = "接口描述")` ，用在 `Controller` 类方法上
* `@Parameters({@Parameter(name = "name", description = "Form请求参数描述")})` ，用在 `Controller` 的方法上，接收参数为表单参数
* `@Schema(title = "请求参数Title", description = "请求参数description")`，用在请求参数类上
* `@Schema(title = "请求字段Title", description = "请求字段Desc")`，用在请求参数类的成员变量上
* 响应的 `VO` 与请求参数一致
# Demo code

```java

@Tag(name = "Controller tag")  
public class TestController {  
  
    @Autowired  
    private TestMapper testMapper;  
  
    @Autowired  
    private RedissonClient redissonClient;  
  
    @Operation(summary = "接口summary1", description = "接口description1")  
    @Parameters({@Parameter(name = "name", description = "表单参数description")})  
    @GetMapping(value = "/hello")  
    public SingleResponse<String> test(@RequestParam(required = true, name = "name") String name) {  
    }  
  
  
    @Operation(summary = "接口summary2", description = "接口description2")  
    @PostMapping(value = "/test_swagger")  
    public SingleResponse<TestDTO> testSwagger(@RequestBody TestQry query) {  
    }  
}


@Schema(title = "请求参数title", description = "请求参数description")  
public class TestQry {  
    @NotBlank(message = "Name is not black")  
    @Schema(title = "请求字段title", description = "请求字段description")  
    private String name;  
}

@Schema(title = "响应title", description = "响应description")  
public class TestDTO {  
    @Schema(description = "响应字段description", title = "响应字段title")  
    private String name;  
}
```

响应字段建议只是用 `description` ，`Swagger models` 不会渲染 `title`

# Image

![image.png](https://images.hnzhrh.com/note/20241029220849.png)



![image.png](https://images.hnzhrh.com/note/20241029221004.png)

# References
* [spring boot整合swagger3,swagger注解详解\_swagger3注解-CSDN博客](https://blog.csdn.net/leonnew/article/details/128142640)