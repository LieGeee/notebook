# Driver.js 新手引导功能 - 完整实施手册

> 📅 创建日期: 2026-01-19  
> 🎯 功能目标: 使用 driver.js 实现新用户引导流程

---

## 📋 一、修改文件清单 (Checklist)

### 后端修改

| 序号  | 文件路径                                                                   | 操作     | 状态  |
| --- | ---------------------------------------------------------------------- | ------ | --- |
| 1   | `backend/services/auth-service/migrations/030_add_user_guided.sql`     | **新建** | ⬜   |
| 2   | `backend/services/auth-service/internal/repository/model/user.go`      | **修改** | ⬜   |
| 3   | `backend/services/auth-service/api/proto/user.proto`                   | **修改** | ⬜   |
| 4   | `backend/services/auth-service/internal/repository/user_repository.go` | **修改** | ⬜   |
| 5   | `backend/services/auth-service/internal/service/user_service.go`       | **修改** | ⬜   |
| 6   | `backend/services/auth-service/internal/handler/user_grpc_handler.go`  | **修改** | ⬜   |
| 7   | `backend/services/gateway/internal/proxy/user_proxy.go`                | **修改** | ⬜   |

### 前端修改

| 序号 | 文件路径 | 操作 | 状态 |
|------|----------|------|------|
| 8 | `frontend/src/api/user.js` | **修改** | ⬜ |
| 9 | `frontend/src/composables/useGuide.js` | **新建** | ⬜ |
| 10 | `frontend/src/styles/driver.css` | **新建** | ⬜ |
| 11 | `frontend/src/main.js` | **修改** | ⬜ |
| 12 | `frontend/src/views/Home.vue` | **修改** | ⬜ |
| 13 | 侧边栏菜单组件 (需确认具体文件) | **修改** | ⬜ |

---

## 📊 二、数据库字段修改

### 2.1 新增字段说明

| 表名 | 字段名 | 类型 | 默认值 | 说明 |
|------|--------|------|--------|------|
| `sys_users` | `has_guided` | `BOOLEAN` | `FALSE` | 是否已完成新手引导 |

### 2.2 迁移 SQL 文件

**文件:** `backend/services/auth-service/migrations/030_add_user_guided.sql`

```sql
-- ================================
-- 用户表添加引导状态字段
-- ================================

-- 添加字段
ALTER TABLE sys_users 
ADD COLUMN IF NOT EXISTS has_guided BOOLEAN DEFAULT FALSE;

-- 添加字段注释
COMMENT ON COLUMN sys_users.has_guided IS '是否已完成新手引导: FALSE=未完成, TRUE=已完成';

-- 为现有用户设置默认值（可选：让老用户跳过引导）
-- UPDATE sys_users SET has_guided = TRUE WHERE create_time < '2026-01-19';
```

---

## 🔧 三、后端代码修改

### 3.1 Model 层 - user.go

**文件:** `backend/services/auth-service/internal/repository/model/user.go`

**修改内容:** 在 `User` 结构体中添加字段

```go
type User struct {
    // ... 原有字段保持不变 ...
    
    UpdateBy   *string    `gorm:"column:update_by;type:varchar(100)" json:"update_by"`

    // ========== 新增字段 ==========
    // 新手引导标记
    HasGuided bool `gorm:"column:has_guided;default:false" json:"has_guided"`
    // ==============================

    // 关联租户（可选，用于联表查询）
    Tenant *Tenant `gorm:"foreignKey:TenantID" json:"tenant,omitempty"`
}
```

### 3.2 Proto 文件 - user.proto

**文件:** `backend/services/auth-service/api/proto/user.proto`

**修改内容:** 在 `UserInfo` message 中添加字段

```protobuf
message UserInfo {
    // ... 原有字段 ...
    
    // 新增：新手引导状态
    bool has_guided = 20;
}

// 新增：更新引导状态的请求
message UpdateGuidedRequest {
    int64 user_id = 1;
    bool guided = 2;
}

message UpdateGuidedResponse {
    int32 code = 1;
    string message = 2;
}
```
  

