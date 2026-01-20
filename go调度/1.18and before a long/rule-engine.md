# 规则引擎实现文档

> 本文档从简单到详细，先伪代码再代码，帮助理解规则引擎的实现思路。

---

## 一、核心概念（一句话版）

| 概念 | 一句话解释 |
|------|-----------|
| **规则 (Rule)** | 一条"如果...就..."的逻辑 |
| **条件 (Condition)** | "如果"的部分，判断任务是否符合 |
| **动作 (Action)** | "就"的部分，对任务做什么操作 |
| **调度器 (Scheduler)** | 定时检查任务，自动触发规则 |

---

## 二、整体流程（最简版）

```
任务发生变化
     ↓
获取所有规则
     ↓
逐条检查：条件是否满足？
     ↓
满足 → 执行动作
```

---

## 三、文件结构

```
task-service/internal/
├── repository/models/
│   └── task_rule_model.go    # 数据模型：规则长什么样
├── repository/
│   └── task_rule_repository.go  # 数据操作：规则怎么存取
└── service/
    ├── task_rule_service.go     # 核心逻辑：规则怎么评估和执行
    └── task_rule_scheduler.go   # 定时调度：自动检查超时
```

---

## 四、数据模型

### 4.1 伪代码

```
规则 {
    名称
    优先级（数字越大越先执行）
    状态（启用/禁用）
    
    流程配置 {
        条件：什么情况下触发
        动作：触发后做什么
    }
    
    行为配置 {
        是否互斥（匹配一条就停）
        超时时间
        是否检查前置任务
    }
}
```

### 4.2 实际代码

```go
// task_rule_model.go

type TaskRule struct {
    ID             uint           `gorm:"primaryKey"`
    RuleName       string         // 规则名称
    RuleType       string         // 规则类型
    TenantID       int64          // 租户ID
    Priority       int32          // 优先级
    Status         int16          // 1启用 0禁用
    FlowConfig     FlowConfig     // 流程配置 (JSON)
    BehaviorConfig BehaviorConfig // 行为配置 (JSON)
}

type FlowConfig struct {
    Conditions *Condition // 条件
    Actions    []*Action  // 动作列表
}

type BehaviorConfig struct {
    IsExclusive           bool  // 互斥
    CheckPrerequisites    bool  // 检查前置任务
    AllowSkipPrerequisite bool  // 允许跳过前置
    TimeoutMinutes        int32 // 超时（分钟）
    NotifyOnFailure       bool  // 失败通知
}
```

---

## 五、条件系统

### 5.1 设计思路

条件是一棵树，支持嵌套组合：

```
         AND
        /   \
    COMPARE  OR
             / \
        COMPARE COMPARE
```

### 5.2 伪代码

```
条件 {
    类型: AND / OR / NOT / COMPARE
    
    如果是 COMPARE:
        字段名（如 task_status）
        操作符（如 等于、大于）
        目标值（如 2）
    
    如果是 AND/OR/NOT:
        子条件列表
}
```

### 5.3 实际代码

```go
type Condition struct {
    Type     ConditionType // AND / OR / NOT / COMPARE
    Field    string        // 字段名（COMPARE用）
    Operator Operator      // 操作符（COMPARE用）
    Value    interface{}   // 目标值
    Children []*Condition  // 子条件（AND/OR/NOT用）
}

// 条件类型
const (
    ConditionTypeAnd     = "AND"     // 所有子条件都要满足
    ConditionTypeOr      = "OR"      // 任一子条件满足
    ConditionTypeNot     = "NOT"     // 取反
    ConditionTypeCompare = "COMPARE" // 比较
)

// 操作符
const (
    OperatorEQ       = "EQ"       // 等于
    OperatorNE       = "NE"       // 不等于
    OperatorGT       = "GT"       // 大于
    OperatorLT       = "LT"       // 小于
    OperatorGTE      = "GTE"      // 大于等于
    OperatorLTE      = "LTE"      // 小于等于
    OperatorIN       = "IN"       // 在数组中
    OperatorContains = "CONTAINS" // 字符串包含
)
```

### 5.4 JSON 示例

```json
{
    "type": "AND",
    "children": [
        {
            "type": "COMPARE",
            "field": "task_status",
            "operator": "EQ",
            "value": 2
        },
        {
            "type": "COMPARE",
            "field": "priority",
            "operator": "GTE",
            "value": 5
        }
    ]
}
```

**含义**：任务状态=2 **并且** 优先级>=5

---

## 六、动作系统

### 6.1 伪代码

```
动作 {
    类型: 更新状态 / 更新字段 / 发通知 / 触发子任务 / 更新父任务
    参数: { ... }
}
```

### 6.2 实际代码

