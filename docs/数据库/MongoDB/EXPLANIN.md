explain() 是 MongoDB 中用于分析查询性能的核心工具，它可以展示查询的执行计划、索引使用情况和性能统计信息。

## 基本用法

```bash
// 基本语法
db.collection.find(query).explain(<verbosity>)

// 常用形式
db.collection.find({field: value}).explain("executionStats")
```

## 详细参数说明

### 详细模式 (verbosity)

| 模式                  | 描述           | 适用场景       | 备注 |
|---------------------|--------------|------------|----|
| "queryPlanner"      | 默认模式，只显示查询计划 | 快速检查是否使用索引 |    |
| "executionStats"    | 包含执行统计信息     | 分析查询实际性能   |
| "allPlansExecution" | 包含所有候选计划信息   | 深度优化复杂查询   |    |

### 关键输出字段解析

#### queryPlanner

```bash
{
  "queryPlanner": {
    "namespace": "test.users",  // 集合名称
    "indexFilterSet": false,    // 是否使用索引过滤器
    "winningPlan": {           // 被选中的执行计划
      "stage": "IXSCAN",       // 可能值: COLLSCAN, IXSCAN, FETCH等
      "keyPattern": {age: 1},  // 使用的索引模式
      "indexName": "age_1"     // 使用的索引名称
    },
    "rejectedPlans": []        // 被拒绝的执行计划
  }
}
```

#### executionStats

```bash
{
  "executionStats": {
    "executionSuccess": true,   // 是否执行成功
    "nReturned": 10,           // 返回的文档数量
    "executionTimeMillis": 0,  // 执行时间(毫秒)
    "totalKeysExamined": 10,   // 检查的索引键数量
    "totalDocsExamined": 10,   // 检查的文档数量
    "executionStages": {       // 详细的执行阶段
      "stage": "FETCH",
      "nReturned": 10,
      "docsExamined": 10
    }
  }
}
```

#### 执行计划阶段说明

| 阶段       | 描述       | 优化建议         | 备注 |
|----------|----------|--------------|----|
| COLLSCAN | 全集合扫描    | 考虑添加索引       |    |
| IXSCAN   | 索引扫描     | 良好，检查索引选择性   |    |
| FETCH    | 根据索引获取文档 | 检查是否可以使用覆盖索引 |    |
| SORT     | 内存排序     | 考虑使用索引排序     |    |
| LIMIT    | 限制结果数量   | 通常无需优化       |    |
| SKIP     | 跳过文档     | 考虑分页优化       |    |

## 使用场景示例

```bash
// 检查是否使用了索引
db.users.find({age: {$gt: 25}}).explain()


```