### 3.3 Repository 层 - user_repository.go

  

**文件:** `backend/services/auth-service/internal/repository/user_repository.go`

  

**新增方法:**

  

```go

// UpdateGuided 更新用户引导状态

func (r *UserRepository) UpdateGuided(ctx context.Context, userID int64, guided bool) error {

    return r.db.WithContext(ctx).

        Model(&model.User{}).

        Where("id = ?", userID).

        Update("has_guided", guided).Error

}

```

### 3.4 Service 层 - user_service.go

**文件:** `backend/services/auth-service/internal/service/user_service.go`

**新增方法:**

```go

// UpdateGuided 更新用户引导状态

func (s *UserService) UpdateGuided(ctx context.Context, userID int64, guided bool) error {

    return s.repo.UpdateGuided(ctx, userID, guided)

}

```
### 3.5 Handler 层 - user_grpc_handler.go

**文件:** `backend/services/auth-service/internal/handler/user_grpc_handler.go`

**新增 gRPC 方法:**

```go

// UpdateGuided 更新用户引导状态 (gRPC)

func (h *UserGrpcHandler) UpdateGuided(ctx context.Context, req *pb.UpdateGuidedRequest) (*pb.UpdateGuidedResponse, error) {

    err := h.userService.UpdateGuided(ctx, req.UserId, req.Guided)

    if err != nil {

        return &pb.UpdateGuidedResponse{

            Code:    -1,

            Message: "更新引导状态失败: " + err.Error(),

        }, nil

    }

    return &pb.UpdateGuidedResponse{

        Code:    0,

        Message: "ok",

    }, nil

}

```

### 3.6 Gateway 层 - user_proxy.go

**文件:** `backend/services/gateway/internal/proxy/user_proxy.go`

**新增 REST 路由:**

```go

// 注册路由时添加

userGroup.PUT("/me/guided", proxy.UpdateCurrentUserGuided)

  

// Handler 方法

func (p *UserProxy) UpdateCurrentUserGuided(c *gin.Context) {

    // 从 JWT 中获取当前用户 ID

    userID, exists := c.Get("user_id")

    if !exists {

        c.JSON(401, gin.H{"code": -1, "message": "未登录"})

        return

    }

    // 调用 gRPC 服务

    resp, err := p.userClient.UpdateGuided(c.Request.Context(), &pb.UpdateGuidedRequest{

        UserId:  userID.(int64),

        Guided:  true,

    })

    if err != nil {

        c.JSON(500, gin.H{"code": -1, "message": "服务调用失败"})

        return

    }

    c.JSON(200, gin.H{"code": resp.Code, "message": resp.Message})

}

```

---
## 🎨 四、前端代码修改
### 4.1 API 层 - user.js

**文件:** `frontend/src/api/user.js

**新增方法:** 在 `userAPI` 对象中添加

```javascript

/**

 * 更新当前用户的引导状态

 * @returns {Promise} 更新响应

 */

updateGuided: async () => {

  try {

    const response = await api.put('/v1/users/me/guided')

    return response.data

  } catch (error) {

    throw error.response?.data || error

  }

},

```

### 4.1.1 引导步骤配置文件（按角色分组）

**文件:** `frontend/src/composables/guided/guideConfig.js` **(新建)**

> 💡 **设计说明：** 引导步骤按角色配置在前端，后端只负责返回用户角色（已有 `GetUserRoles` API）。后续如需动态配置，可迁移到数据库。

```javascript

/**

 * 引导步骤配置

 * 按角色分组，前端根据用户角色选择对应配置

 */
