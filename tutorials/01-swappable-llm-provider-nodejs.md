# How to Add a Swappable LLM Provider (OpenAI + Gemini) to a Node.js API

> **Type:** Step-by-step tutorial  
> **Stack:** Node.js, Express, OpenAI API, Google Gemini API  
> **Based on:** Bilateral-LLM (React + Express trade-insights app)

---

## What you will build

By the end of this tutorial you will have:

1. A single `chatWithLlm()` function that talks to **OpenAI or Gemini**
2. Admin-only Express routes: `GET /api/llm/status` and `POST /api/llm/chat`
3. Environment-based provider switching (no code change to flip vendors)
4. Clear HTTP error codes when the API key is missing

**Time:** ~30–45 minutes  
**Level:** Intermediate

---

## Prerequisites

- Node.js 18+
- An Express API (or the Bilateral-LLM backend)
- An OpenAI **or** Gemini API key
- Basic familiarity with `async/await` and `.env` files

---

## Step 1 — Add environment variables

Create or edit `backend/.env`:

```env
LLM_PROVIDER=openai
OPENAI_API_KEY=your_openai_key
OPENAI_MODEL=gpt-4o-mini

# Or use Gemini instead:
# LLM_PROVIDER=gemini
# GEMINI_API_KEY=your_gemini_key
# GEMINI_MODEL=gemini-2.5-flash
```

Never commit this file. Keep a `.env.example` with empty placeholders only.

---

## Step 2 — Resolve the provider from config

Create `backend/utils/llm.js` and add:

```js
function resolveProvider() {
  const explicit = (process.env.LLM_PROVIDER || '').trim().toLowerCase();
  if (explicit === 'openai' || explicit === 'gemini') return explicit;
  if (process.env.OPENAI_API_KEY?.trim()) return 'openai';
  if (process.env.GEMINI_API_KEY?.trim()) return 'gemini';
  return 'openai';
}

function getProviderConfig() {
  const provider = resolveProvider();
  if (provider === 'openai') {
    return {
      provider: 'openai',
      apiKey: process.env.OPENAI_API_KEY?.trim() || null,
      model: process.env.OPENAI_MODEL?.trim() || 'gpt-4o-mini',
      baseUrl: (process.env.OPENAI_BASE_URL || 'https://api.openai.com/v1').replace(/\/$/, ''),
    };
  }
  return {
    provider: 'gemini',
    apiKey: process.env.GEMINI_API_KEY?.trim() || null,
    model: process.env.GEMINI_MODEL?.trim() || 'gemini-2.5-flash',
    baseUrl: 'https://generativelanguage.googleapis.com/v1beta',
  };
}

export function getLlmStatus() {
  const cfg = getProviderConfig();
  return {
    configured: Boolean(cfg.apiKey),
    provider: cfg.provider,
    model: cfg.model,
    label: cfg.provider === 'openai' ? 'OpenAI' : 'Google Gemini',
  };
}
```

---

## Step 3 — Normalize messages for each vendor

Keep one request shape from the frontend: `{ role, content }[]`.

Convert inside the utility:

| Vendor | System prompt | Roles |
|--------|---------------|-------|
| OpenAI | `messages[].role = system` | `user` / `assistant` |
| Gemini | `systemInstruction` | `user` / `model` |

```js
function toOpenAiMessages(messages, systemPrompt) {
  const out = [];
  if (systemPrompt?.trim()) {
    out.push({ role: 'system', content: systemPrompt.trim() });
  }
  for (const m of messages) {
    if (!m?.content?.trim()) continue;
    const role = (m.role || 'user').toLowerCase();
    out.push({
      role: ['model', 'assistant', 'ai'].includes(role) ? 'assistant' : 'user',
      content: m.content.trim(),
    });
  }
  return out;
}
```

---

## Step 4 — Implement `chatWithLlm()`

```js
export async function chatWithLlm({ messages, systemPrompt, maxTokens } = {}) {
  const cfg = getProviderConfig();
  if (!cfg.apiKey) {
    const err = new Error(
      cfg.provider === 'openai'
        ? 'OPENAI_API_KEY is not configured'
        : 'GEMINI_API_KEY is not configured'
    );
    err.code = 'LLM_NOT_CONFIGURED';
    throw err;
  }

  if (cfg.provider === 'openai') {
    return chatWithOpenAI({ messages, systemPrompt, maxTokens }, cfg);
  }
  return chatWithGeminiProvider({ messages, systemPrompt, maxTokens }, cfg);
}
```

Call OpenAI at `POST {baseUrl}/chat/completions` and Gemini at  
`POST .../models/{model}:generateContent?key=...`.

Use a low temperature (for example `0.2`) when drafting operational / analytical text.

---

## Step 5 — Add admin-only routes

```js
router.get('/status', requireAuth, requireRole('admin'), (req, res) => {
  res.json(getLlmStatus());
});

router.post('/chat', requireAuth, requireRole('admin'), async (req, res) => {
  const { messages, systemPrompt } = req.body;
  // validate messages: non-empty array, max 50 items, max 32k chars each
  try {
    const result = await chatWithLlm({ messages, systemPrompt });
    res.json({ success: true, reply: result.reply, model: result.model });
  } catch (error) {
    if (error.code === 'LLM_NOT_CONFIGURED') {
      return res.status(503).json({ error: error.message, code: error.code });
    }
    const status = error.status === 429 ? 429 : 502;
    res.status(status).json({ error: error.message, code: error.code || 'LLM_ERROR' });
  }
});
```

Mount under `/api/llm`.

---

## Step 6 — Run and verify

```bash
npm run dev
```

1. Sign in as an admin  
2. `GET /api/llm/status` → `{ configured: true, provider: "openai", ... }`  
3. `POST /api/llm/chat` with a simple user message  
4. Change `LLM_PROVIDER=gemini`, restart, repeat — routes stay the same  

---

## Optional — Corporate TLS

If HTTPS to the vendor fails behind a proxy:

```env
LLM_CA_CERT_FILE=C:\certs\corp-root.pem
# Dev only:
# LLM_TLS_SKIP_VERIFY=true
```

Never enable TLS skip in production.

---

## Checklist

- [ ] `.env` has one provider key  
- [ ] `chatWithLlm` is the only function routes call  
- [ ] `/status` and `/chat` require admin  
- [ ] Missing key returns **503** with `LLM_NOT_CONFIGURED`  
- [ ] Flipping `LLM_PROVIDER` does not require UI changes  

---

## Next steps

- Stub `chatWithLlm` in CI (`LLM_STUB=true`) so tests do not spend tokens  
- Add rate limiting on `/api/llm/chat`  
- Keep a human approval step before saving generated text into shared records  

---

## Source

Patterns taken from the Bilateral-LLM codebase (`backend/utils/llm.js`, `backend/routes/llm.js`).
