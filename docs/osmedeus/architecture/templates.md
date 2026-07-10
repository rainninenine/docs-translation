> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 模板引擎体系结构

> 深入探讨 Osmedeus 模板引擎实现

# 模板引擎体系结构

模板引擎为工作流提供 `{{variable}}` 插值。它使用 pongo2 模板库（兼容 Django），并针对高并发场景进行了自定义优化。

## 模板语法

### 标准变量

标准模板变量使用 `{{variable}}` 语法：

```yaml theme={null}
command: "nuclei -l {{Output}}/urls.txt -o {{Output}}/nuclei.json"
```

### 次要变量（Foreach）

Foreach 循环使用 `[[variable]]` 语法以避免冲突：

```yaml theme={null}
step:
  type: bash
  command: "httpx -u [[subdomain]] >> {{Output}}/httpx.txt"
```

### 生成器函数

生成器函数提供动态值：

```yaml theme={null}
exports:
  random_id: "$rand(16)"
  unique_id: "$uuid()"
```

## 引擎类型

模板系统提供两种引擎实现：

| 引擎            | 描述                           | 使用场景         |
| --------------- | ------------------------------ | ---------------- |
| `Engine`        | 标准单线程引擎                 | 简单工作流       |
| `ShardedEngine` | 高性能分片缓存                 | 高并发           |

## TemplateEngine 接口

```go theme={null}
type TemplateEngine interface {
    // Render 使用给定上下文渲染模板字符串
    Render(template string, ctx map[string]any) (string, error)

    // RenderMap 渲染映射中的所有模板值
    RenderMap(m map[string]string, ctx map[string]any) (map[string]string, error)

    // RenderSlice 渲染切片中的所有模板值
    RenderSlice(s []string, ctx map[string]any) ([]string, error)

    // RenderSecondary 使用 [[ ]] 分隔符渲染模板
    RenderSecondary(template string, ctx map[string]any) (string, error)

    // HasSecondaryVariable 检查模板是否包含 [[ ]]
    HasSecondaryVariable(template string) bool

    // ExecuteGenerator 执行生成器函数表达式
    ExecuteGenerator(expr string) (string, error)

    // RegisterGenerator 注册自定义生成器函数
    RegisterGenerator(name string, fn GeneratorFunc)
}
```

## BatchRenderer 接口

对于高吞吐量场景，`BatchRenderer` 接口扩展了 `TemplateEngine`：

```go theme={null}
type BatchRenderer interface {
    TemplateEngine

    // RenderBatch 在单次操作中渲染多个模板
    // 通过分组模板减少锁竞争
    RenderBatch(requests []RenderRequest, ctx map[string]any) (map[string]string, error)
}

type RenderRequest struct {
    Key      string // 标识符（例如字段名）
    Template string // 要渲染的模板字符串
}
```

## ShardedEngine 体系结构

`ShardedEngine` 将模板分布到多个分片以减少锁竞争：

```
+-------------------------------------------------------------+
|                     ShardedEngine                           |
|                                                             |
|   模板哈希 (FNV-1a)                                         |
|        |                                                    |
|        v                                                    |
|   +-----------------------------------------------------+   |
|   |  分片选择: hash & shardMask                          |   |
|   +-----------------------------------------------------+   |
|        |                                                    |
|        +--------+--------+--------+--------+                |
|        v        v        v        v        v                |
|   +--------++--------++--------++--------+...               |
|   |分片 0  ||分片 1  ||分片 2  ||分片 3  |                  |
|   | RWMutex|| RWMutex|| RWMutex|| RWMutex|                  |
|   | LRU    || LRU    || LRU    || LRU    |                  |
|   +--------++--------++--------++--------+                  |
|                                                             |
+-------------------------------------------------------------+
```

### 配置

```go theme={null}
type ShardedEngineConfig struct {
    ShardCount     int  // 分片数量（2 的幂）
    ShardCacheSize int  // 每个分片的 LRU 缓存大小
    EnablePooling  bool // 使用池化上下文映射
}

// 默认值
const DefaultShardCount = 16
const DefaultShardCacheSize = 64
```

### 性能特性

1. **分片缓存**：使用 FNV-1a 哈希将模板分布到各分片，减少锁竞争。

2. **每个分片的 RWMutex**：读密集型工作负载受益于读写锁。

3. **快速路径**：不含 `{{` 的模板直接返回，无需处理。

4. **无锁执行**：编译后的 pongo2 模板是线程安全的；执行在锁外部进行。

5. **上下文池化**：可重用的上下文映射减少 GC 压力。

### 渲染流程

```go theme={null}
func (e *ShardedEngine) Render(template string, ctx map[string]any) (string, error) {
    // 快速路径：无模板变量
    if !strings.Contains(template, "{{") {
        return template, nil
    }

    // 根据模板哈希获取分片
    shard := e.getShard(template)

    // 使用读锁尝试缓存
    shard.mu.RLock()
    tpl, ok := shard.cache.Get(template)
    shard.mu.RUnlock()

    if !ok {
        // 缓存未命中 - 解析并缓存
        shard.mu.Lock()
        // 写锁后双重检查
        tpl, ok = shard.cache.Get(template)
        if !ok {
            tpl, err = pongo2.FromString(template)
            if err != nil {
                shard.mu.Unlock()
                return "", err
            }
            shard.cache.Add(template, tpl)
        }
        shard.mu.Unlock()
    }

    // 在锁外部执行（pongo2 是线程安全的）
    return tpl.Execute(pongo2.Context(ctx))
}
```