export const guideConfigs = {
  // ========== 超级管理员/租户管理员 ==========
  super_admin: [

    {

      element: '#nav-organization',

      popover: {

        title: '① 创建组织架构',

        description: '首先，在这里设置公司的组织架构，包括公司、分公司、部门等层级。',

        side: 'right',

        align: 'start'

      }

    },

    {

      element: '#nav-users',

      popover: {

        title: '② 导入用户',

        description: '组织架构创建后，在用户管理中导入员工，并分配对应部门。',

        side: 'right',

        align: 'start'

      }

    },

    {

      element: '#nav-roles',

      popover: {

        title: '③ 权限分配',

        description: '为不同功能负责人分配角色和权限，实现职责分离。',

        side: 'right',

        align: 'start'

      }

    },

    {

      element: '#nav-dashboard',

      popover: {

        title: '④ 数据看板',

        description: '在看板页面，快速了解系统运行状况和关键指标。',

        side: 'right',

        align: 'start'

      }

    },

    {

      element: '#nav-schedule',

      popover: {

        title: '⑤ 甘特图排程',

        description: '通过甘特图直观查看和调整任务时间线。',

        side: 'right',

        align: 'start'

      }

    }

  ],

  

  // 租户管理员（复用超级管理员配置）

  tenant_admin: null, // 使用时取 super_admin

  

  // ========== 部门经理 ==========

  dept_manager: [

    {

      element: '#nav-users',

      popover: {

        title: '① 团队成员',

        description: '查看和管理您部门的成员。',

        side: 'right',

        align: 'start'

      }

    },

    {

      element: '#nav-tasks',

      popover: {

        title: '② 任务分配',

        description: '为团队成员分配工作任务。',

        side: 'right',

        align: 'start'

      }

    },

    {

      element: '#nav-schedule',

      popover: {

        title: '③ 排程管理',

        description: '查看和调整团队排程。',

        side: 'right',

        align: 'start'

      }

    }

  ],

  

  // ========== 普通员工 ==========

  employee: [

    {

      element: '#nav-my-tasks',

      popover: {

        title: '① 我的任务',

        description: '查看分配给您的工作任务。',

        side: 'right',

        align: 'start'

      }

    },

    {

      element: '#nav-my-schedule',

      popover: {

        title: '② 我的排程',

        description: '查看您的工作时间安排。',

        side: 'right',

        align: 'start'

      }

    }

  ]

}

  

/**

 * 根据角色 key 获取引导步骤

 * @param {string} roleKey - 角色标识 (role_key)

 * @returns {Array} 引导步骤配置

 */

export function getGuideStepsByRole(roleKey) {

  // 优先匹配，没有则用 employee 默认

  const steps = guideConfigs[roleKey] || guideConfigs['employee']

  // 处理 null 引用（如 tenant_admin 复用 super_admin）

  return steps || guideConfigs['super_admin']

}

```
---
### 4.2 Composable - useGuide.js (完整代码)

**文件:** `frontend/src/composables/guided/useGuide.js` **(新建)**

> 💡 **改进：** 现在根据用户角色动态获取对应的引导步骤

```javascript

import { ref, onUnmounted } from 'vue'

import { driver } from 'driver.js'

import 'driver.js/dist/driver.css'

import { useUserStore } from '@/stores'

import { userAPI } from '@/api/user'

import { getGuideStepsByRole } from './guideConfig'

  

/**

 * 新手引导 Composable

 * 封装 driver.js，根据用户角色显示对应引导

 */

