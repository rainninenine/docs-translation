> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 添加 Runner

为工作流命令添加自定义执行环境。

## 概述

Runner 在不同环境中执行 bash 命令。Osmedeus 包含 Host、Docker 和 SSH 三种 Runner。

## Runner 接口

所有 Runner 在 `internal/runner/runner.go` 中实现此接口：

```go
type Runner interface {
    Execute(ctx context.Context, command string) (*CommandResult, error)
    Setup(ctx context.Context) error
    Cleanup(ctx context.Context) error
    Type() core.RunnerType
    IsRemote() bool
}

type CommandResult struct {
    Output   string
    ExitCode int
    Error    error
}
```

## 添加新 Runner 的步骤

### 1. 定义类型常量

在 `internal/core/types.go` 中添加：

```go
type RunnerType string

const (
    RunnerTypeHost   RunnerType = "host"
    RunnerTypeDocker RunnerType = "docker"
    RunnerTypeSSH    RunnerType = "ssh"
    RunnerTypeMyNew  RunnerType = "mynew"  // 添加你的类型
)
```

### 2. 添加配置（如需要）

更新 `internal/core/workflow.go`：

```go
type RunnerConfig struct {
    // 现有字段...

    // MyNew runner 特有
    MyNewOption1 string `yaml:"mynew_option1,omitempty"`
    MyNewOption2 int    `yaml:"mynew_option2,omitempty"`
}
```

### 3. 创建 Runner

创建 `internal/runner/mynew_runner.go`：

```go
package runner

import (
    "context"
    "fmt"

    "github.com/osmedeus/osmedeus-ng/internal/core"
)

type MyNewRunner struct {
    config *core.RunnerConfig
    // 添加任何连接/状态字段
    client *SomeClient
}

func NewMyNewRunner(config *core.RunnerConfig) (*MyNewRunner, error) {
    if config.MyNewOption1 == "" {
        return nil, fmt.Errorf("mynew_option1 是必需的")
    }

    return &MyNewRunner{
        config: config,
    }, nil
}

func (r *MyNewRunner) Setup(ctx context.Context) error {
    // 初始化连接，创建资源等
    client, err := connectToService(r.config.MyNewOption1)
    if err != nil {
        return fmt.Errorf("连接失败: %w", err)
    }
    r.client = client
    return nil
}

func (r *MyNewRunner) Execute(ctx context.Context, command string) (*CommandResult, error) {
    // 检查上下文取消
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }

    // 在你的环境中执行命令
    output, exitCode, err := r.client.RunCommand(ctx, command)
    if err != nil {
        return &CommandResult{
            Output:   output,
            ExitCode: exitCode,
            Error:    err,
        }, nil
    }

    return &CommandResult{
        Output:   output,
        ExitCode: exitCode,
    }, nil
}

func (r *MyNewRunner) Cleanup(ctx context.Context) error {
    // 清理资源
    if r.client != nil {
        return r.client.Close()
    }
    return nil
}

func (r *MyNewRunner) Type() core.RunnerType {
    return core.RunnerTypeMyNew
}

func (r *MyNewRunner) IsRemote() bool {
    return true // 或 false 用于本地执行
}

// 可选：为远程 Runner 实现 CopyFromRemote
func (r *MyNewRunner) CopyFromRemote(ctx context.Context, remotePath, localPath string) error {
    return r.client.DownloadFile(ctx, remotePath, localPath)
}
```

### 4. 在工厂中注册

更新 `internal/runner/runner.go`：

```go
func NewRunnerFromType(
    runnerType core.RunnerType,
    config *core.RunnerConfig,
    binaryPath string,
) (Runner, error) {
    switch runnerType {
    case core.RunnerTypeHost:
        return NewHostRunner()
    case core.RunnerTypeDocker:
        return NewDockerRunner(config)
    case core.RunnerTypeSSH:
        return NewSSHRunner(config, binaryPath)
    case core.RunnerTypeMyNew:
        return NewMyNewRunner(config)
    default:
        return nil, fmt.Errorf("未知的 runner 类型: %s", runnerType)
    }
}
```

### 5. 添加验证

更新 `internal/parser/validator.go`：

```go
func (v *Validator) validateRunnerConfig(runnerType core.RunnerType, config *core.RunnerConfig) error {
    switch runnerType {
    case core.RunnerTypeMyNew:
        if config == nil {
            return fmt.Errorf("mynew runner 需要 runner_config")
        }
        if config.MyNewOption1 == "" {
            return fmt.Errorf("mynew_option1 是必需的")
        }
    }
    return nil
}
```