## 预编译模板

`PrecompiledRegistry` 存储工作流的预编译模板：

```go theme={null}
type PrecompiledRegistry struct {
    mu        sync.RWMutex
    workflows map[string]*WorkflowTemplates
}

type WorkflowTemplates struct {
    // 键格式："stepName:fieldName"
    Templates map[string]*pongo2.Template
}
```

### Precompiler 接口

```go theme={null}
type Precompiler interface {
    // PrecompileWorkflow 扫描并预编译所有模板
    PrecompileWorkflow(workflowName string, templates map[string]string) error

    // GetPrecompiled 获取预编译模板
    GetPrecompiled(workflowName, key string) any

    // ClearPrecompiled 移除预编译模板
    ClearPrecompiled(workflowName string)
}
```

### 使用

```go theme={null}
// 在工作流加载时
registry := NewPrecompiledRegistry()
templates := map[string]string{
    "scan:command": "nuclei -l {{Output}}/urls.txt",
    "scan:exports:result": "{{Output}}/nuclei.json",
}
registry.PrecompileWorkflow("my-workflow", templates)

// 在执行时
if tpl := registry.GetPrecompiled("my-workflow", "scan:command"); tpl != nil {
    // 使用预编译模板
}
```

## 上下文池化

上下文映射被池化以减少内存分配：

```go theme={null}
var contextPool = sync.Pool{
    New: func() interface{} {
        return make(map[string]any, 32)
    },
}

func GetContext() map[string]any {
    return contextPool.Get().(map[string]any)
}

func PutContext(ctx map[string]any) {
    clear(ctx)
    contextPool.Put(ctx)
}
```

### 规范化布尔值处理

布尔值被规范化以兼容模板：

```go theme={null}
func NormalizeBoolsToPooled(ctx map[string]any) map[string]any {
    pooled := GetContext()
    for k, v := range ctx {
        switch val := v.(type) {
        case bool:
            pooled[k] = val
        case string:
            if val == "true" {
                pooled[k] = true
            } else if val == "false" {
                pooled[k] = false
            } else {
                pooled[k] = v
            }
        default:
            pooled[k] = v
        }
    }
    return pooled
}
```

## 生成器函数

生成器函数产生动态值：

```go theme={null}
type GeneratorFunc func(args ...string) (string, error)

// 内置生成器
var builtinGenerators = map[string]GeneratorFunc{
    "rand": func(args ...string) (string, error) {
        length := 8
        if len(args) > 0 {
            length, _ = strconv.Atoi(args[0])
        }
        return generateRandomString(length), nil
    },
    "uuid": func(args ...string) (string, error) {
        return uuid.New().String(), nil
    },
    "timestamp": func(args ...string) (string, error) {
        return strconv.FormatInt(time.Now().Unix(), 10), nil
    },
}
```

### 使用

```yaml theme={null}
exports:
  random_id: "$rand(16)"
  unique_id: "$uuid()"
  time: "$timestamp()"
```

## 批量渲染

对于高吞吐量场景，批量渲染减少锁获取次数：

```go theme={null}
func (e *ShardedEngine) RenderBatch(requests []RenderRequest, ctx map[string]any) (map[string]string, error) {
    results := make(map[string]string, len(requests))

    // 一次性准备上下文
    processedCtx := NormalizeBoolsToPooled(ctx)
    defer PutContext(processedCtx)

    // 按分片分组
    shardGroups := make(map[uint32][]RenderRequest)
    for _, req := range requests {
        if !strings.Contains(req.Template, "{{") {
            results[req.Key] = req.Template
            continue
        }
        idx := e.getShardIndex(req.Template)
        shardGroups[idx] = append(shardGroups[idx], req)
    }

    // 处理每个分片组
    for idx, reqs := range shardGroups {
        shard := e.shards[idx]
        if err := e.renderShardBatch(shard, reqs, processedCtx, results); err != nil {
            return nil, err
        }
    }

    return results, nil
}
```

## 次要模板渲染

Foreach 循环使用 `[[variable]]` 以避免与预渲染的 `{{variables}}` 冲突：

```go theme={null}
func (e *ShardedEngine) RenderSecondary(template string, ctx map[string]any) (string, error) {
    if !strings.Contains(template, "[[") {
        return template, nil
    }

    // 将 [[ ]] 转换为 {{ }} 供 pongo2 使用
    converted := strings.ReplaceAll(template, "[[", "{{")
    converted = strings.ReplaceAll(converted, "]]", "}}")

    return e.Render(converted, ctx)
}
```

## 性能基准测试

典型性能特征：

| 操作               | 单线程   | 10 并发  | 100 并发 |
| ------------------ | -------- | -------- | -------- |
| 简单渲染           | 500ns    | 600ns    | 800ns    |
| 缓存渲染           | 200ns    | 250ns    | 400ns    |
| 批量渲染（10 个）  | 1.5µs    | 2µs      | 3µs      |
| 预编译             | 150ns    | 200ns    | 350ns    |

## 最佳实践

1. **使用预编译** 用于频繁执行的工作流
2. **启用池化** 用于高并发场景
3. **使用批量渲染** 当使用相同上下文渲染多个模板时
4. **避免不必要的模板** - 仅在需要变量替换时使用 `{{}}`
5. **使用次要语法** `[[]]` 用于 foreach 循环变量，以避免双重渲染问题