export function useGuide() {

  const userStore = useUserStore()

  const driverInstance = ref(null)

  const isGuiding = ref(false)

  

  // ===============================

  // 根据用户角色获取引导步骤

  // ===============================

  const getSteps = () => {

    // 从用户信息中获取角色

    const roles = userStore.userInfo?.roles || []

    const roleKey = roles[0]?.role_key || 'employee'

    return getGuideStepsByRole(roleKey)

  }

  

  // ===============================

  // 初始化 Driver 实例

  // ===============================

  const initDriver = () => {

    driverInstance.value = driver({

      showProgress: true,

      animate: true,

      allowClose: true,

      stagePadding: 10,

      stageRadius: 8,

      nextBtnText: '下一步',

      prevBtnText: '上一步',

      doneBtnText: '完成引导',

      progressText: '{{current}} / {{total}}',

      onDestroyStarted: () => {

        handleGuideComplete()

      }

    })

  }

  

  // ===============================

  // 开始引导

  // ===============================

  const startGuide = () => {

    if (!driverInstance.value) {

      initDriver()

    }

    isGuiding.value = true

    const steps = getSteps()  // 动态获取角色对应的步骤

    driverInstance.value.setSteps(steps)

    driverInstance.value.drive()

  }

  

  // ===============================

  // 引导完成处理

  // ===============================

  const handleGuideComplete = async () => {

    isGuiding.value = false

    try {

      await userAPI.updateGuided()

      if (userStore.userInfo) {

        userStore.userInfo.has_guided = true

      }

      console.log('✅ 新手引导完成')

    } catch (error) {

      console.error('❌ 保存引导状态失败:', error)

    }

  }

  

  // ===============================

  // 检查是否需要显示引导

  // ===============================

  const shouldShowGuide = () => {

    return userStore.userInfo?.has_guided !== true

  }

  

  // ===============================

  // 自动启动引导

  // ===============================

  const autoStartGuide = () => {

    if (shouldShowGuide()) {

      setTimeout(() => startGuide(), 800)

    }

  }

  

  // ===============================

  // 销毁实例

  // ===============================

  const destroyDriver = () => {

    if (driverInstance.value) {

      driverInstance.value.destroy()

      driverInstance.value = null

    }

  }

  

  onUnmounted(() => destroyDriver())

  

  return {

    isGuiding,

    startGuide,

    autoStartGuide,

    shouldShowGuide,

    destroyDriver

  }

}

```
### 4.3 自定义样式 - driver.css
**文件:** `frontend/src/styles/driver.css` **(新建)**
```css

/* ================================

   Driver.js 自定义主题样式

   ================================ */

  

/* 弹窗容器 */

.driver-popover {

  background-color: #fff;

  border-radius: 8px;

  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.15);

  max-width: 320px;

}

  

/* 标题 */

.driver-popover-title {

  font-size: 16px;

  font-weight: 600;

  color: #303133;

  margin-bottom: 8px;

}

  

/* 描述文字 */

.driver-popover-description {

  font-size: 14px;

  color: #606266;

  line-height: 1.6;

}

  

/* 进度文字 */

.driver-popover-progress-text {

  color: #909399;

  font-size: 12px;

}

  

/* 按钮通用样式 */

.driver-popover-navigation-btns button {

  border-radius: 4px;

  padding: 8px 16px;

  font-size: 14px;

  cursor: pointer;

  transition: all 0.2s;

}

  

/* 下一步/完成按钮 */

.driver-popover-next-btn {

  background-color: #409eff;

  color: #fff;

  border: none;

}

  

.driver-popover-next-btn:hover {

  background-color: #66b1ff;

}

  

/* 上一步按钮 */

.driver-popover-prev-btn {

  background-color: #f5f7fa;

  color: #606266;

  border: 1px solid #dcdfe6;

}

  

.driver-popover-prev-btn:hover {

  color: #409eff;

  border-color: #c6e2ff;

  background-color: #ecf5ff;

}

  

/* 关闭按钮 */

.driver-popover-close-btn {

  color: #909399;

}

  

.driver-popover-close-btn:hover {

  color: #606266;

}

  

/* 高亮遮罩层 */

.driver-overlay {

  background-color: rgba(0, 0, 0, 0.5);

}

```
### 4.4 引入样式 - main.js
**文件:** `frontend/src/main.js`
**新增内容:**

```javascript

// 引入 driver.js 自定义样式

import '@/styles/driver.css'

```
### 4.5 使用引导 - Home.vue
**文件:** `frontend/src/views/Home.vue`
**修改内容:**

```vue

