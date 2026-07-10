> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 添加函数

注册自定义工具函数，用于工作流中。

## 概述

函数以 Go 语言实现，并通过 `internal/functions/` 目录暴露给 Goja JavaScript VM 运行时。

## 函数注册表

函数在 `internal/functions/goja_runtime.go` 中注册：

```go
func (r *GojaRuntime) registerFunctions() {
    // 文件函数
    r.vm.Set("fileExists", r.fileExists)
    r.vm.Set("fileLength", r.fileLength)
    r.vm.Set("readFile", r.readFile)

    // 字符串函数
    r.vm.Set("trim", r.trim)
    r.vm.Set("split", r.split)

    // 添加你的函数
    r.vm.Set("myNewFunction", r.myNewFunction)
}
```

## 添加函数的步骤

### 1. 实现函数

将函数添加到 `internal/functions/` 中的适当文件：

```go
// internal/functions/util_functions.go

func (r *GojaRuntime) myNewFunction(call goja.FunctionCall) goja.Value {
    // 获取参数
    if len(call.ArgumentList) < 1 {
        return goja.Undefined()
    }

    arg1 := call.Argument(0).String()

    // 可选第二个参数，带默认值
    arg2 := "default"
    if len(call.ArgumentList) > 1 {
        arg2 = call.Argument(1).String()
    }

    // 执行操作
    result, err := doSomething(arg1, arg2)
    if err != nil {
        // 返回 undefined 或错误值
        r.vm.Set("_error", err.Error())
        return goja.Undefined()
    }

    // 将结果转换为 Goja 值
    value, _ := r.vm.ToValue(result)
    return value
}
```

### 2. 注册函数

更新 `internal/functions/goja_runtime.go`：

```go
func (r *GojaRuntime) registerFunctions() {
    // ... 现有注册项 ...

    // 注册你的新函数
    r.vm.Set("myNewFunction", r.myNewFunction)
}
```

### 3. 添加到函数列表

更新 `internal/functions/registry.go` 中的 `ListFunctions()`：

```go
func (r *Registry) ListFunctions() []FunctionInfo {
    return []FunctionInfo{
        // ... 现有函数 ...
        {
            Name:        "myNewFunction",
            Description: "描述其功能",
            Signature:   "myNewFunction(arg1, arg2?)",
            Category:    "util",
        },
    }
}
```

### 4. 编写测试

添加到 `internal/functions/registry_test.go`：

```go
func TestMyNewFunction(t *testing.T) {
    registry := functions.NewRegistry()

    result, err := registry.Execute(`myNewFunction("input", "option")`, nil)

    require.NoError(t, err)
    assert.Equal(t, "expected-output", result)
}

func TestMyNewFunction_DefaultArg(t *testing.T) {
    registry := functions.NewRegistry()

    result, err := registry.Execute(`myNewFunction("input")`, nil)

    require.NoError(t, err)
    assert.Equal(t, "expected-with-default", result)
}
```

## 返回类型

### 字符串

```go
func (r *GojaRuntime) myStringFunc(call goja.FunctionCall) goja.Value {
    result := "hello world"
    value, _ := r.vm.ToValue(result)
    return value
}
```

### 布尔值

```go
func (r *GojaRuntime) myBoolFunc(call goja.FunctionCall) goja.Value {
    result := true
    value, _ := r.vm.ToValue(result)
    return value
}
```

### 数字

```go
func (r *GojaRuntime) myNumberFunc(call goja.FunctionCall) goja.Value {
    result := 42
    value, _ := r.vm.ToValue(result)
    return value
}
```

### 数组

```go
func (r *GojaRuntime) myArrayFunc(call goja.FunctionCall) goja.Value {
    result := []string{"a", "b", "c"}
    value, _ := r.vm.ToValue(result)
    return value
}
```

### 对象/映射

```go
func (r *GojaRuntime) myObjectFunc(call goja.FunctionCall) goja.Value {
    result := map[string]interface{}{
        "key1": "value1",
        "key2": 42,
    }
    value, _ := r.vm.ToValue(result)
    return value
}
```

## 使用上下文

函数可以访问执行上下文：

