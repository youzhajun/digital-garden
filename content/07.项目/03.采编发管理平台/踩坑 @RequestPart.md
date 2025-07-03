---
title: 踩坑 @RequestPart
draft: false
tags:
  - 项目
date: 2025-03-05
---
## 怎么入坑的？
具体来说是在工具上入坑的，项目 `api` 文档使用国产工具 `apifox` ，使用 `apifox` 提供的插件导入接口时发现无法导入该接口，于是自己手动导入到 `apifox` 中。
- `@RequestPart` 的提交类型是 `multipart/form-data`。（没有什么问题，很容易设置）
- `form-data` 中有两个字段，`file` 与 `data`。 （没有什么问题）
- `form-data` 中的 `file` - 文件类型， `data` - string 类型的 `json` 串。（没有什么问题，比较好设置）

OK！一切就绪，发送请求。。。。 error 报错。检查再三，发现需要单独设置 `form-data` 中 `data` 的 `context-type` 属性！结果偏偏 `apifox` 与 `postman` 中这个设置项很是偏僻！（攥拳很气了）


![[踩坑 @RequestPart-1751513344372.png]]

![[踩坑 @RequestPart-1751513359642.png]]

## 是什么？能做什么？

`@RequestPart` 是 Spring Framework（通常用于 Spring Boot 项目）中用于处理 HTTP 请求中 `multipart/form-data` 类型的一种注解，主要用于接收 **表单中的文件上传** 以及同时携带的其他非文件类型的字段。

## 怎么用？
### 接收文件与单字段

**后端实现：**
```java
@PostMapping("/upload")
public ResponseEntity<String> uploadFile(
    @RequestPart("file") MultipartFile file,
    @RequestPart("description") String description
) {
    // 保存文件或处理业务逻辑
    return ResponseEntity.ok("上传成功");
}
```

**前端请求：**
```http
POST /upload HTTP/1.1
Content-Type: multipart/form-data

file: test.jpg
description: 这是一个图片描述
```

### 接收文件与 JSON 对象

**后端实现：**
```java
// bean 信息
public class FileMeta {
    private String name;
    private String description;
    // getter/setter
}


@PostMapping("/upload")
public ResponseEntity<String> upload(
    @RequestPart("file") MultipartFile file,
    @RequestPart("meta") FileMeta meta
) {
    // file 是上传的文件，meta 是描述信息
    return ResponseEntity.ok("上传成功");
}

```

**前端请求：**
```javascript

const formData = new FormData();

formData.append("file", fileInput.files[0]);
// 对象转json 并需要设置context-type ！！！！
formData.append("meta", new Blob([JSON.stringify({name: "test", description: "desc"})], { type: "application/json" }));

fetch("/upload", {
  method: "POST",
  body: formData
});

```

## `@RequestParam` vs `@RequestPart`
| 特点           | `@RequestParam` | `@RequestPart`              |
| ------------ | --------------- | --------------------------- |
| 请求类型         | 通常用于普通表单字段或查询参数 | 专门处理 `multipart/form-data`  |
| 支持文件上传       | 支持，但更适合用于简单文件   | 更适合处理 `MultipartFile` 等复杂对象 |
| 支持复杂 JSON 对象 | 不支持（除非使用额外处理）   | 支持，能够绑定 JSON 到 Java Bean    |