<script setup>

import { onMounted } from 'vue'

import { useGuide } from '@/composables/useGuide.js'

  

// ... 原有代码 ...

  

// 引入引导功能

const { autoStartGuide, startGuide } = useGuide()

  

onMounted(() => {

  // ... 原有的 onMounted 代码 ...

  // 自动检查并启动新手引导

  autoStartGuide()

})

  

// 手动触发引导（绑定到帮助按钮）

const handleShowGuide = () => {

  startGuide()

}

</script>

  

<template>

  <!-- ... 原有模板 ... -->

  <!-- 帮助按钮（可选，放在合适位置） -->

  <el-tooltip content="查看引导" placement="left">

    <el-button

      @click="handleShowGuide"

      type="primary"

      circle

      class="guide-help-btn"

    >

      <el-icon><QuestionFilled /></el-icon>

    </el-button>

  </el-tooltip>

</template>

  

<style scoped>

.guide-help-btn {

  position: fixed;

  right: 20px;

  bottom: 20px;

  z-index: 100;

  width: 48px;

  height: 48px;

}

</style>

```
### 4.6 侧边栏菜单添加 ID
**文件:** 侧边栏菜单组件（需确认具体路径）
**修改内容:** 为需要引导定位的菜单项添加 `id` 属性
```vue

<!-- 组织管理 -->

<el-menu-item index="/organizations" id="nav-organization">

  <el-icon><OfficeBuilding /></el-icon>

  <span>组织管理</span>

</el-menu-item>

  

<!-- 用户管理 -->

<el-menu-item index="/users" id="nav-users">

  <el-icon><User /></el-icon>

  <span>用户管理</span>

</el-menu-item>

  

<!-- 角色管理 -->

<el-menu-item index="/roles" id="nav-roles">

  <el-icon><Key /></el-icon>

  <span>角色管理</span>

</el-menu-item>

  

<!-- 数据看板 -->

<el-menu-item index="/dashboard" id="nav-dashboard">

  <el-icon><DataBoard /></el-icon>

  <span>数据看板</span>

</el-menu-item>

  

<!-- 排程管理 -->

<el-menu-item index="/schedule" id="nav-schedule">

  <el-icon><Calendar /></el-icon>

  <span>排程管理</span>

</el-menu-item>

```
### 4.7 页面按钮添加 data-guide 属性
**文件:** `frontend/src/views/Organizations.vue`
```vue

<!-- 新增组织按钮 -->

<el-button

  type="primary"

  @click="handleAdd(null)"

  data-guide="add-org-btn"

>

  <el-icon><Plus /></el-icon>

  新增组织

</el-button>

```
**文件:** `frontend/src/views/Users.vue`
```vue

<!-- 导入用户按钮 -->

<el-button

  @click="handleImport"

  data-guide="import-user-btn"

>

  <el-icon><Upload /></el-icon>

  导入用户

</el-button>

```
---
## 🔄 五、数据流转图
### 5.1 整体架构
```

┌─────────────────────────────────────────────────────────────────┐

│                         用户首次登录                              │

└─────────────────────────────────────────────────────────────────┘

                              │

                              ▼

┌─────────────────────────────────────────────────────────────────┐

│  检查 userInfo.has_guided === false ?                           │

│  ├── 是 → 延迟 800ms 后自动启动引导                               │

│  └── 否 → 不显示引导                                             │

└─────────────────────────────────────────────────────────────────┘

                              │

                              ▼

┌─────────────────────────────────────────────────────────────────┐

│                    Driver.js 引导流程                            │

│                                                                 │

│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐     │

│  │①组织管理 │ → │②导入用户 │ → │③权限分配 │ → │④看板/甘特│     │

│  └──────────┘   └──────────┘   └──────────┘   └──────────┘     │

└─────────────────────────────────────────────────────────────────┘

                              │

                              ▼ 点击"完成引导"