```go
type Action struct {
    Type   ActionType             // 动作类型
    Params map[string]interface{} // 参数
}

const (
    ActionTypeUpdateStatus       = "UPDATE_STATUS"        // 更新状态
    ActionTypeUpdateParentStatus = "UPDATE_PARENT_STATUS" // 更新父任务状态
    ActionTypeUpdateField        = "UPDATE_FIELD"         // 更新字段
    ActionTypeSendNotification   = "SEND_NOTIFICATION"    // 发通知
    ActionTypeTriggerSubtasks    = "TRIGGER_SUBTASKS"     // 触发子任务
)
```

### 6.3 各动作参数

| 动作类型 | 参数 | 说明 |
|---------|------|------|
| UPDATE_STATUS | `{"status": 3}` | 更新任务状态为3 |
| UPDATE_FIELD | `{"field": "remark", "value": "xxx"}` | 更新任意字段 |
| UPDATE_PARENT_STATUS | `{"status": 3}` | 子任务全完成时更新父任务 |
| TRIGGER_SUBTASKS | `{"target_status": 1}` | 把子任务状态改为1 |
| SEND_NOTIFICATION | `{}` | 发送通知（待实现） |

---

## 七、规则评估（核心逻辑）

### 7.1 伪代码

```
函数 EvaluateRules(任务, 租户ID):
    
    // 1. 获取规则
    如果 任务指定了规则:
        规则列表 = 获取指定的规则
    否则:
        规则列表 = 获取租户下所有规则
    
    // 2. 按优先级排序（大的在前）
    排序(规则列表, 按优先级倒序)
    
    // 3. 开启事务
    事务开始:
        遍历 规则列表:
            // 评估条件
            是否匹配 = evaluateCondition(任务, 规则.条件)
            
            如果 不匹配:
                继续下一条
            
            // 检查前置任务
            如果 规则要求检查前置任务:
                如果 前置任务未完成:
                    记录错误，继续下一条
            
            // 执行动作
            遍历 规则.动作列表:
                执行动作(任务, 动作)
            
            // 互斥检查
            如果 规则是互斥的:
                跳出循环（不再评估后续规则）
    事务结束
    
    返回 结果
```

### 7.2 实际代码

```go
// task_rule_service.go

func (s *taskRuleService) EvaluateRules(ctx context.Context, task *models.Task, tenantID int64) (*EvaluationResult, error) {
    result := &EvaluationResult{
        MatchedRules:    make([]*models.TaskRule, 0),
        ExecutedActions: make([]*ActionResult, 0),
        Errors:          make([]string, 0),
    }

    // 1. 获取规则
    var rules []*models.TaskRule
    if len(task.TaskRules) > 0 {
        rules, _ = s.getRulesByIDs(ctx, task.TaskRules)
    } else {
        rules, _ = s.ruleRepo.GetByTenant(ctx, tenantID)
    }

    // 2. 按优先级排序
    sort.Slice(rules, func(i, j int) bool {
        return rules[i].Priority > rules[j].Priority
    })

    // 3. 事务执行
    err := s.db.Transaction(func(tx *gorm.DB) error {
        for _, rule := range rules {
            // 评估条件
            matched, _ := s.evaluateCondition(task, rule.FlowConfig.Conditions)
            if !matched {
                continue
            }

            result.MatchedRules = append(result.MatchedRules, rule)

            // 检查前置任务
            if rule.BehaviorConfig.CheckPrerequisites {
                if !s.checkPrerequisites(ctx, task, rule.BehaviorConfig.AllowSkipPrerequisite) {
                    continue
                }
            }

            // 执行动作
            for _, action := range rule.FlowConfig.Actions {
                actionResult := s.executeAction(ctx, tx, task, action)
                result.ExecutedActions = append(result.ExecutedActions, actionResult)
            }

            // 互斥规则
            if rule.BehaviorConfig.IsExclusive {
                break
            }
        }
        return nil
    })

    return result, err
}
```

---

## 八、条件评估（递归）

### 8.1 伪代码

```
函数 evaluateCondition(任务, 条件):
    
    如果 条件为空:
        返回 true
    
    根据 条件.类型:
        
        AND:
            遍历 子条件:
                如果 evaluateCondition(任务, 子条件) == false:
                    返回 false
            返回 true
        
        OR:
            遍历 子条件:
                如果 evaluateCondition(任务, 子条件) == true:
                    返回 true
            返回 false
        
        NOT:
            返回 !evaluateCondition(任务, 子条件[0])
        
        COMPARE:
            字段值 = 获取任务字段值(条件.字段名)
            返回 比较(字段值, 条件.操作符, 条件.目标值)
```

### 8.2 实际代码

