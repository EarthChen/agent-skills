---
name: amar-generate-api-doc
description: 为 Java MoaWebService 接口生成 API 文档。当用户要求"生成 API 文档"、"创建接口文档"、"写接口文档"或需要为 MoaWebService 接口生成包含请求/响应示例的 Markdown 文档时使用。
---

# 生成 API 文档

为 Java MoaWebService 接口生成 Markdown 格式的 API 文档。

## 工作流程

1. 读取 MoaWebService 接口文件，识别方法和 @path 注解
2. 读取相关的 Param（请求）类获取字段定义
3. 读取相关的 Vo（响应）类获取响应结构
4. 按指定格式生成 Markdown 文档

## 输出格式

每个 API 方法生成以下格式：

```markdown
## {从 @path 注解获取的 URL}

{方法描述}

### Req

```json
{
    "field1": "示例值",
    "field2": 123
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| field1 | String | 是 | 字段说明 |
| field2 | Integer | 否 | 字段说明 |

### Resp

```json
{
    "ec": 0,
    "em": "success",
    "data": {
        "result": "示例"
    }
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| result | String | 字段说明 |
```

## 字段映射规则

- 从 JavaDoc 注释或 @see 注解提取字段说明
- Java 类型映射：String、Integer/int、Long/long、Boolean/boolean、List、LocalDateTime
- 通过 @NotNull、@NotBlank、@NotEmpty 判断必填字段
- 通过 @Min、@Max 获取值约束
- 请求示例中跳过父类字段（如 ClientCommonReq）
- 根据字段名和类型使用有意义的示例值

## 输出位置

文档保存到 `.docs/API/{ServiceName}-API.md`

## URL 规则

URL 路径来自方法 JavaDoc 注释中的 @path 注解。确保 URL 最后一段与方法名一致。
