# MoltGuard Gateway Testing Guide

This guide helps you test the MoltGuard Gateway with real LLM APIs to verify that sensitive data is properly sanitized and restored.

## Prerequisites

- OpenClaw installed and configured
- MoltGuard plugin installed (`openclaw plugins install @openguardrails/moltguard`)
- API keys for at least one LLM provider (Anthropic, OpenAI, or Kimi)

## Test Setup

### 1. Enable Gateway in Plugin Config

Edit `~/.openclaw/openclaw.json`:

```json
{
  "plugins": {
    "entries": {
      "moltguard": {
        "config": {
          "sanitizePrompt": true,
          "gatewayPort": 8900,
          "gatewayAutoStart": true
        }
      }
    }
  }
}
```

### 2. Set Environment Variables

```bash
# For Anthropic (Claude)
export ANTHROPIC_API_KEY="sk-ant-your-key-here"

# For OpenAI (GPT)
export OPENAI_API_KEY="sk-your-key-here"

# For Kimi (Moonshot)
export KIMI_API_KEY="sk-your-key-here"
```

### 3. Configure Model to Use Gateway

Edit `~/.openclaw/openclaw.json` to add a gateway-protected model:

#### Option A: Anthropic (Claude)

```json
{
  "models": {
    "providers": {
      "claude-protected": {
        "baseUrl": "http://127.0.0.1:8900",
        "api": "anthropic-messages",
        "apiKey": "${ANTHROPIC_API_KEY}",
        "models": [
          {
            "id": "claude-sonnet-4-20250514",
            "name": "Claude Sonnet (Protected)",
            "reasoning": false,
            "input": ["text", "image"],
            "cost": {
              "input": 3.0,
              "output": 15.0,
              "cacheRead": 0.3,
              "cacheWrite": 3.75
            },
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
        "primary": "claude-protected/claude-sonnet-4-20250514"
      }
    }
  }
}
```

#### Option B: OpenAI (GPT)

```json
{
  "models": {
    "providers": {
      "gpt-protected": {
        "baseUrl": "http://127.0.0.1:8900",
        "api": "openai-completions",
        "apiKey": "${OPENAI_API_KEY}",
        "models": [
          {
            "id": "gpt-4o",
            "name": "GPT-4o (Protected)",
            "reasoning": false,
            "input": ["text", "image"],
            "cost": {
              "input": 2.5,
              "output": 10.0
            },
            "contextWindow": 128000,
            "maxTokens": 4096
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "gpt-protected/gpt-4o"
      }
    }
  }
}
```

#### Option C: Kimi (Moonshot)

```json
{
  "models": {
    "providers": {
      "kimi-protected": {
        "baseUrl": "http://127.0.0.1:8900",
        "api": "openai-completions",
        "apiKey": "${KIMI_API_KEY}",
        "models": [
          {
            "id": "moonshot-v1-128k",
            "name": "Kimi (Protected)",
            "reasoning": false,
            "input": ["text"],
            "cost": {
              "input": 0.2,
              "output": 0.2
            },
            "contextWindow": 128000,
            "maxTokens": 4096
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "kimi-protected/moonshot-v1-128k"
      }
    }
  }
}
```

### 4. Restart OpenClaw

```bash
openclaw gateway restart
```

## Verification Steps

### Step 1: Check Gateway Status

In OpenClaw chat, run:

```
/mg_status
```

**Expected output:**
```
**MoltGuard Gateway Status**

- Running: ✓ Yes
- Ready: ✓ Yes
- Port: 8900
- Endpoint: http://127.0.0.1:8900
```

If gateway is not running, start it:
```
/mg_start
```

### Step 2: Verify Gateway is Receiving Requests

Open a terminal and tail the OpenClaw logs:

```bash
openclaw logs --follow | grep gateway
```

Or check the gateway health endpoint:

```bash
curl http://127.0.0.1:8900/health
# Should return: {"status":"ok","version":"6.0.0"}
```

### Step 3: Test with Sensitive Data

Send a prompt containing sensitive information to test sanitization:

#### Test Case 1: Bank Card

```
我的银行卡号是 6222021234567890，请帮我查询余额
```

**What should happen:**
1. Gateway sanitizes the card number to `__bank_card_1__`
2. LLM sees: "我的银行卡号是 __bank_card_1__，请帮我查询余额"
3. LLM responds with `__bank_card_1__` in the response
4. Gateway restores it back to `6222021234567890`

**In the logs**, you should see:
```
[moltguard-gateway] POST /v1/messages
[moltguard-gateway] POST /v1/chat/completions
```

#### Test Case 2: Multiple Sensitive Items

```
请用这些信息注册账号：
- 邮箱：user@example.com
- 手机：+86-138-1234-5678
- API Key：sk-1234567890abcdef
```