```go
func (s *taskRuleService) evaluateCondition(task *models.Task, cond *models.Condition) (bool, error) {
    if cond == nil {
        return true, nil
    }

    switch cond.Type {
    case models.ConditionTypeAnd:
        for _, child := range cond.Children {
            matched, err := s.evaluateCondition(task, child)
            if err != nil || !matched {
                return false, err
            }
        }
        return true, nil

    case models.ConditionTypeOr:
        for _, child := range cond.Children {
            matched, err := s.evaluateCondition(task, child)
            if err != nil {
                return false, err
            }
            if matched {
                return true, nil
            }
        }
        return false, nil

    case models.ConditionTypeNot:
        matched, err := s.evaluateCondition(task, cond.Children[0])
        return !matched, err

    case models.ConditionTypeCompare:
        return s.evaluateCompare(task, cond)
    }

    return false, fmt.Errorf("未知条件类型")
}
```

---

## 九、字段值获取（反射）

### 9.1 伪代码

```
函数 getFieldValue(任务, 字段名):
    // task_status → TaskStatus
    转换后字段名 = snake转PascalCase(字段名)
    
    // 用反射获取结构体字段值
    返回 任务结构体.字段[转换后字段名]
```

### 9.2 实际代码

```go
func (s *taskRuleService) getFieldValue(task *models.Task, field string) (interface{}, error) {
    v := reflect.ValueOf(task).Elem()
    fieldName := s.toPascalCase(field)  // task_status → TaskStatus
    
    f := v.FieldByName(fieldName)
    if !f.IsValid() {
        return nil, fmt.Errorf("未知字段: %s", field)
    }
    
    return f.Interface(), nil
}

func (s *taskRuleService) toPascalCase(snake string) string {
    parts := strings.Split(snake, "_")
    for i, part := range parts {
        if len(part) > 0 {
            parts[i] = strings.ToUpper(part[:1]) + part[1:]
        }
    }
    return strings.Join(parts, "")
}
```

---

## 十、动作执行

### 10.1 伪代码

```
函数 executeAction(任务, 动作):
    
    根据 动作.类型:
        
        UPDATE_STATUS:
            新状态 = 动作.参数["status"]
            更新数据库: UPDATE task SET task_status = 新状态 WHERE id = 任务ID
        
        UPDATE_FIELD:
            字段名 = 动作.参数["field"]
            字段值 = 动作.参数["value"]
            更新数据库: UPDATE task SET 字段名 = 字段值 WHERE id = 任务ID
        
        UPDATE_PARENT_STATUS:
            如果 任务没有父任务:
                返回
            获取父任务的所有子任务
            如果 所有子任务都完成:
                更新父任务状态
        
        TRIGGER_SUBTASKS:
            获取子任务ID列表
            更新所有子任务状态
```

### 10.2 实际代码

```go
func (s *taskRuleService) executeAction(ctx context.Context, tx *gorm.DB, task *models.Task, action *models.Action) *ActionResult {
    result := &ActionResult{Action: action, Success: false}

    switch action.Type {
    case models.ActionTypeUpdateStatus:
        status := int32(action.Params["status"].(float64))
        err := tx.Model(&models.Task{}).
            Where("id = ?", task.ID).
            Update("task_status", status).Error
        result.Success = (err == nil)

    case models.ActionTypeUpdateField:
        field := action.Params["field"].(string)
        value := action.Params["value"]
        err := tx.Model(&models.Task{}).
            Where("id = ?", task.ID).
            Update(field, value).Error
        result.Success = (err == nil)

    case models.ActionTypeUpdateParentStatus:
        // 检查所有子任务是否完成，是则更新父任务
        s.executeUpdateParentStatus(ctx, tx, task, action)
        result.Success = true

    case models.ActionTypeTriggerSubtasks:
        // 更新所有子任务状态
        s.executeTriggerSubtasks(ctx, tx, task, action)
        result.Success = true
    }

    return result
}
```

---

## 十一、定时调度器

### 11.1 伪代码

```
调度器:
    启动:
        每隔1分钟:
            扫描并评估()
    
    扫描并评估():
        // 1. 获取有超时配置的规则
        规则列表 = 获取启用的规则().过滤(超时时间 > 0)
        
        // 2. 获取进行中的任务
        任务列表 = 查询(状态 = 进行中)
        
        // 3. 评估每个任务
        遍历 任务列表:
            EvaluateRules(任务)
        
        // 4. 检查超时
        遍历 任务列表:
            遍历 规则列表:
                截止时间 = 任务.开始时间 + 规则.超时分钟
                如果 当前时间 > 截止时间:
                    处理超时(任务, 规则)
```

### 11.2 实际代码

