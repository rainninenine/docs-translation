> ## 文档索引
> 获取完整文档索引：https://docs.osmedeus.org/llms.txt
> 在进一步探索前，请使用此文件发现所有可用页面。

# 常见问题解答

> 常见问题与常见错误

收集了常见问题以及常见错误的解释和解决方法。

## 1. 通用问题


  <details>
<summary>答案</summary>

    Osmedeus 是一个用于安全自动化的工作流引擎。它执行 YAML 定义的工作流，支持多种执行环境（主机、Docker、SSH）、调度和分布式扫描。
  

</details>



  



  <details>
<summary>答案</summary>

    ```bash
    # 从源码构建
    make build

    # 安装到 $GOBIN
    make install

    # 安装安全工具
    osmedeus install binary --all
    ```
  

</details>



  <details>
<summary>答案</summary>

    当然可以。Osmedeus 内置了对 LLM 的支持，你可以在工作流中使用它来生成侦察报告、编写自定义脚本，甚至构建你自己的代理工作流。你可以查看 [LLM 工作流示例](/advanced/llm/) 了解其工作原理。

    请注意，使用 LLM 可能需要你拥有 LLM 提供商的 API 密钥，并且可能会根据你的使用情况产生额外费用。在工作流中使用 LLM 时，请始终监控你的使用情况和成本。

    由于 Osmedeus 是一个编排框架，你可以利用它来协调你自己的自定义 AI/LLM 工具，包括直接在 YAML 工作流中集成 Claude Code 或 OpenCode。例如，你可以设计一个自定义代理，作为定义管道的一部分调用多个工具，并将其无缝插入到你的工作流中。灵活性几乎是无限的。
  

</details>



  <details>
<summary>答案</summary>

    是的，我在 [github.com/osmedeus/osmedeus-skills](https://github.com/osmedeus/osmedeus-skills) 上构建了 [osmedeus-expert](https://github.com/osmedeus/osmedeus-skills) 技能，你可以在你的代理工具中使用它来编写 YAML 工作流、运行 CLI 命令和配置高级功能。
  

</details>


***

## 2. 二进制安装


  



  



  



  


***

## 3. 扫描执行与扫描结果


  



  



  



  



  



  



  



  



  



  



  



  



  



  



  



  



  



  



  



  



  


***

## 4. 工作流


  



  



  



  


***

## 5. API 与服务器


  



  



  



  



  


***

## 6. 调度


  



  


***

## 7. 运行器


  



  



  


***

## 8. 分布式模式


  



  


***

## 9. 故障排除


  



  



  



  



  


***

## 10. 配置


  



  



  


## 11. 清理与卸载


  



  



  