```go
func (r *GojaRuntime) myContextFunc(call goja.FunctionCall) goja.Value {
    // 获取上下文变量
    ctxValue, err := r.vm.Get("_context")
    if err != nil {
        return goja.Undefined()
    }

    ctx, _ := ctxValue.Export()
    contextMap := ctx.(map[string]interface{})

    target := contextMap["target"].(string)

    // 在函数逻辑中使用上下文
    result := processWithContext(target)

    value, _ := r.vm.ToValue(result)
    return value
}
```

## 示例：哈希函数

```go
// internal/functions/util_functions.go

import (
    "crypto/md5"
    "crypto/sha256"
    "encoding/hex"
)

func (r *GojaRuntime) hash(call goja.FunctionCall) goja.Value {
    if len(call.ArgumentList) < 2 {
        return goja.Undefined()
    }

    input := call.Argument(0).String()
    algorithm := call.Argument(1).String()

    var result string
    switch algorithm {
    case "md5":
        hash := md5.Sum([]byte(input))
        result = hex.EncodeToString(hash[:])
    case "sha256":
        hash := sha256.Sum256([]byte(input))
        result = hex.EncodeToString(hash[:])
    default:
        return goja.Undefined()
    }

    value, _ := r.vm.ToValue(result)
    return value
}
```

注册：

```go
r.vm.Set("hash", r.hash)
```

在工作流中使用：

```yaml
- name: compute-hash
  type: function
  function: hash("{{target}}", "sha256")
  exports:
    target_hash: "{{result}}"
```

## 示例：HTTP 获取函数

```go
// internal/functions/http_functions.go

func (r *GojaRuntime) httpFetch(call goja.FunctionCall) goja.Value {
    if len(call.ArgumentList) < 1 {
        return goja.Undefined()
    }

    url := call.Argument(0).String()

    // 可选方法（默认 GET）
    method := "GET"
    if len(call.ArgumentList) > 1 {
        method = call.Argument(1).String()
    }

    // 可选请求头
    headers := make(map[string]string)
    if len(call.ArgumentList) > 2 {
        headersArg, _ := call.Argument(2).Export()
        if h, ok := headersArg.(map[string]interface{}); ok {
            for k, v := range h {
                headers[k] = fmt.Sprintf("%v", v)
            }
        }
    }

    // 发起请求
    req, err := http.NewRequest(method, url, nil)
    if err != nil {
        return goja.Undefined()
    }

    for k, v := range headers {
        req.Header.Set(k, v)
    }

    client := &http.Client{Timeout: 30 * time.Second}
    resp, err := client.Do(req)
    if err != nil {
        return goja.Undefined()
    }
    defer resp.Body.Close()

    body, _ := io.ReadAll(resp.Body)

    result := map[string]interface{}{
        "status":  resp.StatusCode,
        "body":    string(body),
        "headers": resp.Header,
    }

    value, _ := r.vm.ToValue(result)
    return value
}
```

使用方式：

```yaml
- name: fetch-api
  type: function
  function: httpFetch("https://api.example.com/data", "GET", {"Authorization": "Bearer token"})
  exports:
    api_data: "{{result.body}}"
```

## 最佳实践

1. **验证参数** - 检查参数数量和类型
2. **优雅处理错误** - 错误时返回 undefined
3. **记录函数** - 添加到 ListFunctions()
4. **编写测试** - 覆盖正常路径和边界情况
5. **保持函数纯净** - 最小化副作用
6. **支持可选参数** - 使用合理的默认值

## 函数分类

按类别组织函数：

| 类别     | 文件                    | 函数                                           |
| -------- | ----------------------- | ---------------------------------------------- |
| `file`   | `file_functions.go`     | fileExists, fileLength, readFile, writeFile    |
| `string` | `string_functions.go`   | trim, split, join, replace, contains           |
| `util`   | `util_functions.go`     | log\_info, getEnvVar, exit                     |
| `http`   | `http_functions.go`     | http\_get, http\_post                          |
| `db`     | `db_functions.go`       | db\_select, db\_select\_assets                 |
| `jq`     | `jq.go`                 | jq                                             |

## 下一步

* [添加步骤类型](step-types.md) - 自定义步骤执行器
* [函数参考](../functions/reference.md) - 所有函数
* [函数概述](../functions/overview.md) - 使用指南