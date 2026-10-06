# Your Route Handlers Should Not Know Vendor Names

> **Type:** Opinion / experience piece  
> **Stack:** Node.js, Express, LLM APIs  
> **Based on:** Bilateral-LLM (OpenAI + Gemini behind one function)

---

## The claim

When you add AI to a product, the first abstraction you need is not a framework or an agent SDK.

It is this:

```text
HTTP route  →  chatWithLlm(messages)  →  OpenAI | Gemini | tomorrow’s vendor
```

If `routes/llm.js` imports OpenAI’s client and Gemini’s client, you will rewrite the same auth, validation, and error mapping twice—and you will ship the wrong one to the environment that only allows the other.

---

## What we actually needed

In Bilateral-LLM, admins draft insight prompts for bilateral trade charts. Requirements that mattered more than model quality:

1. **Switch providers without a UI deploy** — ops preference and network policy change  
2. **Admin-only** — this is not a public chatbot  
3. **Honest failures** — missing key → 503, not a stack trace  
4. **Corporate TLS reality** — certificates and proxies break naive `fetch`  

None of those belong inside a React component. Most of them do not belong inside a route handler either.

---

## The smell

```js
// route handler
if (process.env.LLM_PROVIDER === 'openai') {
  const openai = new OpenAI({ apiKey: ... });
  const result = await openai.chat.completions.create(...);
  res.json({ reply: result.choices[0].message.content });
} else {
  // 80 lines of Gemini-shaped JSON
}
```

Problems:

- Message role names differ (`assistant` vs `model`)  
- System prompts are attached differently  
- Error shapes differ  
- Tests must mock two SDKs  
- The next vendor (Azure OpenAI, gateway, local model) forks the route again  

---

## The fix that stayed small

One module. One exported function. Provider resolved from env:

```js
export async function chatWithLlm({ messages, systemPrompt, maxTokens } = {}) {
  const cfg = getProviderConfig();
  if (!cfg.apiKey) {
    const err = new Error('API key is not configured');
    err.code = 'LLM_NOT_CONFIGURED';
    throw err;
  }
  return cfg.provider === 'openai'
    ? chatWithOpenAI({ messages, systemPrompt, maxTokens }, cfg)
    : chatWithGeminiProvider({ messages, systemPrompt, maxTokens }, cfg);
}
```

Routes only:

- Authenticate + require admin  
- Validate payload size  
- Call `chatWithLlm`  
- Map `error.code` → HTTP status  

That is enough. You do not need a plugin architecture on day one.

---

## Opinions I am willing to defend

**1. “Configured?” is a product feature.**  
Expose `GET /status` with `{ configured, provider, model }`. Hiding a dead chat button beats a cryptic 502.

**2. Low temperature is a product choice for ops text.**  
Drafting statistical prompts is not creative writing. `temperature: 0.2` reduced invented figures.

**3. Vendor SDKs are optional.**  
Plain HTTPS + JSON kept the dependency surface small and made custom CA handling easier on locked-down networks.

**4. Human review stays in front of shared writes.**  
LLMs draft. Approval workflows publish. Do not let “Save to all countries” become “trust the model.”

**5. Dummy admin and TLS skip are local tools, not production flags.**  
If it is convenient in January, it will be left on in June. Disable them when `NODE_ENV=production`.

---

## When a thicker abstraction is worth it

- You stream tokens to the UI  
- You orchestrate tools / RAG / multi-step agents  
- You need per-tenant model routing  

Until then, a 200-line utility beats an “AI platform” rewrite.

---

## Bottom line

Routes should speak **your** domain language: messages in, reply out, structured errors.

Vendor names belong in config and in one adapter file—the same way database drivers belong behind a repository, not inside every controller.

If you cannot flip `LLM_PROVIDER` without touching `routes/`, the abstraction is in the wrong place.

---

## Related reading in this repo

- Tutorial: [Swappable LLM provider (step-by-step)](../tutorials/01-swappable-llm-provider-nodejs.md)  
