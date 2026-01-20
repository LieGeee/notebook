# 📘 API 设计逻辑：动作与状态的命名艺术 - 深度学习指南

## 1. 🧠 底层逻辑深度拆解 (Deep Dive)

这一部分解释“为什么要这么做”，帮助你建立架构思维。

### 	1.基础例子：单纯改个名 (就是你现在的例子)

​	这是最简单的情况。前端叫 `guided`（为了简便），数据库叫 `has_guided`（为了严谨）。

- **外部 (Proto Request)**: 只有 `guided`。
- **内部 (DB Model)**: 只有 `has_guided`。
- **中间人 (Handler)**: 做了一次“赋值搬运”。

Go

```go
// 1. 接收请求 (Proto)
// 前端发来: { "guided": true }
func (h *UserHandler) UpdateGuided(req *pb.UpdateGuidedRequest) {

    // 2. 准备数据 (Model)
    user := &model.User{}

    // 3. 【这就是映射！】(Mapping)
    // 翻译官说：外界的 guided，就是内部的 HasGuided
    user.HasGuided = req.Guided 

    // 4. 存入数据库
    h.repo.Save(user)
}
```

------

### 2. 进阶例子：数据形态转换 (逻辑加工)

这个例子最能体现“翻译官”的重要性。比如**修改密码**。

- **Proto**: 用户传的是明文密码 `"123456"`。
- **DB**: 数据库绝对不能存明文，必须存加密后的哈希值 `"e10adc3949ba..."`。
- **如果强制一样**：那数据库就得存明文，或者前端得自己加密（不安全）。

Go

```go
// 1. 外部请求 (Proto)
// message UpdatePwdRequest { string password = 1; }
// 前端发来: "123456"

func (s *UserService) UpdatePassword(req *pb.UpdatePwdRequest) {

    // 2. 【这就是映射 + 加工！】
    // 不能直接划等号！中间层必须进行“加密逻辑”转换
    // 外部的 "password" -> 内部的 "password_hash"
    encryptedPwd := md5.Encrypt(req.Password) 

    // 3. 存入数据库 (Model)
    user := &model.User{
        PasswordHash: encryptedPwd, // 数据库里只认 Hash
    }
    
    s.repo.Update(user)
}
```

------

### 3. 结构例子：拆分与合并 (结构不对等)

有时候，外部给的一个字段，内部要拆成两个字段存；或者反过来。比如**全名**。

- **Proto (Request)**: `full_name` = "张三" (前端只给一个框)。
- **DB (Model)**: 为了分析数据，拆成了 `first_name`="三", `last_name`="张"。

Go

```go
// 1. 外部请求 (Proto)
// message UpdateNameRequest { string full_name = 1; }
// 前端发来: "张三"

func (s *UserService) UpdateName(req *pb.UpdateNameRequest) {

    // 2. 【这就是映射 + 拆解！】
    // 翻译官负责把一个字段拆成两个
    names := strings.Split(req.FullName, "") // 简单模拟拆分
    firstName := names[1]
    lastName := names[0]

    // 3. 存入数据库 (Model)
    user := &model.User{
        FirstName: firstName, // 存 "三"
        LastName:  lastName,  // 存 "张"
    }
    // 此时数据库根本不知道有一个叫 full_name 的东西
    s.repo.Update(user)
}
```

------

### 💡 总结 (理解核心)

**中间层 (Handler/Service)** 就像一个 **海关**：

1. **Request (外部旅客)**：带的行李千奇百怪（`guided`, 明文密码, 全名）。
2. **Mapping (海关检查/打包)**：
   - 把 `guided` 改标签贴成 `has_guided` (改名)。
   - 把 `明文密码` 粉碎做成 `Hash` (加工)。
   - 把 `全名` 拆开分装到不同箱子 (重组)。
3. **Model (内部仓库)**：只接收海关处理好、符合仓库规范的标准货物。

### A. 编程范式的转换：RPC (远程过程调用) vs. 数据存储

- **RPC 的本质是“函数调用”**： gRPC 的设计哲学是 Remote Procedure Call。当你发送一个 `UpdateGuidedRequest` 时，你实际上是在远程调用一个函数。
  - 函数签名逻辑：`func Update(guided bool)`。在这里，参数 `guided` 是一个输入变量。在函数式编程或命令式编程中，输入参数通常代表“我要赋予的值”。
- **数据库的本质是“资源快照”**： `UserInfo` 代表的是数据库里的一行记录，是一个静态的快照。
  - 对象属性逻辑：`user.has_guided`。这是一个属性，用来描述这个对象当前的特征。在面向对象编程中，布尔属性通常用 `is/has` 开头来表示状态。

### B. 语义语言学：祈使句 vs. 陈述句

- **Request 是祈使句 (Imperative)**： 它代表**写模型 (Write Model)**。你在对系统下达指令。
  - 语义：“把这个开关设为 `guided` (true/false)。” 这里强调的是动作的参数。
