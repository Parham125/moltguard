# MoltGuard v6.0.0 Release Checklist

## 🎯 Release Overview

**Version:** 6.0.0 (Major Release)
**Release Date:** TBD
**Major Features:**
- ✨ Local Prompt Sanitization Gateway
- 🛡️ Enhanced Prompt Injection Detection (existing)

---

## ✅ Development - COMPLETED

### Core Implementation

- [x] Gateway HTTP server (`gateway/index.ts`)
- [x] Sanitizer module with recursive processing (`gateway/sanitizer.ts`)
- [x] Restorer module for placeholders (`gateway/restorer.ts`)
- [x] Protocol handlers:
  - [x] Anthropic Messages API (`gateway/handlers/anthropic.ts`)
  - [x] OpenAI Chat Completions API (`gateway/handlers/openai.ts`)
  - [x] Google Gemini API (`gateway/handlers/gemini.ts`)
- [x] Gateway configuration management (`gateway/config.ts`)
- [x] Gateway process manager (`gateway-manager.ts`)

### Plugin Integration

- [x] Plugin spawns and manages gateway process
- [x] Configuration schema updates (`openclaw.plugin.json`)
- [x] Type definitions (`agent/types.ts`)
- [x] Config resolver (`agent/config.ts`)
- [x] Gateway management commands:
  - [x] `/mg_status`
  - [x] `/mg_start`
  - [x] `/mg_stop`
  - [x] `/mg_restart`

### Testing

- [x] Unit tests for sanitizer (`gateway/test-gateway.ts`)
- [x] Verification: sanitization ✓
- [x] Verification: restoration ✓
- [x] Verification: nested structures ✓

### Documentation

- [x] Main README updated with Gateway features
- [x] SKILL.md updated for clawhub
- [x] Comprehensive Gateway guide (`GATEWAY_GUIDE.md`)
- [x] Testing guide (`TESTING.md`)
- [x] Release checklist (this file)

### Package Management

- [x] Version bumped to 6.0.0 in `package.json`
- [x] Version bumped to 6.0.0 in `openclaw.plugin.json`
- [x] `gateway/` directory added to `files` in `package.json`
- [x] Binary entry point added (`moltguard-gateway`)

---

## 🔍 Pre-Release Testing - TODO

### Local Testing

- [ ] Test gateway with real Anthropic API
  - [ ] Simple prompt with bank card
  - [ ] Multiple sensitive items
  - [ ] Tool calls with sensitive data
  - [ ] Streaming responses
- [ ] Test gateway with real OpenAI API
  - [ ] Same test cases as Anthropic
- [ ] Test gateway with Kimi API (if available)
  - [ ] Same test cases
- [ ] Verify injection detection still works
  - [ ] Download test file
  - [ ] Run `/og_status` and `/og_report`
  - [ ] Verify blocking functionality

### Integration Testing

- [ ] Test with different OpenClaw setups:
  - [ ] macOS (primary)
  - [ ] Linux (if available)
- [ ] Test gateway auto-start on OpenClaw restart
- [ ] Test gateway crash recovery
- [ ] Test concurrent requests to gateway
- [ ] Test large messages (>100KB)

### Edge Cases

- [ ] Test with no sensitive data (should pass through unchanged)
- [ ] Test with only placeholders already in text
- [ ] Test with unicode/emoji in sensitive data context
- [ ] Test tool calls with nested parameter structures
- [ ] Test error handling (invalid API keys, network errors)

---

## 📦 Build & Package - TODO

### Build

```bash
# Type check
npm run typecheck

# Build (if needed for distribution)
npm run build
```

### Package Verification

```bash
# Check package contents
npm pack --dry-run

# Verify files are included
# Should include: gateway/, agent/, memory/, index.ts, etc.
```

---

## 🚀 Release Process - TODO

### 1. Version Verification

- [ ] Confirm version 6.0.0 in:
  - [ ] `package.json`
  - [ ] `openclaw.plugin.json`
  - [ ] `gateway/index.ts` (health endpoint)

### 2. Git Preparation