┌─────────────────────────────────────────────────────────────────┐

│                    onDestroyStarted 回调                         │

│  1. 调用 API: PUT /v1/users/me/guided                           │

│  2. 更新 userStore.userInfo.has_guided = true                   │

│  3. 同步 localStorage                                           │

└─────────────────────────────────────────────────────────────────┘

```
### 5.2 后端请求链路
```

前端请求                Gateway                  Auth-Service              Database

    │                     │                          │                        │

    │ PUT /v1/users/me/guided                        │                        │

    │────────────────────►│                          │                        │

    │                     │  从 JWT 解析 user_id     │                        │

    │                     │                          │                        │

    │                     │  gRPC: UpdateGuided()    │                        │

    │                     │─────────────────────────►│                        │

    │                     │                          │                        │

    │                     │                          │  UPDATE sys_users      │

    │                     │                          │  SET has_guided=true   │

    │                     │                          │  WHERE id=?            │

    │                     │                          │───────────────────────►│

    │                     │                          │◄───────────────────────│

    │                     │◄─────────────────────────│                        │

    │◄────────────────────│                          │                        │

    │  { code: 0 }        │                          │                        │

```
---
## 🧪 六、测试验证
### 6.1 数据库验证

  

```sql

-- 检查字段是否添加成功

SELECT column_name, data_type, column_default

FROM information_schema.columns

WHERE table_name = 'sys_users' AND column_name = 'has_guided';

  

-- 查看用户引导状态

SELECT id, user_name, has_guided FROM sys_users LIMIT 10;

```

  

### 6.2 API 验证

  

```bash

# 测试更新引导状态接口

curl -X PUT http://localhost:8080/v1/users/me/guided \

  -H "Authorization: Bearer <your_token>"

```

  

### 6.3 前端调试

  

```javascript

// 在浏览器控制台执行，重置引导状态以便测试

const userInfo = JSON.parse(localStorage.getItem('user_info'))

userInfo.has_guided = false

localStorage.setItem('user_info', JSON.stringify(userInfo))

location.reload()

```

  

---

  

## 📝 七、核心代码速查

  

### 7.1 Driver.js 基本用法

  

```javascript

import { driver } from 'driver.js'

  

// 创建实例

const driverObj = driver({

  showProgress: true,

  nextBtnText: '下一步',

  onDestroyStarted: () => { /* 完成回调 */ }

})

  

// 设置步骤

driverObj.setSteps([

  {

    element: '#target-id',

    popover: {

      title: '标题',

      description: '描述',

      side: 'right'  // top/right/bottom/left

    }

  }

])

  

// 启动

driverObj.drive()

  

// 销毁

driverObj.destroy()

```

  

### 7.2 判断是否显示引导

  

```javascript

const shouldShowGuide = () => {

  const userInfo = userStore.userInfo

  if (!userInfo) return false

  return userInfo.has_guided !== true

}

```

  

### 7.3 GORM 更新单字段

  

```go

db.Model(&User{}).Where("id = ?", id).Update("has_guided", true)

```

  

---

  

## ⚠️ 八、注意事项

  

1. **元素定位**: 确保引导的目标元素有唯一的 `id` 或 `data-guide` 属性

2. **延迟启动**: 引导需要等待 DOM 渲染完成，使用 `setTimeout` 延迟

3. **实例销毁**: 组件卸载时必须调用 `destroy()` 防止内存泄漏

4. **错误处理**: API 调用失败不应阻止用户继续使用系统

5. **样式引入**: 必须引入 `driver.js/dist/driver.css` 基础样式

  

---

  

## 🔄 九、扩展功能（后续迭代）

  

- [ ] 分页面引导：不同页面有独立的引导流程

- [ ] 引导重置：设置页添加"重新查看引导"按钮

- [ ] 引导统计：记录用户在哪一步跳过

- [ ] 角色引导：根据用户角色显示不同引导内容