### 6. 编写测试

创建 `internal/runner/mynew_runner_test.go`：

```go
package runner

import (
    "context"
    "testing"

    "github.com/osmedeus/osmedeus-ng/internal/core"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestMyNewRunner_Execute(t *testing.T) {
    config := &core.RunnerConfig{
        MyNewOption1: "test-value",
    }

    runner, err := NewMyNewRunner(config)
    require.NoError(t, err)

    ctx := context.Background()
    err = runner.Setup(ctx)
    require.NoError(t, err)
    defer runner.Cleanup(ctx)

    result, err := runner.Execute(ctx, "echo hello")
    require.NoError(t, err)
    assert.Contains(t, result.Output, "hello")
    assert.Equal(t, 0, result.ExitCode)
}
```

## 示例：Kubernetes Runner

```go
// internal/runner/k8s_runner.go

type K8sRunner struct {
    config    *core.RunnerConfig
    clientset *kubernetes.Clientset
    pod       *v1.Pod
}

func NewK8sRunner(config *core.RunnerConfig) (*K8sRunner, error) {
    kubeconfig, err := clientcmd.BuildConfigFromFlags("", config.KubeConfigPath)
    if err != nil {
        return nil, err
    }

    clientset, err := kubernetes.NewForConfig(kubeconfig)
    if err != nil {
        return nil, err
    }

    return &K8sRunner{
        config:    config,
        clientset: clientset,
    }, nil
}

func (r *K8sRunner) Setup(ctx context.Context) error {
    // 创建用于执行的 Pod
    pod := &v1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            Name:      fmt.Sprintf("osmedeus-%s", uuid.New().String()[:8]),
            Namespace: r.config.Namespace,
        },
        Spec: v1.PodSpec{
            Containers: []v1.Container{{
                Name:    "runner",
                Image:   r.config.Image,
                Command: []string{"sleep", "infinity"},
            }},
            RestartPolicy: v1.RestartPolicyNever,
        },
    }

    created, err := r.clientset.CoreV1().Pods(r.config.Namespace).Create(ctx, pod, metav1.CreateOptions{})
    if err != nil {
        return err
    }
    r.pod = created

    // 等待 Pod 就绪
    return r.waitForPod(ctx)
}

func (r *K8sRunner) Execute(ctx context.Context, command string) (*CommandResult, error) {
    req := r.clientset.CoreV1().RESTClient().Post().
        Resource("pods").
        Name(r.pod.Name).
        Namespace(r.config.Namespace).
        SubResource("exec").
        VersionedParams(&v1.PodExecOptions{
            Container: "runner",
            Command:   []string{"sh", "-c", command},
            Stdout:    true,
            Stderr:    true,
        }, scheme.ParameterCodec)

    exec, err := remotecommand.NewSPDYExecutor(r.config, "POST", req.URL())
    if err != nil {
        return nil, err
    }

    var stdout, stderr bytes.Buffer
    err = exec.StreamWithContext(ctx, remotecommand.StreamOptions{
        Stdout: &stdout,
        Stderr: &stderr,
    })

    return &CommandResult{
        Output: stdout.String() + stderr.String(),
        // 如果可用，从错误中提取退出码
    }, err
}

func (r *K8sRunner) Cleanup(ctx context.Context) error {
    if r.pod != nil {
        return r.clientset.CoreV1().Pods(r.config.Namespace).Delete(ctx, r.pod.Name, metav1.DeleteOptions{})
    }
    return nil
}
```

在工作流中的使用：

```yaml
kind: module
name: k8s-scan
runner: k8s
runner_config:
  kubeconfig_path: ~/.kube/config
  namespace: osmedeus
  image: alpine:latest

steps:
  - name: scan
    type: bash
    command: nmap -sV {{target}}
```

## 最佳实践

1. **实现上下文取消** - 检查 `ctx.Done()` 以实现优雅关闭
2. **清理资源** - 始终在 `Cleanup()` 方法中进行清理
3. **处理重连** - 对于远程 Runner，处理连接断开
4. **记录操作** - 使用结构化日志进行调试
5. **验证配置** - 尽早检查必填字段
6. **支持文件传输** - 如适用，实现 `CopyFromRemote`

## 下一步

* [添加步骤类型](step-types.md) - 自定义步骤执行器
* [添加函数](functions.md) - 工具函数
* [Runner 概念](../concepts/runners.md) - Runner 概述