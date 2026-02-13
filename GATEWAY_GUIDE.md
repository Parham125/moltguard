# MoltGuard Gateway - 本地 Prompt 脱敏指南

## 功能概述

MoltGuard Gateway 是一个**本地 HTTP 代理**，在你的 prompt 发送给大模型之前，自动脱敏敏感信息（银行卡号、邮箱、密码、API key 等），然后在大模型返回后自动还原。

### 核心原理

```
用户输入："我的卡号是 6222021234567890，帮我订酒店"
    ↓
Gateway 脱敏：→ "我的卡号是 __bank_card_1__，帮我订酒店"
    ↓
发送给 LLM（Claude/GPT/Kimi/等）
    ↓
LLM 返回："好的，我用 __bank_card_1__ 帮您预订"
    ↓
Gateway 还原：→ "好的，我用 6222021234567890 帮您预订"
    ↓
本地执行工具调用（填入真实卡号）
```

## 支持的敏感数据类型

| 类型 | 占位符示例 | 匹配内容 |
|------|-----------|---------|
| 邮箱 | `__email_1__` | user@example.com |
| 银行卡 | `__bank_card_1__` | 6222021234567890 (16-19位) |
| 信用卡 | `__credit_card_1__` | 1234-5678-9012-3456 |
| 手机号 | `__phone_1__` | +86-138-1234-5678, (555) 123-4567 |
| API Key/密钥 | `__secret_1__` | sk-1234..., ghp_abc..., Bearer tokens |
| IP 地址 | `__ip_1__` | 192.168.1.1 |
| SSN | `__ssn_1__` | 123-45-6789 |
| IBAN | `__iban_1__` | GB82WEST12345698765432 |
| URL | `__url_1__` | https://example.com/api |

## 安装和启动

### 方式一：通过 MoltGuard Plugin 自动启动（推荐）

1. **安装插件**（如果尚未安装）：
   ```bash
   openclaw plugins install @openguardrails/moltguard
   ```

2. **启用 Gateway 功能**：

   编辑 `~/.openclaw/openclaw.json`:
   ```json
   {
     "plugins": {
       "entries": {
         "moltguard": {
           "config": {
             "sanitizePrompt": true,       // 启用 prompt 脱敏
             "gatewayPort": 8900,          // Gateway 端口
             "gatewayAutoStart": true      // 自动启动
           }
         }
       }
     }
   }
   ```

3. **重启 OpenClaw**：
   ```bash
   openclaw gateway restart
   ```

   Gateway 会自动启动在 `http://127.0.0.1:8900`

### 方式二：独立运行 Gateway

```bash
# 设置环境变量
export ANTHROPIC_API_KEY="sk-ant-..."
export OPENAI_API_KEY="sk-..."

# 启动 gateway
npx moltguard-gateway

# 或指定端口
MOLTGUARD_GATEWAY_PORT=9000 npx moltguard-gateway
```

## 配置 OpenClaw 使用 Gateway

### Anthropic (Claude) 示例

**原始配置**（直连 Anthropic）：
```json
{
  "models": {
    "providers": {
      "anthropic": {
        "baseUrl": "https://api.anthropic.com",
        "api": "anthropic-messages",
        "apiKey": "${ANTHROPIC_API_KEY}"
      }
    }
  }
}
```

**修改为使用 Gateway**：
```json
{
  "models": {
    "providers": {
      "anthropic-protected": {
        "baseUrl": "http://127.0.0.1:8900",        // 指向 Gateway
        "api": "anthropic-messages",                // 保持协议不变
        "apiKey": "${ANTHROPIC_API_KEY}",
        "models": [
          {
            "id": "claude-sonnet-4-20250514",
            "name": "Claude Sonnet (Protected)",
            "input": ["text", "image"],
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic-protected/claude-sonnet-4-20250514"
      }
    }
  }
}
```

### OpenAI (GPT) 示例

```json
{
  "models": {
    "providers": {
      "openai-protected": {
        "baseUrl": "http://127.0.0.1:8900",        // 指向 Gateway
        "api": "openai-completions",                // 保持协议不变
        "apiKey": "${OPENAI_API_KEY}",
        "models": [
          {
            "id": "gpt-4o",
            "name": "GPT-4o (Protected)",
            "input": ["text", "image"],
            "contextWindow": 128000,
            "maxTokens": 4096
          }
        ]
      }
    }
  }
}
```

### Kimi (月之暗面) 示例

```json
{
  "models": {
    "providers": {
      "kimi-protected": {
        "baseUrl": "http://127.0.0.1:8900",        // 指向 Gateway
        "api": "openai-completions",                // Kimi 兼容 OpenAI 协议
        "apiKey": "${KIMI_API_KEY}",
        "models": [
          {
            "id": "moonshot-v1-128k",
            "name": "Kimi (Protected)",
            "input": ["text"],
            "contextWindow": 128000,
            "maxTokens": 4096
          }
        ]
      }
    }
  }
}
```

## Gateway 配置文件

如果需要更精细的控制，可以创建 `~/.moltguard/gateway.json`:

```json
{
  "port": 8900,
  "backends": {
    "anthropic": {
      "baseUrl": "https://api.anthropic.com",
      "apiKey": "sk-ant-..."
    },
    "openai": {
      "baseUrl": "https://api.openai.com",
      "apiKey": "sk-..."
    },
    "kimi": {
      "baseUrl": "https://api.moonshot.cn",
      "apiKey": "sk-..."
    }
  }
}
```