- **Model 是陈述句 (Declarative)**： 它代表**读模型 (Read Model)**。系统在向你陈述事实。
  - 语义：“这个用户**已经** (has) 完成引导了。” 这里强调的是客观事实。

### C. 解耦与映射 (Decoupling & Mapping)

- 许多新手认为 API 定义（Proto）必须和数据库定义（SQL/Model）一模一样。这是错误的。
- **中间层的作用**：后端代码（Handler/Service）存在的意义之一，就是作为“翻译官”，将“外部的指令参数”翻译成“内部的存储状态”。这就是文档中提到的“手动映射”过程。

------

## 2. ❓ 核心自问 (Questions to Ask)

在设计或阅读代码时，问自己这 3 个问题，就能瞬间厘清思路：

1. **我现在是在“下命令”还是在“看数据”？**
   - 下命令 -> 用动词/参数名 (Request)。
   - 看数据 -> 用形容词/状态名 (Model)。
2. **如果把这个 Request 变成一个本地函数，它长什么样？**
   - 是 `SetLocked(true)` (好) 还是 `SetIsLocked(true)` (啰嗦)？
3. **Handler 层有没有做正确的“赋值映射”？**
   - 检查代码里是否有 `model.Field = req.Field` 的操作。

------

## 3. 🎯 重点分层 (Focus Areas)

### 💡 需要深度理解的 (Understand)

1. **语义区分**：Request 是“写模型/动作参数”，Model 是“读模型/状态描述”。
2. **映射机制**：名字不同不是魔法，是因为在 Go 代码的 Handler 层显式地写了 `user.HasGuided = req.Guided`。
3. **gRPC vs REST 的区别**：gRPC 更像函数调用，所以参数名简洁；REST 有时为了 CRUD 方便，会牺牲语义追求字段名一致。

### 💾 需要死记硬背的 (Memorize)

1. **前缀惯例**：
   - Model/DB 字段：必须带 `is_` 或 `has_` (如 `is_active`, `has_verified`)。
   - Request 参数：通常**不带**前缀 (如 `active`, `verified`)。
2. **高频对照表**：
   - `guided` <-> `has_guided`
   - `enable` <-> `is_active`
   - `lock` <-> `is_locked`
   - `show` <-> `is_visible`
   - `deleted` <-> `is_deleted`

------

## 4. ✍️ 完形填空 (Cloze Test)

请根据笔记内容填空，检测记忆效果：

在 gRPC 接口设计中，我们经常发现 Request 和 Model 的字段命名不一致。 `UserInfo` 中的 `has_guided` 属于 **(1) ______** 模型，描述的是用户当前的 **(2) ______**，因此通常带有 `is_` 或 `has_` 前缀。 而 `UpdateGuidedRequest` 中的 `guided` 属于 **(3) ______** 模型，它更像是一个函数调用的 **(4) ______**，强调的是“我要设置的值”。 这两个字段之所以能对应上，全靠后端的 **(5) ______** 层进行手动 **(6) ______**，将请求参数赋值给数据库模型。

------

## 5. 🛠 实战练习题 (Exercises)

### 习题 1：命名转化 (Naming Conversion)

**场景**：你要设计一个“冻结用户”的功能。

1. **数据库字段 (Model)**：用户当前是否被冻结？请命名：`__________`
2. **请求参数 (Request)**：管理员要把用户冻结（true）或解冻（false）。请命名：`__________`

### 习题 2：代码纠错 (Code Debugging)

**场景**：Review 别人的代码，发现 Handler 层报错，请修正。

**Proto 定义**:

Protocol Buffers

```
message SetVisibleRequest { bool show = 1; }
```

**Model 定义**:

Go

```
type Product struct { IsVisible bool }
```

**错误的 Handler 代码**:

Go

```
func SetVisible(req *SetVisibleRequest) {
    product := &Product{}
    // 错误：找不到字段
    product.IsVisible = req.IsVisible 
}
```

**请写出正确的赋值代码**： `__________________________________________________`

### 习题 3：判断题 (True or False)

1. ( ) 在 gRPC 中，Proto 里的字段名必须和数据库表的列名完全一致，否则程序会运行失败。
2. ( ) `has_guided` 这种命名方式属于“读模型”，用于描述客观状态。
3. ( ) 为了保持 RESTful API 的 CRUD 简便性，有时候我们会打破语义区分，强制让 Request 和 Model 字段名一致。

------

### ✅ 答案区 (Key)

**完形填空答案**： (1) 读 (Read) (2) 状态 (State) (3) 写 (Write) (4) 参数 (Parameter) (5) Handler (6) 映射 (Mapping)

**习题 1 答案**：

1. `is_frozen` (或 `is_locked`)
2. `frozen` (或 `lock`)

**习题 2 答案**： `product.IsVisible = req.Show` (因为 Proto 里定义的参数名叫 `show`，对应 Model 的 `IsVisible`)

**习题 3 答案**：

1. **错** (Proto 和 DB 是解耦的，靠代码映射)。
2. **对**。
3. **对** (特殊情况下的权衡)。