**What should happen:**
- Email → `__email_1__`
- Phone → `__phone_1__`
- API key → `__secret_1__`

All placeholders should be restored in the final response.

#### Test Case 3: Tool Calls with Sensitive Data

```
我的密码是 MySecretPass123!，请保存到配置文件
```

**What should happen:**
1. Password is sanitized to `__secret_1__`
2. If the LLM decides to call a tool (e.g., write file)
3. The tool parameters should have the placeholder restored to the real password
4. The file should be written with the actual password

### Step 4: Verify Logs (Optional)

Gateway logs show the full request/response flow. If you started the gateway manually (not through plugin), you'll see detailed logs in the console.

## Test Checklist

- [ ] Gateway starts successfully (`/mg_status` shows running)
- [ ] Health endpoint responds (`curl http://127.0.0.1:8900/health`)
- [ ] Bank card numbers are sanitized and restored
- [ ] Email addresses are sanitized and restored
- [ ] Phone numbers are sanitized and restored
- [ ] API keys/secrets are sanitized and restored
- [ ] Multiple sensitive items in one message are all handled
- [ ] Tool calls receive restored (real) values, not placeholders
- [ ] Streaming responses work correctly
- [ ] No errors in OpenClaw logs

## Troubleshooting

### Gateway Won't Start

**Error:** `Port 8900 already in use`

**Solution:** Change the port in config:
```json
{
  "gatewayPort": 9000
}
```

Then update model config to use port 9000.

---

**Error:** Gateway starts but shows "Not Ready"

**Solution:** Check logs for errors:
```bash
openclaw logs --follow | grep gateway
```

### OpenClaw Can't Connect to Gateway

**Check:**
1. Is gateway running? `/mg_status`
2. Is the port correct in model config? (default: 8900)
3. Is `baseUrl` set to `http://127.0.0.1:8900`?

**Debug:**
```bash
# Test if gateway is listening
curl http://127.0.0.1:8900/health

# Should return: {"status":"ok","version":"6.0.0"}
```

### Sensitive Data Not Being Sanitized

**Possible causes:**
1. Gateway is not enabled (`sanitizePrompt: true`)
2. Model is not configured to use gateway (check `baseUrl`)
3. Data format doesn't match patterns (check `gateway/sanitizer.ts`)

**Verify request is going through gateway:**
```bash
# In one terminal, start gateway manually to see logs
cd /Users/tom/workspace/dev/moltguard/moltguard
npx tsx gateway/index.ts

# In another terminal, send a request via OpenClaw
# You should see POST requests in the gateway terminal
```

### Placeholders Not Being Restored

This is very rare. If it happens:

1. Check that it's the same request cycle (mappings are per-request)
2. Check gateway logs for errors during restoration
3. Verify the placeholder format matches (e.g., `__bank_card_1__`)

## Advanced Testing

### Test with Direct API Call

You can test the gateway directly without OpenClaw:

```bash
# Test Anthropic endpoint
curl -X POST http://127.0.0.1:8900/v1/messages \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -d '{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 1024,
    "messages": [
      {
        "role": "user",
        "content": "我的邮箱是 test@example.com，卡号是 6222021234567890"
      }
    ]
  }'
```

Check the response - you should see the original sensitive data restored.

### Test Streaming

```bash
curl -X POST http://127.0.0.1:8900/v1/messages \
  -H "Content-Type: application/json" \
  -H "anthropic-version: 2023-06-01" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -d '{
    "model": "claude-sonnet-4-20250514",
    "max_tokens": 1024,
    "stream": true,
    "messages": [
      {
        "role": "user",
        "content": "我的API key是 sk-1234567890，请告诉我它的格式"
      }
    ]
  }' -N
```

The SSE stream should show the restored API key in the chunks.

## Success Criteria

The gateway is working correctly if:

1. ✅ All sensitive data is replaced with numbered placeholders before reaching the LLM
2. ✅ Placeholders are correctly restored to original values in responses
3. ✅ Tool calls execute with real (restored) values, not placeholders
4. ✅ No errors or warnings in logs
5. ✅ Performance is acceptable (<50ms sanitization overhead)
6. ✅ Works with streaming responses

## Reporting Issues

If you encounter issues:

1. Collect logs: `openclaw logs > moltguard-debug.log`
2. Include your configuration (with API keys redacted)
3. Describe the test case that failed
4. Open an issue: https://github.com/moltguard/moltguard/issues

## Next Steps

After successful testing:

1. Use the gateway with your daily workflows
2. Monitor performance and accuracy
3. Report any false positives (data that should be sanitized but isn't)
4. Report any false negatives (data that shouldn't be sanitized but is)
5. Consider contributing additional patterns to `gateway/sanitizer.ts`
