# Claude Code Third-Party Client Development Guide

Based on reverse engineering analysis of v2.1.88 source code, this provides a complete implementation guide for API calls and authentication.

---

## Table of Contents

1. Two Authentication Methods  
2. Method 1: API Key Direct Usage  
3. Method 2: OAuth Login (Claude Max/Pro Subscription)  
4. Messages API Full Request Format  
5. Streaming Response Handling  
6. Tool Call Loop  
7. System Prompt Construction  
8. Full Code Examples  
9. Auxiliary API Endpoints  
10. Notes  

---

## 1. Two Authentication Methods

| Method | Use Case | Billing |
|--------|----------|--------|
| API Key | Developers, pay-as-you-go | Console billing |
| OAuth Login | Claude Max/Pro/Team users | Included in subscription |

---

## 2. Method 1: API Key Direct Usage

The simplest method, requiring only an API Key.

### 2.1 Get API Key

Create one at:
https://console.anthropic.com/settings/keys

### 2.2 Usage Example

```javascript
const response = await fetch('https://api.anthropic.com/v1/messages', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-api-key': 'sk-ant-xxxxx',
    'anthropic-version': '2023-06-01',
    'anthropic-beta': 'interleaved-thinking-2025-05-14',
    'x-app': 'cli',
  },
  body: JSON.stringify({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 16384,
    messages: [
      { role: 'user', content: 'Hello, Claude!' }
    ],
    stream: true
  })
})
```

---

## 3. Method 2: OAuth Login (Claude Max/Pro)

This is the method used by Claude Code, using OAuth PKCE flow.

### Key Parameters

- CLIENT_ID: 9d1c250a-e61b-44d9-88ed-5944d1962f5e  
- AUTHORIZE_URL: https://claude.com/cai/oauth/authorize  
- TOKEN_URL: https://platform.claude.com/v1/oauth/token  

---

### OAuth Flow Summary

1. Generate PKCE parameters  
2. Start local callback server  
3. Open browser for authorization  
4. Exchange code for token  
5. Refresh token when expired  

---

### Token Storage

- macOS: Keychain  
- Linux/Windows: `~/.claude/.credentials.json`  

---

## 4. Messages API Format

### Headers

- x-api-key OR Authorization: Bearer token  
- anthropic-version required  
- optional beta headers  

### Body includes:

- model  
- messages  
- system prompt  
- tools  
- thinking config  
- metadata  

---

## 5. Streaming Response Handling

Uses SSE events:

- message_start  
- content_block_start  
- content_block_delta  
- message_stop  

---

## 6. Tool Call Loop

Core logic:

1. Send request  
2. Receive tool_use  
3. Execute tool  
4. Send result back  
5. Repeat until finished  

---

## 7. System Prompt Construction

Includes:

- Static sections (role, rules)  
- Dynamic sections (context, environment)  

---

## 8. Full Code Examples

Includes:

- Minimal API Key client  
- OAuth client  
- Tool execution logic  
- Streaming parser  

---

## 9. Auxiliary API Endpoints

- User profile  
- Roles  
- Create API key via OAuth  

---

## 10. Notes

### Key Differences

| Feature | API Key | OAuth |
|--------|--------|--------|
| Header | x-api-key | Authorization Bearer |
| Expiry | No | Yes |
| Billing | Pay-as-you-go | Subscription |

### Security Recommendations

- Never hardcode API keys  
- Securely store refresh tokens  
- Validate file paths  
- Sandbox shell execution  

---

*Based on Claude Code v2.1.88 reverse engineering analysis*  
*Generated on 2026-03-31*
