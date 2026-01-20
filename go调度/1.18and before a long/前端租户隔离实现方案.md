# 前端租户隔离实现

## 概述

在请求拦截器中统一实现租户隔离，确保非超级管理员只能查看当前租户的数据。

## 核心逻辑

```
用户发起请求
    ↓
请求拦截器检查用户身份
    ↓
┌─────────────────────────────────────┐
│ 超级管理员 (tenant_id = 0)          │ → 不添加参数，可查看所有租户数据
│ 普通用户 (tenant_id > 0)            │ → 自动添加 tenant_id 参数
│ 白名单路径 (登录/注册/字典)          │ → 不添加参数
└─────────────────────────────────────┘
    ↓
发送请求到后端
```

## 修改文件

### `frontend/src/api/request.js`

#### 1. 白名单配置

```javascript
// 不需要添加租户参数的路径
const TENANT_EXCLUDE_PATHS = [
  '/v1/login',
  '/v1/register',
  '/v1/captcha',
  '/v1/refresh-token',
  '/v1/dict'  // 字典数据是公共的
]
```

#### 2. 获取租户信息

```javascript
/**
 * 获取当前用户的租户信息
 * 从 localStorage 读取，避免 Pinia store 循环依赖
 */
function getTenantInfo() {
  try {
    const userInfo = localStorage.getItem('user_info')
    if (userInfo) {
      const parsed = JSON.parse(userInfo)
      return {
        tenantId: parsed.tenant_id,
        isSuperAdmin: parsed.tenant_id === 0
      }
    }
  } catch (e) {
    console.warn('[request.js] 获取租户信息失败:', e)
  }
  return { tenantId: null, isSuperAdmin: false }
}
```

#### 3. 路径检查

```javascript
/**
 * 检查路径是否需要添加租户参数
 */
function shouldAddTenantParam(url) {
  return !TENANT_EXCLUDE_PATHS.some(path => url?.includes(path))
}
```

#### 4. 拦截器中添加租户参数

```javascript
// 请求拦截器
api.interceptors.request.use((config) => {
  // ... 其他逻辑 ...
  
  // 租户隔离：非超级管理员自动添加 tenant_id 参数
  const { tenantId, isSuperAdmin } = getTenantInfo()
  if (!isSuperAdmin && tenantId !== null && shouldAddTenantParam(config.url)) {
    // GET 请求添加到 params
    if (config.method?.toLowerCase() === 'get') {
      config.params = { ...config.params, tenant_id: tenantId }
    }
  }
  
  return config
})
```

## 后端支持

后端 `task_repository.go` 已有租户过滤逻辑：

```go
func (r *taskRepo) ListByTenantID(ctx context.Context, tenantID int) ([]*models.Task, error) {
    var list []*models.Task
    db := r.db.WithContext(ctx)
    // tenant_id = 0 表示平台管理员，查询所有数据
    if tenantID != 0 {
        db = db.Where("tenant_id = ?", tenantID)
    }
    err := db.Where("is_deleted = ?", false).Find(&list).Error
    return list, err
}
```

## 设计优点

| 特性 | 说明 |
|------|------|
| **统一管理** | 所有 GET 请求自动处理，无需在每个 API 方法中手动添加 |
| **不会遗漏** | 新增 API 无需额外处理租户隔离 |
| **白名单机制** | 公共接口（登录、字典等）可灵活排除 |
| **避免循环依赖** | 从 localStorage 读取用户信息，而非 Pinia store |
| **兜底处理** | 获取租户信息失败时不影响正常请求 |

## 扩展：如何添加新的排除路径

如果有新的公共接口不需要租户隔离，在 `TENANT_EXCLUDE_PATHS` 数组中添加：

```javascript
const TENANT_EXCLUDE_PATHS = [
  '/v1/login',
  '/v1/register',
  // ... 其他路径 ...
  '/v1/your-public-api'  // 新增
]
```

## 验证方法

1. **普通用户登录** → 打开浏览器开发者工具 → Network 面板
   - GET 请求应自动带有 `?tenant_id=xxx` 参数
   
2. **超级管理员登录** (tenant_id = 0)
   - GET 请求不应带有 `tenant_id` 参数
   - 可查看所有租户的数据
