# CI/CD Integration

Vigolium 可以无缝集成到您的 CI/CD 流水线中，在每次部署时自动扫描安全漏洞。

## 快速开始

```bash
# 在 CI 中运行快速扫描
vigolium scan -t https://staging.example.com --strategy lite --json -o results.json
```

## 无状态扫描（推荐用于 CI）

无状态模式是 CI/CD 集成的最佳选择——它运行快速、无持久化状态，且易于解析：

```bash
vigolium scan -t https://staging.example.com --stateless --json -o results.json
```

详见 [Stateless Scan](stateless-scan.md)。

## GitHub Actions 示例

```yaml
name: Vigolium Security Scan
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Vigolium Scan
        run: |
          vigolium scan -t ${{ secrets.STAGING_URL }} \
            --strategy lite \
            --stateless \
            --json \
            -o results.json

      - name: Check for Critical Findings
        run: |
          if jq -e '.[] | select(.info.severity == "critical")' results.json > /dev/null; then
            echo "Critical vulnerabilities found!"
            exit 1
          fi
```

## GitLab CI 示例

```yaml
stages:
  - security

vigolium-scan:
  stage: security
  image: vigolium/vigolium:latest
  script:
    - vigolium scan -t ${STAGING_URL} --strategy lite --stateless --json -o results.json
    - |
      if jq -e '.[] | select(.info.severity == "critical" or .info.severity == "high")' results.json > /dev/null; then
        echo "High/Critical vulnerabilities found!"
        exit 1
      fi
  artifacts:
    paths:
      - results.json
```

## Jenkins Pipeline 示例

```groovy
pipeline {
    agent any
    environment {
        STAGING_URL = credentials('staging-url')
    }
    stages {
        stage('Security Scan') {
            steps {
                sh 'vigolium scan -t $STAGING_URL --strategy lite --stateless --json -o results.json'
            }
        }
        stage('Check Results') {
            steps {
                sh '''
                    if jq -e '.[] | select(.info.severity == "critical")' results.json > /dev/null; then
                        echo "Critical vulnerabilities found!"
                        exit 1
                    fi
                '''
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'results.json'
        }
    }
}
```

## 失败阈值

```bash
# 如果发现任何严重或高危漏洞则失败
vigolium scan -t https://example.com --stateless --json -o results.json
jq -e '.[] | select(.info.severity == "critical" or .info.severity == "high")' results.json && exit 1

# 如果发现超过 N 个漏洞则失败
vigolium scan -t https://example.com --stateless --json -o results.json
COUNT=$(jq '[.[] | select(.info.severity == "critical")] | length' results.json)
if [ "$COUNT" -gt 5 ]; then echo "Too many critical findings"; exit 1; fi
```

## 最佳实践

1. **使用 `--strategy lite`** — 在 CI 中快速运行，将深度扫描留给预定的完整扫描
2. **使用 `--stateless`** — 无持久化状态，每次运行都是全新的
3. **输出 JSON** — `--json` 使结果易于解析
4. **设置失败阈值** — 根据严重级别或数量定义流水线何时失败
5. **缓存扫描器** — 在流水线之间缓存 Vigolium 二进制文件以加快运行速度
6. **使用暂存环境** — 扫描暂存环境而非生产环境，除非您有充分的理由

## 延伸阅读

- [Stateless Scan](stateless-scan.md) — 无状态扫描的详细说明
- [Scanning an API](scanning-an-api.md) — API 扫描指南
- [Native Scan Phases](native-scan-phases.md) — 扫描阶段详解
