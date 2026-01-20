

作为后端 Model 层，它起到了一个 **“双向翻译官”** 的作用：

1. **向下（数据库）：** 通过 `gorm:"..."` 标签，告诉程序如何把 Go 语言的结构体存入数据库表。
2. **向上（前端/API）：** 通过 `json:"..."` 标签，告诉程序如何把结构体数据转换成 JSON 格式发给前端。

------

### 1. 这里的映射关系详解

让我们拆解你提供的这行代码：

Go

```
HasGuided bool `gorm:"column:has_guided;default:false" json:"has_guided"`
```

- **Go 语言层面**: 变量名叫 `HasGuided` (大驼峰命名，Go 的公开字段标准)。
- **`gorm:"..."` (数据库映射)**:
  - `column:has_guided`: 告诉 GORM，你在数据库表里找一个叫 `has_guided` 的列，把数据存进去。
  - `default:false`: 如果创建数据时没给这个字段赋值，默认填 `false`。
- **`json:"..."` (前端映射)**:
  - `"has_guided"`: 当后端把这个结构体 `return` 给前端时，JSON 里的 key 叫 `has_guided`（而不是 Go 里的 `HasGuided`）。这是为了符合前后端分离的命名规范（前端通常喜欢小写下划线或小驼峰）。

------

### 2. GORM 常用标签 (Tags) 速查表

你问到了 `default`，其实 GORM 还有很多好用的标签。对于新手，你只需要掌握下面这些最高频的：

| **标签关键词** | **示例写法**        | **作用解释**                                                 |
| -------------- | ------------------- | ------------------------------------------------------------ |
| **column**     | `column:user_email` | **最常用**。指定数据库表中对应的列名。如果不写，GORM 会自动尝试把驼峰转下划线（但建议显式写出来，更清晰）。 |
| **type**       | `type:varchar(100)` | 指定数据库的数据类型。比如 `text`（长文本）、`date`（日期）、`boolean` 等。 |
| **primaryKey** | `primaryKey`        | 标记这个字段是主键（ID）。                                   |
| **default**    | `default:false`     | **默认值**。当插入数据且该字段为空时，使用此值。常见如 `default:0` 或 `default:''`。 |
| **not null**   | `not null`          | **非空约束**。告诉数据库这一列不允许存 `NULL` 值。           |
| **unique**     | `unique`            | **唯一约束**。这一列的值不能重复（例如身份证号、邮箱）。     |
| **index**      | `index`             | 为这一列创建普通索引，加快查询速度。                         |
| **-**          | `gorm:"-"`          | **忽略字段**。非常重要！表示这个字段**只在 Go 代码里用**，不需要存到数据库里（比如由两个字段临时计算出的结果）。 |

------

### 3. 必须要补充的 3 个关键知识点

对于基础薄弱的开发者，这三个点很容易踩坑：

#### A. `json` 标签里的 `omitempty`

在你的代码最后一行有这个：

Go

```
Tenant *Tenant `gorm:"foreignKey:TenantID" json:"tenant,omitempty"`
```

注意那个 **`omitempty`**。

- **含义**：Omit if empty（如果为空则忽略）。
- **作用**：如果 `Tenant` 是 `nil` 或者空值，后端返回给前端的 JSON 数据里，**根本就不会出现 `tenant` 这个字段**。
- **好处**：节省流量，让 JSON 更干净。

#### B. 指针 (`*string`) vs 值类型 (`string`)

你看代码里 `UpdateBy` 是 `*string`，而 `HasGuided` 是 `bool`。

- **使用 `\*string` (指针)**：通常代表这个字段在数据库里**允许为 NULL**。如果指针是 `nil`，数据库里就存 `NULL`。
- **使用 `string` / `bool` (值类型)**：通常代表这个字段**不允许为 NULL**，或者是你希望它总有一个确定的默认值（如空字符串 `""` 或 `false`）。

#### C. 代码写了 `default`，数据库就会自动变吗？

**不会！** 这是一个常见的误区。

- 在 Go 结构体里写 ``default:false``，主要是告诉 GORM 在**插入代码层面**如何处理。
- 如果数据库表已经存在了，且表结构里没有设置默认值，光改 Go 代码通常是不够的。
- **正确做法**：必须配合 SQL 迁移文件（如你文档中的 `030_add_user_guided.sql`），在数据库层面真正执行 `ALTER TABLE ... DEFAULT FALSE`。
# GORM & JSON Struct 标签练习题

> **练习目标**：掌握 Go 语言结构体中 `gorm` 和 `json` 标签的写法，理解字段映射、默认值、忽略字段和空值处理。

## 第一部分：基础映射 (必做)

请为下面的 `Student` 结构体添加 Tag，满足以下需求：

1. **ID**: 设为主键 (`primaryKey`)，JSON 显示为 `id`。
    
2. **Name**: 数据库列名为 `stu_name`，数据类型为 `varchar(50)`，JSON 显示为 `name`。
    
3. **Age**: 数据库列名为 `age`，JSON 显示为 `age`。
    

Go

```
type Student struct {
    ID   int64  // 在此处添加 tag
    Name string // 在此处添加 tag
    Age  int    // 在此处添加 tag
}
```

