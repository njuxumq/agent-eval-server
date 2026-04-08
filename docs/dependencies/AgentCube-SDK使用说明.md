# AgentCube Go SDK 使用说明

> 来源: `/home/xmq/projects/go/AgentCube/go-sdk/USAGE.md`

## 概述

AgentCube 是一个提供隔离沙箱执行环境的服务，支持:
- 沙箱创建与会话管理
- 命令执行 (Shell命令、Python代码)
- 文件操作 (上传、下载、读写)
- 运行时管理 (Shell、Python环境)

## 安装与依赖

### 模块引入

```go
import "github.com/volcano-sh/agentcube/go-sdk/agentcube"
```

### 公司仓库配置

在内网使用时，在 `go.mod` 中增加 `replace`:

```go
replace github.com/volcano-sh/agentcube/go-sdk => git.iflytek.com/AIaaS/AgentCube/go-sdk vX.Y.Z
```

## 环境变量配置

SDK 支持以下环境变量:

| 变量名 | 说明 |
|--------|------|
| `WORKLOAD_MANAGER_URL` / `SANDBOX_WORKLOAD_MANAGER_URL` | 控制面服务地址 |
| `ROUTER_URL` / `SANDBOX_ROUTER_URL` | 数据面服务地址 |
| `APP_ID` / `SANDBOX_APP_ID` | 应用ID |
| `USER_ID` / `SANDBOX_USER_ID` | 用户ID |
| `API_TOKEN` / `SANDBOX_API_TOKEN` | Bearer Token |
| `KONG_TOKEN` / `SANDBOX_KONG_TOKEN` | API网关Token |

**优先级**: `SANDBOX_` 前缀 > 无前缀

## 快速开始

```go
package main

import (
    "context"
    "fmt"
    "os"

    "github.com/volcano-sh/agentcube/go-sdk/agentcube"
)

func main() {
    // 设置环境变量(如果未配置)
    os.Setenv("SANDBOX_WORKLOAD_MANAGER_URL", "http://127.0.0.1:8081")
    os.Setenv("SANDBOX_ROUTER_URL", "http://127.0.0.1:8079")
    os.Setenv("SANDBOX_APP_ID", "demo-app")
    os.Setenv("SANDBOX_USER_ID", "demo-user")

    ctx := context.Background()
    
    // 创建沙箱
    sb, err := agentcube.NewSandbox(ctx, agentcube.SandboxOptions{
        Verbose: true,
    })
    if err != nil {
        panic(err)
    }
    defer sb.Close(ctx)

    // 执行命令
    out, err := sb.ExecuteCommand(ctx, "whoami", agentcube.ExecuteCommandOptions{})
    if err != nil {
        panic(err)
    }
    fmt.Println(out)
}
```

## SandboxOptions 参数详解

### 基础配置

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `Name` | string | `"picod"` | Sandbox模板名称 |
| `Namespace` | string | `"default"` | Kubernetes命名空间 |
| `TTL` | time.Duration | `1h` | 会话过期时间 |

### 服务地址

| 参数 | 说明 |
|------|------|
| `WorkloadManagerURL` | 控制面服务地址，用于创建和管理Sandbox |
| `RouterURL` | 数据面服务地址，用于执行命令和文件操作 |

### 身份认证

| 参数 | 对应请求头 |
|------|-----------|
| `AuthToken` | `Authorization: Bearer <token>` |
| `KongToken` | `x-api-key: <token>` |
| `AppID` | `Ifly-App-ID` |
| `UserID` | `Ifly-User-ID` |

### 会话管理

| 参数 | 说明 |
|------|------|
| `SessionID` | 复用已有会话ID，用于重新连接到现有Sandbox |

### 超时配置

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `ConnectTimeout` | `5s` | 连接超时 |
| `RequestTimeout` | `120s` | 请求读取超时 |

### 调试选项

| 参数 | 说明 |
|------|------|
| `Verbose` | 开启详细日志输出 |

## 核心 API

### 会话管理

| 方法 | 说明 |
|------|------|
| `NewSandbox(ctx, opts)` | 创建新沙箱 |
| `Close(ctx)` | 关闭沙箱 |
| `Info(ctx)` | 获取沙箱信息 |

### 命令执行

| 方法 | 说明 |
|------|------|
| `ExecuteCommand(ctx, command, opts)` | 执行Shell命令 |
| `RunCode(ctx, code, opts)` | 执行代码(Python等) |

### 运行时管理