**注意**：配置文件优先级低于环境变量。如果同时设置，环境变量会覆盖配置文件。

## 使用 Gateway 管理命令

在 OpenClaw 中使用以下命令管理 Gateway：

### `/mg_status` - 查看状态

```
/mg_status
```

返回：
- Gateway 是否运行
- 端口和地址
- 配置示例

### `/mg_start` - 启动 Gateway

```
/mg_start
```

### `/mg_stop` - 停止 Gateway

```
/mg_stop
```

### `/mg_restart` - 重启 Gateway

```
/mg_restart
```

## 验证 Gateway 是否工作

### 方法一：健康检查

```bash
curl http://127.0.0.1:8900/health
```

应该返回：
```json
{"status":"ok","version":"6.0.0"}
```

### 方法二：实际测试

在 OpenClaw 中发送包含敏感信息的 prompt：

```
我的邮箱是 user@example.com，API key 是 sk-1234567890，帮我查询余额
```

查看 Gateway 日志（会显示脱敏过程）：

```bash
# 如果通过 plugin 启动，查看 OpenClaw 日志
openclaw logs --follow | grep gateway

# 如果独立运行，直接看控制台输出
```

## 故障排查

### Gateway 无法启动

**错误**: `Port 8900 already in use`

**解决**: 更改端口
```json
{
  "gatewayPort": 9000
}
```

### OpenClaw 连接不到 Gateway

**检查**:
1. Gateway 是否在运行：`/mg_status`
2. 端口是否正确：默认 8900
3. Models 配置的 baseUrl 是否指向 `http://127.0.0.1:8900`

### 敏感数据没有被脱敏

**可能原因**:
1. Gateway 没有启用 - 检查 `sanitizePrompt: true`
2. Model 配置错误 - baseUrl 仍指向真实 API
3. 数据格式不匹配 - 检查是否符合正则表达式

**调试**:
查看 Gateway 日志确认请求是否通过了代理

### 响应中占位符没有还原

**这通常不会发生**，因为 Gateway 会自动还原。如果出现：
1. 检查 Gateway 日志是否有错误
2. 确认是同一个请求周期（映射表是 per-request 的）

## 性能影响

Gateway 是本地代理，性能影响极小：

- **脱敏延迟**: < 5ms (简单 prompt) 到 ~50ms (复杂嵌套结构)
- **还原延迟**: < 1ms (流式响应实时处理)
- **内存开销**: ~10MB (进程本身)
- **网络影响**: 0 (localhost 通信)

## 隐私保证

✅ **本地优先**: 所有脱敏/还原在本机完成
✅ **无持久化**: 映射表仅存在于请求周期内，用完即弃
✅ **无日志**: Gateway 不记录敏感信息原文
✅ **无外发**: 只有脱敏后的内容发送给 LLM
✅ **可审计**: 全部源码开放，可自行审查

## 支持的协议

| 协议 | 端点 | 支持的 Provider |
|------|-----|----------------|
| Anthropic Messages API | `/v1/messages` | Claude, Anthropic-compatible |
| OpenAI Chat Completions | `/v1/chat/completions` | GPT, Kimi, DeepSeek, 通义千问, 文心一言, etc. |
| Google Gemini | `/v1/models/:model:generateContent` | Gemini Pro, Flash |

## 高级用法

### 同时使用多个后端

Gateway 可以同时代理多个 LLM provider：

```json
{
  "models": {
    "providers": {
      "claude-protected": {
        "baseUrl": "http://127.0.0.1:8900",
        "api": "anthropic-messages",
        "apiKey": "${ANTHROPIC_API_KEY}"
      },
      "gpt-protected": {
        "baseUrl": "http://127.0.0.1:8900",
        "api": "openai-completions",
        "apiKey": "${OPENAI_API_KEY}"
      }
    }
  }
}
```

Gateway 会根据 API key 和协议自动路由到正确的后端。

### 自定义脱敏规则

编辑 `gateway/sanitizer.ts` 添加自定义规则：

```typescript
const ENTITIES: Entity[] = [
  // ... 现有规则
  {
    category: "CUSTOM_ID",
    categoryKey: "custom_id",
    pattern: /MY-ID-\d{8}/g,  // 自定义格式
  },
];
```

重启 Gateway 生效。

## 常见问题

**Q: Gateway 会影响 tool calling 吗？**
A: 不会。Tool 的参数会被正常还原，本地执行时使用的是真实值。

**Q: 多轮对话中的占位符会混乱吗？**
A: 不会。每个请求都有独立的映射表，编号从 1 开始。

**Q: 支持图片输入吗？**
A: 支持。图片内容（base64）不会被脱敏，只处理文本。

**Q: 可以关闭某些类型的脱敏吗？**
A: 目前需要修改源码。未来版本会支持配置化。

**Q: Gateway 崩溃会影响 OpenClaw 吗？**
A: 不会影响 OpenClaw 进程，但请求会失败。建议启用 `gatewayAutoStart: true` 让 plugin 自动重启。

## 更多帮助

- GitHub Issues: https://github.com/moltguard/moltguard/issues
- 文档: https://github.com/moltguard/moltguard#readme