```bash
# Stage all changes
git add .

# Commit
git commit -m "Release v6.0.0: Add local prompt sanitization gateway

- Add gateway HTTP proxy for local sensitive data protection
- Support Anthropic, OpenAI, and Gemini protocols
- Implement recursive sanitization with numbered placeholders
- Add gateway management commands (/mg_*)
- Update documentation with comprehensive guides
- Bump version to 6.0.0

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"

# Create tag
git tag -a v6.0.0 -m "Version 6.0.0 - Local Prompt Sanitization Gateway"

# Push to remote
git push origin main
git push origin v6.0.0
```

### 3. NPM Publishing

```bash
# Login to npm (if not already)
npm login

# Dry run
npm publish --dry-run

# Publish (for real)
npm publish

# Verify on npm
# https://www.npmjs.com/package/@openguardrails/moltguard
```

### 4. GitHub Release

- [ ] Create GitHub release for v6.0.0
- [ ] Add release notes (see template below)
- [ ] Attach compiled artifacts (if any)

---

## 📝 Release Notes Template

```markdown
# MoltGuard v6.0.0 - Local Prompt Sanitization Gateway

## 🎉 Major New Feature: Local Prompt Sanitization

Protect your sensitive data before it reaches any LLM with the new **local sanitization gateway**.

### What's New

✨ **Local Gateway Proxy**
- Automatically sanitizes bank cards, passwords, API keys, and more
- Works with any LLM provider (Claude, GPT, Kimi, etc.)
- 100% local processing - sensitive data never leaves your machine
- Transparent to users and LLMs

🔒 **Supported Data Types**
- Bank cards, credit cards
- Email addresses, phone numbers
- API keys, secrets, Bearer tokens
- IP addresses, SSNs, IBANs, URLs

🚀 **Easy Setup**
- Enable in config: `"sanitizePrompt": true`
- Point your model to gateway: `"baseUrl": "http://127.0.0.1:8900"`
- Restart OpenClaw - done!

### New Commands

- `/mg_status` - View gateway status
- `/mg_start` - Start gateway
- `/mg_stop` - Stop gateway
- `/mg_restart` - Restart gateway

### Documentation

- [Gateway Guide](./GATEWAY_GUIDE.md) - Complete setup and usage
- [Testing Guide](./TESTING.md) - Test with real APIs
- [Updated README](./README.md) - Overview of all features

## 🛡️ Existing Feature: Prompt Injection Detection

Unchanged from v5.0, still works as before:
- Detects malicious instructions in external content
- Local sanitization before API analysis
- Commands: `/og_status`, `/og_report`, `/og_feedback`

## 📦 Installation

```bash
# New installation
openclaw plugins install @openguardrails/moltguard

# Upgrade from v5.x
openclaw plugins update @openguardrails/moltguard
openclaw gateway restart
```

## 🔄 Breaking Changes

None. Gateway is opt-in via `sanitizePrompt: true`.

## 🐛 Bug Fixes

- Improved phone number detection (international formats)
- Better handling of nested message structures

## 📊 Stats

- **Files changed:** 15+
- **Lines added:** ~2000
- **New modules:** 8
- **Test coverage:** Core sanitizer/restorer verified

## 🙏 Credits

Built with Claude Opus 4.6.

---

**Full Changelog:** https://github.com/moltguard/moltguard/compare/v5.0.0...v6.0.0
```

---

## 🎯 Post-Release - TODO

### Immediate

- [ ] Announce on GitHub
- [ ] Update clawhub skill listing (if applicable)
- [ ] Update moltguard.com website (if applicable)

### Monitoring

- [ ] Monitor npm downloads
- [ ] Watch for GitHub issues
- [ ] Collect user feedback
- [ ] Monitor gateway performance in production

### Future Improvements

- [ ] Add configuration UI for sensitive data types
- [ ] Support custom sanitization patterns
- [ ] Add metrics/telemetry (opt-in)
- [ ] Support additional protocols (if needed)
- [ ] Performance optimization for large messages
- [ ] Add caching layer for repeated patterns

---

## 📞 Support

- **Issues:** https://github.com/moltguard/moltguard/issues
- **Email:** (if applicable)
- **Discord:** (if applicable)

---

## ✅ Sign-Off

**Ready for release when all TODO items are checked.**

- [ ] All development completed
- [ ] All tests passed
- [ ] All documentation updated
- [ ] Package verified
- [ ] Git tagged
- [ ] NPM published
- [ ] GitHub release created
- [ ] Announcements sent

**Release Manager:** _____________
**Date:** _____________