---

## 第二部分：默认值与约束 (进阶)

请为下面的 `Order` (订单) 结构体添加 Tag，满足以下需求：

1. **OrderID**: 数据库列名 `order_id`，JSON 显示为 `order_id`。
    
2. **Status**: 订单状态。数据库默认值为 `0` (`default:0`)，JSON 显示为 `status`。
    
3. **CreateTime**: 数据库列名 `created_at`，不允许为空 (`not null`)。
    

Go

```
type Order struct {
    OrderID    string // 在此处添加 tag
    Status     int    // 在此处添加 tag
    CreateTime string // 在此处添加 tag
}
```

---

## 第三部分：忽略字段与空值处理 (易错点)

请为下面的 `UserProfile` (用户档案) 结构体添加 Tag，满足以下需求：

1. **Password**: 只需要存入数据库 (列名 `password`)，**绝对不能** 返回给前端（JSON 设为不可见）。
    
2. **Avatar**: 用户头像。如果用户没有上传头像（字段为空字符串），则 JSON 中**不要出现这个字段** (`omitempty`)。
    
3. **TempToken**: 这是一个临时验证码，**不需要存入数据库** (`-`)，但是**需要** 返回给前端 JSON。
    

Go

```
type UserProfile struct {
    Password  string // 在此处添加 tag
    Avatar    string // 在此处添加 tag
    TempToken string // 在此处添加 tag
}
```

---

## 第四部分：综合实战 (模拟真实业务)

请根据描述写出完整的结构体 `Article` (文章)：

- **业务背景**：这是一个博客系统的文章表。
    
- **字段要求**：
    
    1. `Title` (标题): 数据库叫 `title`，JSON 叫 `title`。
        
    2. `ViewCount` (阅读量): 数据库叫 `view_count`，默认是 `0`。
        
    3. `IsPublished` (是否发布): 这是一个布尔值，数据库叫 `is_pub`，默认为 `false`。
        
    4. `Content` (内容): 数据库类型是 `text` (长文本)。
        
    5. `Summary` (摘要): 指针类型 `*string` (因为摘要可能是 NULL)，JSON 如果是 nil 则不显示。
        

Go

```
// 请在此处写出完整的 struct
type Article struct {
    // ...
}
```
## 💡 第一部分答案：基础映射

Go

```go
type Student struct {
    ID   int64  `gorm:"primaryKey" json:"id"`
    Name string `gorm:"column:stu_name;type:varchar(50)" json:"name"`
    Age  int    `gorm:"column:age" json:"age"`
}
```

> **解析**：
>
> - `primaryKey`：告诉 GORM 这是主键 ID。
> - `column:stu_name`：实现了 Go 字段名 `Name` 到数据库列名 `stu_name` 的转换。
> - `type:varchar(50)`：精准控制了数据库字段的长度。

## 💡 第二部分答案：默认值与约束

Go

```go
type Order struct {
    OrderID    string `gorm:"column:order_id" json:"order_id"`
    Status     int    `gorm:"column:status;default:0" json:"status"`
    CreateTime string `gorm:"column:created_at;not null" json:"create_time"`
}
```

> **解析**：
>
> - `default:0`：非常实用，新订单插入时如果没填状态，数据库会自动填 0（例如代表“待支付”）。
> - `not null`：数据库层面的硬性约束，防止脏数据。

## 💡 第三部分答案：忽略字段与空值 (重点)

Go

```go
type UserProfile struct {
    // 1. 密码：JSON 里的 "-" 表示该字段在序列化时被忽略，不会发给前端
    Password  string `gorm:"column:password" json:"-"`
    
    // 2. 头像：omitempty 表示如果是空值，JSON 里直接没有这个 key
    Avatar    string `gorm:"column:avatar" json:"avatar,omitempty"`
    
    // 3. 临时Token：GORM 里的 "-" 表示不需要在数据库建列，只在内存/前端交互用
    TempToken string `gorm:"-" json:"temp_token"`
}
```

> **解析**：
>
> - **GORM 的 `-`** vs **JSON 的 `-`**：这是新手最容易混淆的。
>   - `gorm:"-"` = 数据库里**没有**，内存里有。
>   - `json:"-"` = 数据库里有（看gorm配置），发给前端时**没有**。

## 💡 第四部分答案：综合实战

Go

```go
type Article struct {
    Title       string  `gorm:"column:title" json:"title"`
    ViewCount   int     `gorm:"column:view_count;default:0" json:"view_count"`
    IsPublished bool    `gorm:"column:is_pub;default:false" json:"is_published"`
    Content     string  `gorm:"column:content;type:text" json:"content"`
    
    // 指针类型配合 omitempty
    Summary     *string `gorm:"column:summary" json:"summary,omitempty"`
}
```

> **解析**：
>
> - **布尔值默认值**：`default:false` 常见于草稿状态。
> - **Type Text**：文章内容通常很长，不能用默认的 varchar(255)，所以必须指定 `type:text`。
> - **指针**：`*string` 允许数据库存 `NULL`。如果这里用 `string`，数据库存的是空字符串 `""`，这在业务上是有区别的（NULL 可能代表“未填写”，空字符串可能代表“填了但删除了”）。