```go
// task_rule_scheduler.go

type ruleScheduler struct {
    config      *RuleSchedulerConfig
    ruleService TaskRuleService
    taskRepo    repository.TaskRepository
    ruleRepo    repository.RuleRepository
    stopCh      chan struct{}
}

func (s *ruleScheduler) Start(ctx context.Context) error {
    go s.runLoop(ctx)
    return nil
}

func (s *ruleScheduler) runLoop(ctx context.Context) {
    ticker := time.NewTicker(s.config.ScanInterval)  // 默认1分钟
    defer ticker.Stop()

    s.scanAndEvaluate(ctx)  // 启动时立即执行一次

    for {
        select {
        case <-ticker.C:
            s.scanAndEvaluate(ctx)
        case <-s.stopCh:
            return
        }
    }
}

func (s *ruleScheduler) scanAndEvaluate(ctx context.Context) {
    // 1. 获取有超时配置的规则
    rules, _ := s.getTimeBasedRules(ctx)
    
    // 2. 获取进行中的任务
    tasks, _ := s.taskRepo.ListByStatus(ctx, 2, s.config.BatchSize)
    
    // 3. 评估每个任务
    for _, task := range tasks {
        s.ruleService.EvaluateRules(ctx, task, int64(task.TenantID))
    }
    
    // 4. 检查超时
    s.checkTimeoutTasks(ctx, rules)
}

func (s *ruleScheduler) checkTimeoutTasks(ctx context.Context, rules []*models.TaskRule) {
    now := time.Now()
    tasks, _ := s.taskRepo.ListByStatus(ctx, 2, s.config.BatchSize)

    for _, task := range tasks {
        if task.ActualStartTime == nil {
            continue
        }
        
        for _, rule := range rules {
            timeout := time.Duration(rule.BehaviorConfig.TimeoutMinutes) * time.Minute
            deadline := task.ActualStartTime.Add(timeout)
            
            if now.After(deadline) {
                // 任务超时，执行规则
                s.ruleService.EvaluateRules(ctx, task, int64(task.TenantID))
            }
        }
    }
}
```

---

## 十二、完整示例

### 场景：任务完成时自动更新父任务状态

**规则配置：**
```json
{
    "rule_name": "子任务完成更新父任务",
    "rule_type": "STATUS_CHANGE",
    "priority": 100,
    "status": 1,
    "flow_config": {
        "conditions": {
            "type": "COMPARE",
            "field": "task_status",
            "operator": "EQ",
            "value": 3
        },
        "actions": [
            {
                "type": "UPDATE_PARENT_STATUS",
                "params": { "status": 3 }
            }
        ]
    },
    "behavior_config": {
        "is_exclusive": false,
        "check_prerequisites": false
    }
}
```

**执行流程：**
```
1. 任务状态变为3（已完成）
2. 触发规则评估
3. 条件匹配：task_status == 3 ✓
4. 执行动作：UPDATE_PARENT_STATUS
   - 获取父任务
   - 检查所有兄弟任务是否完成
   - 如果都完成，更新父任务状态为3
```

---

## 十三、扩展指南

### 添加新的操作符

1. 在 `task_rule_model.go` 添加常量：
```go
const OperatorStartsWith Operator = "STARTS_WITH"
```

2. 在 `task_rule_service.go` 的 `compare` 函数添加处理：
```go
case models.OperatorStartsWith:
    return strings.HasPrefix(fmt.Sprintf("%v", fieldValue), fmt.Sprintf("%v", targetValue)), nil
```

### 添加新的动作类型

1. 在 `task_rule_model.go` 添加常量：
```go
const ActionTypeCallWebhook ActionType = "CALL_WEBHOOK"
```

2. 在 `task_rule_service.go` 的 `validateAction` 添加验证：
```go
case models.ActionTypeCallWebhook:
    if _, ok := action.Params["url"]; !ok {
        return fmt.Errorf("CALL_WEBHOOK 需要 url 参数")
    }
```

3. 在 `executeAction` 添加执行逻辑：
```go
case models.ActionTypeCallWebhook:
    url := action.Params["url"].(string)
    // 调用 webhook...
```

---

## 十四、关键设计总结

| 设计点 | 实现方式 | 好处 |
|--------|---------|------|
| 条件嵌套 | 树形结构 + 递归 | 支持复杂逻辑组合 |
| 字段获取 | 反射 | 不用为每个字段写代码 |
| 动作扩展 | switch + 接口 | 容易添加新动作 |
| 优先级 | 排序后顺序执行 | 控制执行顺序 |
| 互斥规则 | IsExclusive 标志 | 避免重复处理 |
| 原子性 | 数据库事务 | 保证一致性 |
| 定时触发 | Scheduler + Ticker | 自动检查超时 |
