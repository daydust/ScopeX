# ScopeX
泛查询漏洞扫描Burp插件

`ScopeX` 是一个 Burp 插件，用来辅助做“泛查询 / 越权 / 参数置空类”测试。  
它的核心思路是：把一个请求送进插件后，自动对你选定作用域里的参数逐个做 mutation，然后回放请求，并直接展示：

- 原始响应体长度
- 变异后响应体长度
- 长度差值 `Delta`
- 原始请求 / 原始响应
- 变异请求 / 变异响应

## 版本说明

1、2026.4.28发布第一版

开发环境：JDK17 Burp2024.3.1

2、2026.6.26 发布2.0

新增自定义参数，新增结果排序，优化响应和UI

## 支持的请求类型

当前支持：

- `GET` query 参数
- `POST application/x-www-form-urlencoded`
- `POST application/json`
- 嵌套 JSON，例如 `query.filters.categoryId`
- 请求头参数
- `Cookie` 拆分后的单个 cookie 参数

## Mutation 规则

默认会对每个参数逐个测试，不是同时改多个参数。

例如原始请求：

```json
{
  "categoryId": "cat-1001",
  "pageNum": 1,
  "pageSize": 5
}
```

会拆成：

- 先只改 `categoryId`
- 再只改 `pageNum`
- 再只改 `pageSize`

每个参数默认会尝试：

- `置空`
- `删除`
- `置 0`
- `百分号 %`

如果原值本身像容器值，还会加：

- `[]`
- `{}`

## `%` 的处理规则

- `GET / query` 中的 `%` 会编码成 `%25`
- `POST form` 中的 `%` 不编码
- `POST JSON` 中的 `%` 不编码
- `Header` 中的 `%` 不编码
- `Cookie` mutation 中的 `%` 不编码

这样做的目的是：

- 避免 `GET` 场景因为裸 `%` 导致 `400`
- 保留 `POST` 场景下原始 `%` 的测试语义

## Request Header 是什么

`Request Header` 不是“只测 Cookie”。

它分两种情况：

- 普通请求头：
  会识别 `X-*`、名称里带 `token`、`tenant` 的头，然后直接对这个头值做 mutation
- `Cookie`：
  不会整条 Cookie 一起乱改，而是先拆成单个 cookie 键值，再对每个 cookie 单独做 mutation

例如：

```http
Cookie: uid=1001; tenantId=tenant-a; sid=abc123
```

会拆成：

- `Cookie.uid`
- `Cookie.tenantId`
- `Cookie.sid`

然后分别测试。

## 怎么加载

### 1. Burp 2024.3.1

1. 打开 Burp
2. 进入 `Extensions`
3. 选择 `Add`
4. 选择 Java 类型扩展
5. 加载：

加载成功后，你会看到页签名字是 `ScopeX`。

## 怎么使用

觉得有泛查询的位置直接右键发送到插件，让插件进行自动化扫描即可

## 插件界面说明

### Scope

用于选择作用域
- `Request first line`
  只测试 URL query 参数
- `Request header`
  只测试 Header / Cookie 参数
- `Request body`
  只测试 POST body 参数

### Rebuild Preview

重新对当前选中的请求做一轮 mutation 和自动重放。

适合你切换了 Scope 后重新跑。

### Clear

清空当前插件面板中的：

- 已捕获请求
- 候选 mutation 列表
- 原始请求/响应
- 变异请求/响应

### Candidate Mutations

这里会展示每条候选：

- `Parameter`
- `Location`
- `Mutation`
- `Original`
- `Mutated`
- `Delta`

其中：

- `Original` = 原始响应体长度
- `Mutated` = 当前 mutation 请求返回的响应体长度
- `Delta` = 二者差值

### 下方四个窗口

- `Original Request`
- `Original Response`
- `Mutated Request`
- `Mutated Response`

选中某条 mutation 后，就能直接看这条测试对应的请求和响应。

## 适合怎么判断结果

一般重点看：

- 响应长度是否明显变大
- 原本单条 / 小范围数据是否变成全量数据
- `total / count / list` 是否明显变化
- 是否回显了其他租户或其他用户的数据

## 说明

这个插件更适合“授权测试下的人工辅助验证”，它不是为了替代人工判断。  
最好的使用方式，是先让插件帮你快速扫出长度变化明显的参数，再结合响应内容做人工确认。