| 方法 | 说明 |
|------|------|
| `NewShell(ctx, env, workingDir)` | 创建Shell运行时 |
| `NewPython(ctx, env, workingDir)` | 创建Python运行时 |
| `CheckoutExecutor(envID)` | 切换执行器 |
| `StopRuntime(ctx, envID)` | 停止运行时 |

### 文件操作

| 方法 | 说明 |
|------|------|
| `WriteFile(ctx, content, remotePath)` | 写入文件 |
| `UploadFile(ctx, localPath, remotePath)` | 上传本地文件 |
| `DownloadFile(ctx, remotePath, localPath)` | 下载文件到本地 |
| `ReadFile(ctx, remotePath)` | 读取文件内容 |
| `ListFiles(ctx, path)` | 列出目录内容 |
| `CreateDir(ctx, remotePath, mode)` | 创建目录 |
| `GetEntryInfo(ctx, remotePath)` | 获取文件/目录信息 |
| `DownloadURL(ctx, url, path)` | 从URL下载文件 |
| `Expose(ctx, remotePath)` | 暴露文件(生成访问URL) |

## 命令执行超时机制

### 默认超时

- `ConnectTimeout`: `5s`
- `RequestTimeout`: `120s`

### 执行超时

`ExecuteCommand` / `RunCode` 的 `Timeout` 参数是服务端执行超时。

客户端请求超时规则:
- 未显式设置: 请求超时默认 `122s`
- 显式设置超时 `t`: 请求超时为 `t + 2s`

### 长请求超时

| 操作 | 最小超时 |
|------|---------|
| `DownloadURL` | `300s` |
| `Expose` | `60s` |

## 鉴权机制

SDK 自动在请求头中注入:

| 请求头 | 来源 |
|--------|------|
| `sdk-version` | SDK版本号 |
| `Ifly-App-ID` | AppID参数或环境变量 |
| `Ifly-User-ID` | UserID参数或环境变量 |
| `Authorization` | AuthToken参数或环境变量 |
| `x-api-key` | KongToken参数或环境变量 |
| `x-agentcube-session-id` | 会话ID (DataPlane请求) |

## 使用示例

### 上传文件并执行命令

```go
ctx := context.Background()
sb, _ := agentcube.NewSandbox(ctx, agentcube.SandboxOptions{})
defer sb.Close(ctx)

// 上传配置文件
err := sb.UploadFile(ctx, "./local/config.yaml", "/root/workspace/config.yaml")
if err != nil {
    panic(err)
}

// 执行评测命令
out, err := sb.ExecuteCommand(ctx, "python run_eval.py --config /root/workspace/config.yaml", 
    agentcube.ExecuteCommandOptions{
        Timeout: 10 * time.Minute,
    })
if err != nil {
    panic(err)
}
fmt.Println(out)

// 下载结果
err = sb.DownloadFile(ctx, "/root/workspace/output.json", "./local/output.json")
if err != nil {
    panic(err)
}
```

### 复用已有会话

```go
// 首次创建会话
sb1, _ := agentcube.NewSandbox(ctx, agentcube.SandboxOptions{})
sessionID := sb1.SessionID

// 后续复用会话
sb2, _ := agentcube.NewSandbox(ctx, agentcube.SandboxOptions{
    SessionID: sessionID,
})
```

## 高级功能

### Sandbox 迁移 (Migrate)

将 Sandbox 从一个平台迁移到另一个平台:

```go
err := sb.MigrateAndWait(ctx, agentcube.MigrateOptions{
    ToPlatform:     "linux",
    TargetTemplate: "picod3",
    TransferPairs: []controlplane.TransferPair{
        {
            SourcePath: "/root/.openclaw",
            SourceType: "DIR",
            TargetPath: "/root/openclaw",
            TargetType: "DIR",
        },
    },
    Timeout:      5 * time.Minute,
    PollInterval: 3 * time.Second,
})
```

### Sandbox 升级 (Upgrade)

升级 Sandbox 到新的模板版本:

```go
err := sb.UpgradeAndWait(ctx, agentcube.UpgradeOptions{
    TargetTemplate: "picod3",
    TransferPairs: []controlplane.TransferPair{
        {
            SourcePath: "/root/.openclaw",
            SourceType: "DIR",
            TargetPath: "/root/openclaw",
            TargetType: "DIR",
        },
    },
})
```

## 注意事项

1. **会话生命周期**: 创建的 Sandbox 需要显式调用 `Close()` 释放资源
2. **超时处理**: 长时间运行的任务需设置合适的 Timeout
3. **并发安全**: Sandbox 实例不保证并发安全，如需并发执行请创建多个 Sandbox
4. **资源清理**: Sandbox 有 TTL 自动过期，但建议手动 Close 以便及时释放资源