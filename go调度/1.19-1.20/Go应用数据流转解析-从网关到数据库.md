### 第一棒：前台点餐 (Gateway 接收 HTTP)

**位置**：`backend/services/gateway/internal/proxy/user_proxy.go` **动作**：Gin 收到 HTTP 请求，Proxy 准备打电话给 Auth Service。

```Go
// 1. 注册路由 (在 main.go 或 router.go)
// 告诉 Gin：收到 PUT /users/me/guided 就找 userProxy
userGroup.PUT("/me/guided", userProxy.UpdateCurrentUserGuided)

// 2. Proxy 处理逻辑
func (p *UserProxy) UpdateCurrentUserGuided(c *gin.Context) {
    // [Jump 1 起点]：从 HTTP 上下文拿数据
    userID := c.GetInt64("user_id") // 假设中间件已设置
    
    // [关键连接点]：拿到 gRPC 客户端 (电话机)
    // 这个 client 是在 main.go 里初始化好并注入进来的
    client, err := p.clientManager.GetUserServiceClient()
    if err != nil {
        c.JSON(503, gin.H{"msg": "服务连不上"})
        return
    }

    // [Jump 2 发起]：拨通电话！
    // 注意：这里调用的是 .pb.go 文件里生成的接口
    resp, err := client.UpdateGuided(c.Request.Context(), &pb.UpdateGuidedRequest{
        UserId:    userID,
        HasGuided: true, // 把 JSON 变成 Proto
    })

    // ... 处理 resp
}
```

---

### 📡 第二棒：电话传音 (Proto 契约)

**位置**：`backend/services/auth-service/api/proto/user.proto` **动作**：这是 Gateway 和 Auth Service 的共同语言。如果没有这几行代码，Gateway 没法拨号，Auth Service 没法接听。

```Go
service UserService {
    // [Jump 2 & 3 的桥梁]
    // 只有写了这一行，client.UpdateGuided 才能用
    // 只有写了这一行，UserGrpcHandler 才能去实现它
    rpc UpdateGuided(UpdateGuidedRequest) returns (UpdateGuidedResponse);
}

message UpdateGuidedRequest {
    int64 user_id = 1;
    bool has_guided = 2;
}
```

---

### 👨‍🍳 第三棒：厨房接单 (Auth Service 接听 gRPC)

**位置**：`backend/services/auth-service/internal/handler/user_grpc_handler.go` **动作**：Auth Service 的入口。把二进制 Proto 包拆开，拿出核心数据。

```Go
// 必须挂载在 UserGrpcHandler 上
func (h *UserGrpcHandler) UpdateGuided(ctx context.Context, req *pb.UpdateGuidedRequest) (*pb.UpdateGuidedResponse, error) {
    // [Jump 3 到达]：收到 Proto 请求 req
    
    // [关键解耦动作]：拆包！
    // 把 req.UserId (Proto) -> userID (int64)
    // 把 req.HasGuided (Proto) -> guided (bool)
    
    // [Jump 4 发起]：喊主厨过来处理
    // 注意：h.userService 是接口，具体实现逻辑在 Service 层
    err := h.userService.UpdateGuided(ctx, req.UserId, req.HasGuided)
    
    if err != nil {
        return &pb.UpdateGuidedResponse{Code: -1}, nil
    }
    return &pb.UpdateGuidedResponse{Code: 0}, nil
}
```

---

### 🥘 第四棒：主厨烹饪 (Service 业务逻辑)

**位置**：`backend/services/auth-service/internal/service/user_service.go` **动作**：纯粹的业务逻辑。这里看不到任何 HTTP 或 gRPC 的影子，只有纯粹的 Go 语言类型。

```Go
// 接口定义 (为了解耦)
type UserService interface {
    UpdateGuided(ctx context.Context, id int64, guided bool) error
}

// 具体实现
func (s *userService) UpdateGuided(ctx context.Context, id int64, guided bool) error {
    // [Jump 4 到达]：拿到干净的参数
    
    // 这里可以写业务逻辑，比如：
    // if id <= 0 { return error("非法用户") }
    
    // [Jump 5 发起]：让仓管去改数据库
    // s.repo 是 Repository 接口
    return s.repo.UpdateGuided(ctx, id, guided)
}
```

---

### 📦 第五棒：冰箱拿货 (Repository 数据库操作)

**位置**：`backend/services/auth-service/internal/repository/user_repository.go` **动作**：生成 SQL，操作数据库。
Go
```Go
func (r *userRepository) UpdateGuided(ctx context.Context, id int64, guided bool) error {
    // [Jump 5 到达]：准备写库
    
    // [关键避坑]：使用 Update("列名", 值)
    return r.db.WithContext(ctx).
        Model(&model.User{}).
        Where("id = ?", id).
        Update("has_guided", guided). // SQL: UPDATE users SET has_guided = ? WHERE id = ?
        Error
}
```

---

### 🔗 终极组装：Main.go (把它们串起来)

**位置**：`backend/services/auth-service/cmd/main.go` **动作**：如果说上面是 5 个零件，这里就是把零件组装成机器的地方。**依赖注入**就在这里发生。

```go
func main() {
    // 1. 初始化数据库连接
    db := initDB()

    // 2. 初始化 Repository (第五棒)
    userRepo := repository.NewUserRepository(db)

    // 3. 初始化 Service (第四棒)
    // 把 Repo 塞给 Service，这样 Service 才能调 Repo
    userSvc := service.NewUserService(userRepo)

    // 4. 初始化 Handler (第三棒)
    // 把 Service 塞给 Handler，这样 Handler 才能调 Service
    userHandler := handler.NewUserGrpcHandler(userSvc)

    // 5. 启动 gRPC Server
    server := grpc.NewServer()
    // 把 Handler 注册到 Server 上，这样外部请求才能找到它
    pb.RegisterUserServiceServer(server, userHandler)

    // 启动监听...
    server.Serve(listener)
}
```

### 🧠 核心复习

你看，数据就像水流一样：

1. **JSON** (在 HTTP 层)
    
2. **Proto** (在 Gateway -> Handler 的网络层)
    
3. **int64/bool** (在 Handler -> Service -> Repo 的内部层)
    
4. **SQL** (在 Repo -> DB 层)
    

每一层都在做**数据格式转换**，这就是“跳”的本质！
### 📝 总结：下次遇到“不知道怎么跳”怎么办？

按这个顺序查：

1. **查合同 (Proto)**：`rpc` 定义了吗？`pb.go` 生成了吗？
    
2. **查入口 (Gateway)**：Proxy 里拿到 Client 了吗？调用方法名对吗？
    
3. **查终点 (Handler)**：gRPC Handler 的接收者写对了吗？是 `*UserGrpcHandler` 吗？
    
4. **查逻辑 (Service/Repo)**：参数传对了吗？SQL 拼对了吗？
    

理解了这个**“洋葱模型”**（一层包一层），你就真正掌握了这个框架。