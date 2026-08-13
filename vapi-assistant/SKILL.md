---
name: vapi-assistant
description: Build and manage Vapi.ai AI voice agents with the user's OWN Vapi API key. First run asks the user to paste their key, stores it in the per-user keychain, then creates/manages assistants, phone numbers, and calls. Use when the user wants a voice agent, phone assistant, or outbound calling.
version: 1.0.0
author: Diazites
license: MIT
metadata:
  hermes:
    tags: [vapi, voice-agent, telephony, keychain]
---

# Vapi Voice Agent Skill (user-owned API key)

Build and manage Vapi.ai voice assistants **with the user's own Vapi API key**.
Every user brings their own key — this workspace never holds a shared Vapi secret.

## Step 0 — The user's API key (MANDATORY first step)

Before ANY Vapi API call, check whether the user's keychain already has a Vapi credential:

```bash
curl -fsS -H "x-agent-capability: $AGENT_API_TOKEN" "$AGENT_API_URL/v1/keychain/credentials"
```

If the response contains a credential with `"service": "vapi"`, read its `id` and skip to Step 1.

**If there is NO vapi credential yet:**
1. Ask the user directly in chat: *"Please paste your Vapi API key (from app.vapi.ai → Settings → API Keys). It stays in your private keychain — I never see it again after saving."*
2. Save it to their personal keychain:
```bash
curl -fsS -X POST -H "x-agent-capability: $AGENT_API_TOKEN" "$AGENT_API_URL/v1/keychain/credentials" \
  -H 'content-type: application/json' \
  -d '{"service":"vapi","secret":"<KEY-THE-USER-PASTED>","envKey":"VAPI_API_KEY"}'
```
3. Confirm to the user: *"Saved. Your key is encrypted in your workspace and only used for your assistants."*
4. Do NOT echo the key back into the conversation. Do NOT store it in any file.

If the user refuses to provide a key, tell them Vapi is required and stop — never proceed with a made-up or shared key.

## Step 1 — Materialize the key into the current turn (each turn)

Get the credential id from Step 0's list, then load it into this turn's shell:

```bash
curl -fsS -X POST -H "x-agent-capability: $AGENT_API_TOKEN" "$AGENT_API_URL/v1/keychain/use" \
  -H 'content-type: application/json' -d '{"credential":"<vapi-credential-id>"}' \
  -o /tmp/keychain.env && . /tmp/keychain.env
```

This exports `VAPI_API_KEY` for the rest of the turn. Verify it's set without printing the value:
```bash
[ -n "$VAPI_API_KEY" ] && echo "VAPI key loaded" || echo "MISSING"
```

## Step 2 — Core Vapi operations

Base URL: `https://api.vapi.ai` · Auth: `Authorization: Bearer $VAPI_API_KEY` on every request.

### List assistants (sanity check that the key works)
```bash
curl -s https://api.vapi.ai/assistant -H "Authorization: Bearer $VAPI_API_KEY" | head -c 500
```
HTTP 200 with JSON = key valid. HTTP 401 = wrong key → tell user to re-paste.

### Create an assistant
```bash
curl -s -X POST https://api.vapi.ai/assistant \
  -H "Authorization: Bearer $VAPI_API_KEY" -H "Content-Type: application/json" \
  -d '{
    "name": "Receptionist",
    "model": {"provider": "openai", "model": "gpt-4o", "messages": [
      {"role": "system", "content": "You are a friendly receptionist for a local business. Answer calls, take details, book appointments."}
    ]},
    "voice": {"provider": "11labs", "voiceId": "21m00Tcm4TlvDq8ikWAM"},
    "transcriber": {"provider": "deepgram", "model": "nova-2", "language": "en-US"},
    "firstMessage": "Hi! Thanks for calling. How can I help you today?",
    "recordingEnabled": true
  }'
```
Save the returned `id` — that's the assistant id the user needs.

### List / assign phone numbers
```bash
curl -s https://api.vapi.ai/phone-number -H "Authorization: Bearer $VAPI_API_KEY"
```
To point a number at the assistant:
```bash
curl -s -X PATCH "https://api.vapi.ai/phone-number/<phone-number-id>" \
  -H "Authorization: Bearer $VAPI_API_KEY" -H "Content-Type: application/json" \
  -d '{"assistantId":"<assistant-id>"}'
```

### Place an outbound call
```bash
curl -s -X POST https://api.vapi.ai/call \
  -H "Authorization: Bearer $VAPI_API_KEY" -H "Content-Type: application/json" \
  -d '{"assistantId":"<assistant-id>","phoneNumberId":"<phone-number-id>","customer":{"number":"+15551234567","name":"Customer"}}'
```

### Check call result (poll after ~15-30s)
```bash
curl -s "https://api.vapi.ai/call/<call-id>" -H "Authorization: Bearer $VAPI_API_KEY" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print('status:', d.get('status')); print('ended:', d.get('endedReason')); print('cost: \$%.4f' % d.get('cost',0)); print('dur:', d.get('durationSeconds',0), 's')"
```

## Golden rules

- **The user's key is theirs alone.** Never print it, never write it to a file, never share it between users. The keychain keeps it per-user.
- **Never fabricate an API key or reuse a shared one.** If there's no vapi credential in the keychain and the user won't provide one, stop and say the key is required.
- **Confirm before anything that costs money** (phone number purchases, campaigns, long outbound lists).
- **Assistant ids and phone number ids can go stale** — always re-list before placing calls in a new turn.
- **Deliver results in writing** in chat: assistant id, number assigned, call outcome. No narration without output.
- **Webhooks must be HTTPS.** If the user needs call results sent somewhere, ask for their webhook URL; never invent one.
- **Cloudflare may block POST /call from some IPs** — retry with `curl --http1.1` and a mobile browser User-Agent if you get timeouts or 503s.

## Common errors → plain-language fix

| Error | Meaning | Fix |
|---|---|---|
| HTTP 401 | Key invalid/revoked | Ask user to re-paste their Vapi key, save again |
| `daily outbound call limit` | Vapi-bought number capped (~50/day) | User should import a Twilio number instead |
| `artifact` is null | Call too short/still running | Check `recordingUrl` at top level, or poll again |
| 400 `voicemailDetection should not exist` | Field rejected on POST /call | Remove `voicemailDetection` from the payload |
| Timeouts / 503 | Cloudflare or Vapi backend hiccup | Retry after 10-30s, use `--http1.1` |

## What the user needs to have

1. A Vapi account (app.vapi.ai) with an API key — they paste it once.
2. A phone number on Vapi or Twilio (optional — needed to receive/place real calls).
3. Their own OpenAI/Anthropic key or Vapi's built-in billing for the LLM inside the assistant.
