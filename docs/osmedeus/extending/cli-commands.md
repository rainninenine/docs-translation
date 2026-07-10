> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 添加 CLI 命令

向 Osmedeus CLI 添加自定义命令。

## 概述

CLI 命令使用 Cobra 实现，定义在 `pkg/cli/` 目录下。每个命令文件通常包含一个主命令及可选的子命令。

## 命令结构

```go theme={null}
// pkg/cli/mycommand.go

package cli

import (
    "fmt"

    "github.com/spf13/cobra"
)

var (
    // 命令标志
    myFlag string
    myBoolFlag bool
)

var myCmd = &cobra.Command{
    Use:   "mycommand",
    Short: "简短描述",
    Long:  `包含详细信息的详细描述。`,
    RunE: func(cmd *cobra.Command, args []string) error {
        return runMyCommand(args)
    },
}

func init() {
    // 注册到根命令
    rootCmd.AddCommand(myCmd)

    // 添加标志
    myCmd.Flags().StringVarP(&myFlag, "flag", "f", "", "标志描述")
    myCmd.Flags().BoolVarP(&myBoolFlag, "verbose", "v", false, "详细输出")

    // 添加子命令
    myCmd.AddCommand(mySubCmd)
}

func runMyCommand(args []string) error {
    // 加载配置
    cfg, err := loadConfig()
    if err != nil {
        return err
    }

    // 命令逻辑
    fmt.Printf("使用标志 %s 运行 mycommand\n", myFlag)

    return nil
}
```

## 添加命令的步骤

### 1. 创建命令文件

创建 `pkg/cli/mycommand.go`：

```go theme={null}
package cli

import (
    "fmt"

    "github.com/osmedeus/osmedeus-ng/internal/config"
    "github.com/osmedeus/osmedeus-ng/internal/terminal"
    "github.com/spf13/cobra"
)

var (
    targetFlag   string
    outputFlag   string
    verboseFlag  bool
)

var myCmd = &cobra.Command{
    Use:     "mycommand [subcommand]",
    Aliases: []string{"my", "mc"},
    Short:   "我的自定义命令",
    Long: `我的自定义命令执行一些有用的操作。

示例：
  osmedeus mycommand do-something -t example.com
  osmedeus mycommand list`,
    RunE: func(cmd *cobra.Command, args []string) error {
        // 如果没有子命令，显示帮助
        return cmd.Help()
    },
}

var myDoCmd = &cobra.Command{
    Use:   "do-something",
    Short: "执行特定操作",
    RunE: func(cmd *cobra.Command, args []string) error {
        return runDoSomething()
    },
}

var myListCmd = &cobra.Command{
    Use:     "list",
    Aliases: []string{"ls"},
    Short:   "列出项目",
    RunE: func(cmd *cobra.Command, args []string) error {
        return runList()
    },
}

func init() {
    // 注册主命令
    rootCmd.AddCommand(myCmd)

    // 为主命令添加标志（子命令继承）
    myCmd.PersistentFlags().StringVarP(&targetFlag, "target", "t", "", "目标")
    myCmd.PersistentFlags().BoolVarP(&verboseFlag, "verbose", "v", false, "详细输出")

    // 添加子命令特定标志
    myDoCmd.Flags().StringVarP(&outputFlag, "output", "o", "", "输出路径")

    // 注册子命令
    myCmd.AddCommand(myDoCmd)
    myCmd.AddCommand(myListCmd)
}

func runDoSomething() error {
    cfg, err := config.Load(settingsFile)
    if err != nil {
        return fmt.Errorf("加载配置失败: %w", err)
    }

    if targetFlag == "" {
        return fmt.Errorf("需要指定目标")
    }

    terminal.PrintInfo("正在处理目标: %s", targetFlag)

    // 你的逻辑代码...

    terminal.PrintSuccess("完成！")
    return nil
}

func runList() error {
    cfg, err := config.Load(settingsFile)
    if err != nil {
        return err
    }

    items := []string{"item1", "item2", "item3"}

    terminal.PrintTable([]string{"名称", "状态"}, [][]string{
        {"item1", "active"},
        {"item2", "pending"},
        {"item3", "complete"},
    })

    return nil
}
```

### 2. 使用终端辅助函数

`internal/terminal` 包提供了输出格式化功能：

```go theme={null}
import "github.com/osmedeus/osmedeus-ng/internal/terminal"

// 打印消息
terminal.PrintInfo("信息消息")
terminal.PrintSuccess("成功消息")
terminal.PrintWarning("警告消息")
terminal.PrintError("错误消息")

// 格式化打印
terminal.PrintInfo("正在处理 %s，使用 %d 个线程", target, threads)

// 打印表格
terminal.PrintTable(
    []string{"列1", "列2"},
    [][]string{
        {"行1-列1", "行1-列2"},
        {"行2-列1", "行2-列2"},
    },
)

// 带颜色打印
terminal.PrintColored(terminal.Green, "成功！")

// 长时间操作的旋转指示器
spinner := terminal.NewSpinner("加载中...")
spinner.Start()
// ... 执行工作 ...
spinner.Stop()
```

### 3. 访问配置

