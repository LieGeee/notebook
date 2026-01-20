# 知识薄弱点分析与学习笔记

  

> 根据 Driver.js 引导功能开发过程中暴露的问题整理

  

---

  

## 一、Go 语法基础

  

### 1.1 函数大括号位置 ❌

  

**你的写法：**

```go

func (s *userService) UpdateGuided(...) error

{

    // 报错！

}

```

  

**正确写法：**

```go

func (s *userService) UpdateGuided(...) error {

    // Go 要求 { 必须和函数签名在同一行

}

```

  

**原因：** Go 编译器会在行尾自动插入分号，换行会导致语法错误。

  

---

  

### 1.2 拼写错误

  

| 你写的 | 正确的 | 类型 |

|--------|--------|------|

| `fuc` | `func` | 关键字 |

| `grom` | `gorm` | tag 名 |

| `Messsage` | `Message` | 字段名 |

| `conde` | `Code` | 字段名 |

  

**建议：** 使用 IDE 的拼写检查插件，或写完后用 `go build` 检查。

  

---

  

### 1.3 多余的大括号

  

**你的写法：**

```go

users.PUT("/me/guided", ...)

}  // ❌ 多余的 }

}

```

  

**原因：** 复制粘贴时没注意代码块边界。

  

**建议：** 每次添加代码后，检查 `{` 和 `}` 是否配对。

  

---

  

## 二、项目结构认知

  

### 2.1 不熟悉 struct 字段名

  

**你的写法：**

```go

return s.repo.UpdateGuided(...)  // ❌ 字段名错误

```

  

**正确写法：**

```go

return s.userRepo.UpdateGuided(...)  // ✅ 项目用的是 userRepo

```

  

**问题：** 没有先查看 struct 定义就直接写代码。

  

**建议：** 写代码前先看 struct 定义：

```go

type userService struct {

    userRepo repository.UserRepository  // ← 看这里

}

```

  

---

  

### 2.2 不熟悉 gRPC 客户端获取方式

  

**你的写法：**

```go

resp, err := p.users.UpdateGuided(...)  // ❌ 没有 users 字段

```

  

**正确写法：**

```go

client, err := p.clientManager.GetUserServiceClient()  // 先获取 client

resp, err := client.UpdateGuided(...)                   // 再调用方法

```

  

**建议：** 参考项目中其他方法的写法，保持一致。

  

---

  

### 2.3 结构体名称混淆

  

| 文档写的 | 项目实际用的 |

|----------|--------------|

| `UserHandler` | `UserGrpcHandler` |

  

**建议：** 写代码前用 `Ctrl+Shift+F` 搜索确认实际名称。

  

---

  

## 三、Protobuf 理解

  

### 3.1 rpc 方法位置错误 ❌❌❌

  

**你的写法：**

```protobuf

service UserService {

  rpc CreateUser(...) returns (...);

}  // service 已经关闭

  

message UpdateGuidedRequest { ... }

message UpdateGuidedResponse { ... }

  

rpc UpdateGuided(...) returns (...);  // ❌ 在 service 外面！

}  // ❌ 多余的 }

```

  

**正确写法：**

```protobuf

service UserService {

  rpc CreateUser(...) returns (...);

  // ✅ 新的 rpc 必须在 service { } 内部

  rpc UpdateGuided(UpdateGuidedRequest) returns (UpdateGuidedResponse);

}

  

// message 可以在 service 外面

message UpdateGuidedRequest { ... }

message UpdateGuidedResponse { ... }

```

  

**核心规则：**

- `service` 定义所有 RPC 方法（接口声明）

- `message` 定义数据结构（可以在 service 外面）

- **rpc 必须在 service { } 内部！**

  

---

  

### 3.2 只定义 message，忘了声明 rpc

  

**你做了：**

```protobuf

message UpdateGuidedRequest { ... }

message UpdateGuidedResponse { ... }

```

  

**但忘了：**

```protobuf

service UserService {

  rpc UpdateGuided(UpdateGuidedRequest) returns (UpdateGuidedResponse);  // ← 忘了这个

}

```

  

**结果：** `pb.go` 里没有 `UpdateGuided` 方法，编译报错 `undefined`。

  

---

  

### 3.3 字段编号规则

  

**你的疑问：** "从 10 要跳到 20 吗？"

  

**答案：** 不需要。Proto 字段编号：

- 不需要连续

- 跳号是为了预留空间（可选）

- 直接用下一个数字（11）完全可以

  

---

  

## 四、Context 传递

  

### 4.1 ctx 应该传还是不传？

  

**你的疑问：** "为什么有的代码传 ctx，有的不传？"

  

**答案：**

  

| 情况 | 原因 |

|------|------|

| 不传 ctx | 早期代码，不规范 |

| 传 ctx | 最佳实践，支持超时/取消 |

  

**最佳实践：全链路传递**

```

Handler(ctx) → Service(ctx) → Repository(ctx) → db.WithContext(ctx)

```

  

**你的项目现状：** 原有代码不传 ctx，为了保持一致性，你也可以不传。

  

---

  

## 五、命名规范

  

### 5.1 Request 参数 vs 状态字段

  

| 场景 | 推荐命名 | 原因 |

|------|----------|------|

| 数据库字段 | `has_guided` | 表示状态"是否已引导" |

| Request 参数 | `guided` | 表示动作"设置为什么值" |

  

**但：** 保持一致性比完美命名更重要。你用 `has_guided` 也没问题。

  

---

  

### 5.2 Proto → Go 命名转换

  

| Proto (snake_case) | Go (PascalCase) |

|--------------------|-----------------|

| `user_id` | `UserId` |

| `has_guided` | `HasGuided` |

  

---

  

## 六、学习建议

  

### 6.1 写代码前

  

1. **看 struct 定义** - 确认字段名

2. **看现有代码** - 保持风格一致

3. **搜索关键词** - 确认命名

  

### 6.2 写代码时

  

1. **一次只改一个文件** - 减少错误

2. **改完立即编译** - `go build ./...`

3. **注意拼写** - 用 IDE 检查

  

### 6.3 Proto 修改流程

  

```

1. 在 service { } 里加 rpc 方法

2. 在 service 外面加 message 定义

3. 运行 protoc 重新生成

4. 实现 Handler 方法

```

  

---

  

## 七、速查卡片

  

### Go 函数定义

```go

func (接收者) 方法名(参数) 返回值 {  // { 必须在这行

    // 实现

}

```

  

### Proto service 结构

```protobuf

service 服务名 {

  rpc 方法名(请求) returns (响应);  // rpc 在这里面

}

  

message 请求 { ... }   // message 在外面

message 响应 { ... }

```

  

### gRPC 调用模式（你的项目）

```go

client, err := p.clientManager.GetXxxServiceClient()

if err != nil { return }

resp, err := client.方法名(ctx, &pb.请求{...})

```

  

---

  

## 八、待加强领域

  

| 领域 | 优先级 | 建议 |

|------|--------|------|

| Go 语法基础 | ⭐⭐⭐ | 多写、多编译、看报错 |

| Protobuf 结构 | ⭐⭐⭐ | 理解 service vs message |

| 项目代码阅读 | ⭐⭐ | 写之前先看现有代码 |

| Context 机制 | ⭐ | 后续重构时学习 |