```go theme={null}
import "github.com/osmedeus/osmedeus-ng/internal/config"

func runMyCommand() error {
    // 加载配置（使用 root.go 中的全局 settingsFile）
    cfg, err := config.Load(settingsFile)
    if err != nil {
        return err
    }

    // 使用配置值
    baseFolder := cfg.BaseFolder
    dbPath := cfg.Database.DBPath

    // 访问环境路径
    workflowsPath := cfg.Environments.Workflows
    binariesPath := cfg.Environments.Binaries

    return nil
}
```

### 4. 处理错误

```go theme={null}
func runMyCommand() error {
    // 返回错误 - Cobra 负责显示
    if targetFlag == "" {
        return fmt.Errorf("需要 --target 参数")
    }

    result, err := doSomething(targetFlag)
    if err != nil {
        return fmt.Errorf("操作失败: %w", err)
    }

    return nil
}
```

### 5. 添加命令帮助

```go theme={null}
var myCmd = &cobra.Command{
    Use:   "mycommand <action>",
    Short: "简短的一行描述",
    Long: `命令的详细描述。

此命令执行 X、Y 和 Z 操作。当你需要...时使用它。

示例：
  # 基本用法
  osmedeus mycommand action -t target

  # 带选项
  osmedeus mycommand action -t target -o output.txt --verbose

  # 多个目标
  osmedeus mycommand action -t target1 -t target2`,
    RunE: func(cmd *cobra.Command, args []string) error {
        return cmd.Help()
    },
}
```

## 示例：导出命令

```go theme={null}
// pkg/cli/export.go

package cli

import (
    "fmt"
    "os"

    "github.com/osmedeus/osmedeus-ng/internal/config"
    "github.com/osmedeus/osmedeus-ng/internal/database"
    "github.com/osmedeus/osmedeus-ng/internal/terminal"
    "github.com/spf13/cobra"
)

var (
    exportWorkspace string
    exportFormat    string
    exportOutput    string
)

var exportCmd = &cobra.Command{
    Use:   "export",
    Short: "从项目空间导出数据",
    Long: `从项目空间导出资产、漏洞或其他数据。

示例：
  osmedeus export assets -w example.com -f json -o assets.json
  osmedeus export vulns -w example.com -f csv`,
}

var exportAssetsCmd = &cobra.Command{
    Use:   "assets",
    Short: "导出资产",
    RunE: func(cmd *cobra.Command, args []string) error {
        return runExportAssets()
    },
}

var exportVulnsCmd = &cobra.Command{
    Use:   "vulns",
    Short: "导出漏洞",
    RunE: func(cmd *cobra.Command, args []string) error {
        return runExportVulns()
    },
}

func init() {
    rootCmd.AddCommand(exportCmd)

    exportCmd.PersistentFlags().StringVarP(&exportWorkspace, "workspace", "w", "", "项目空间名称（必填）")
    exportCmd.PersistentFlags().StringVarP(&exportFormat, "format", "f", "json", "输出格式（json, csv, jsonl）")
    exportCmd.PersistentFlags().StringVarP(&exportOutput, "output", "o", "", "输出文件（为空则输出到标准输出）")

    exportCmd.MarkPersistentFlagRequired("workspace")

    exportCmd.AddCommand(exportAssetsCmd)
    exportCmd.AddCommand(exportVulnsCmd)
}

func runExportAssets() error {
    cfg, err := config.Load(settingsFile)
    if err != nil {
        return err
    }

    db, err := database.Connect(cfg)
    if err != nil {
        return fmt.Errorf("数据库连接失败: %w", err)
    }
    defer db.Close()

    assets, err := db.GetAssetsByWorkspace(exportWorkspace)
    if err != nil {
        return err
    }

    output, err := formatOutput(assets, exportFormat)
    if err != nil {
        return err
    }

    if exportOutput != "" {
        return os.WriteFile(exportOutput, []byte(output), 0644)
    }

    fmt.Println(output)
    return nil
}
```

## 标志类型

```go theme={null}
// 字符串标志
cmd.Flags().StringVarP(&myString, "string", "s", "default", "描述")

// 布尔标志
cmd.Flags().BoolVarP(&myBool, "bool", "b", false, "描述")

// 整数标志
cmd.Flags().IntVarP(&myInt, "count", "c", 10, "描述")

// 字符串切片（可重复）
cmd.Flags().StringSliceVarP(&mySlice, "item", "i", []string{}, "描述")

// 持久标志（子命令继承）
cmd.PersistentFlags().StringVarP(&myPersistent, "global", "g", "", "描述")

// 必填标志
cmd.Flags().StringVarP(&required, "required", "r", "", "描述")
cmd.MarkFlagRequired("required")
```

## 最佳实践

1. **使用子命令** 组织相关操作
2. **提供别名** 方便常用命令
3. **在 Long 描述中编写有用的示例**
4. **尽早验证输入** 在处理之前
5. **使用终端辅助函数** 保持输出一致
6. **使用 context 处理中断**

## 下一步

* [添加 API 端点](api-endpoints.md) - REST 端点
* [添加步骤类型](step-types.md) - 自定义步骤
* [CLI 参考](../cli/run.md) - 现有命令