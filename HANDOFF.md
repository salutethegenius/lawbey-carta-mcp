# Carta Integration — LawBey MCP Test Endpoint (Handoff)

**Live server:** https://lawbey-mcp.fly.dev  
**Status:** deployed · `/health` → `{"status":"ok","service":"lawbey-mcp"}`  
**Handoff repo (this doc):** https://github.com/salutethegenius/lawbey-carta-mcp  
**Implementation repo (private):** https://github.com/salutethegenius/lawbey-mcp

> The Carta **partner API key** is a secret and is **not** in this document.
> It is delivered out-of-band (Slack / 1Password / email). The header value to
> send is `Bearer carta:<PARTNER_KEY>`.

---

## 1. Two ways to connect

### Option A — REST shim (simplest; start here)

```
POST https://lawbey-mcp.fly.dev/debug/query
Authorization: Bearer carta:<PARTNER_KEY>
Content-Type: application/json

{"query":"What are the penalties for smuggling migrants in The Bahamas?"}
```

### Option B — MCP SSE transport (for an MCP-aware agent)

- SSE stream: `GET  https://lawbey-mcp.fly.dev/sse`
- Messages:  `POST https://lawbey-mcp.fly.dev/messages`
- Same `Authorization: Bearer carta:<PARTNER_KEY>` header on both.
- Tool exposed: `search_bahamian_law(query: str, context: str | None = None)`

---

## 2. Tool: `search_bahamian_law`

**Args**
- `query` (str, required): plain-English legal question. Max 2000 chars.
- `context` (str, optional): situational framing. Do **not** include PII.

**Returns**
```json
{
  "answer":      "On summary conviction for basic smuggling: a fine not exceeding $100,000 or imprisonment up to 7 years, or both [§5(3)(a)]...",
  "sources":     [{"document_name": "smuggling_of_migrants_act_2025_part1.md",
                   "collection": "CARTA mcp-test",
                   "chunks": ["..."], "relevance_scores": [0.31]}],
  "citations":   ["§5(3)(a)", "§5(3)(b)", "§5(5)", "§6(1)"],
  "grounded":    true,
  "disclaimer":  "For specific legal matters, please consult with a qualified Bahamian attorney or contact the Bahamas Bar Association."
}
```

| Field | Meaning |
|---|---|
| `answer` | Grounded legal answer with inline `[§x(y)]` citations, or a decline |
| `sources` | Retrieved statute chunks + the KB collection name |
| `citations` | Extracted bracketed references |
| `grounded` | `true` if the answer is based on retrieved docs; `false` if declined / off-scope |
| `disclaimer` | Standard legal disclaimer — always present |

**Error envelope:**
```json
{"answer": null, "sources": [], "citations": [], "grounded": false,
 "disclaimer": "...", "error": "<kind>", "message": "<detail>"}
```

Common `error` kinds: `empty_query`, `query_too_long`, `auth_error`,
`upstream_unavailable`, `upstream_error`, `rag_unavailable` (KB retrieval
returned no matching documents).

---

## 3. What’s in the knowledge base (current)

Multi-act Bahamian legal library (~140 markdown files) scoped to the CARTA
collection. Coverage includes:

| Area | Examples in KB |
|---|---|
| **Entities / corporate** | International Business Companies (IBC) Act; Companies Act; Foundations Act; Trustee Act; Exempted Limited Partnerships (ELP); Executive Entities |
| **Economic substance / AML** | CESRA 2023 guidelines; FTRA (2000 / 2018 / amendments); beneficial ownership |
| **Digital assets** | DARE Act 2024; DARE AML/CFT rules; SCB digital asset regulation notices |
| **Financial services** | FCSP Act 2020 + regs / fees; Investment Funds Act (IFA) materials; SMART fund constitutive notes |
| **Banking / trust** | Banks and Trust Companies Regulation Act (BTCRA) 2000 rev. + 2020 + 2025 amendments |
| **Securities** | Securities Industry Act (SIA) materials |
| **Exchange control** | Exchange Control regs (Ch. 360); 2024 relaxation |
| **Migration / offences** | Smuggling of Migrants Act 2025 |
| **Notices / guidance** | SCB FCSP AML briefing; SCB AML/CFT briefing 2025 |

New statutes can be added to the same knowledge base with **no Carta code change**.

---

## 4. Sample prompts (copy/paste)

Use these against `/debug/query` or `search_bahamian_law`. Expect
`grounded: true` and sources whose filenames match the topic.

### Smuggling of migrants
```text
What are the penalties for smuggling migrants in The Bahamas?
```
```text
What is the jurisdiction of the Smuggling of Migrants Act?
```

### IBC / companies
```text
What are the requirements for incorporating an International Business Company (IBC) in The Bahamas?
```
```text
Does a Bahamian IBC need a registered office in The Bahamas?
```

### CESRA / economic substance
```text
What are the economic substance requirements under CESRA for a Bahamian company?
```
```text
What is an Included Entity under CESRA and what activities are relevant?
```

### DARE / digital assets
```text
What licensing requirements apply to digital asset businesses under the DARE Act?
```

### FCSP
```text
What is required to obtain a Financial and Corporate Service Provider (FCSP) licence?
```

### Off-scope (should decline)
```text
What is the US federal corporate tax rate?
```

---

## 5. Acceptance test cases

| # | Query | Expected |
|---|---|---|
| 1 | Smuggling penalties (sample above) | `grounded: true`; sources include `smuggling_of_migrants_act_2025_*.md` |
| 2 | IBC incorporation (sample above) | `grounded: true`; sources include `ibc_act_*.md` |
| 3 | CESRA substance (sample above) | `grounded: true`; sources include `cesra_2023_guidelines_*.md` |
| 4 | US federal corporate tax rate | `grounded: false` (clean decline — off-scope) |
| 5 | Omit / wrong `Authorization` | `401` |
| 6 | > 100 requests in an hour | `429` with `Retry-After` |
| 7 | `query` > 2000 chars | `400`, `error: "query_too_long"` |

### Quick curl
```bash
curl -s -X POST https://lawbey-mcp.fly.dev/debug/query \
  -H "Authorization: Bearer carta:<PARTNER_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"query":"What are the penalties for smuggling migrants in The Bahamas?"}'
```

---

## 6. Behavior notes

- **Latency:** typically **~30–60s** per query; complex questions can reach
  **~90–120s**. Budget agent timeouts accordingly (recommend **≥150s** client
  timeout).
- **How retrieval works:** multi-query rewrite (acronym expand + keyword
  distill) → knowledge-base vector search → top statute files passed into
  generation. If retrieval finds nothing, the tool returns
  `error: "rag_unavailable"` rather than an ungrounded guess.
- **Decline retry:** if the model declines (“context does not contain…”), the
  server retries once. Further retries rarely help.
- **`grounded: false`** means decline / off-scope / no usable retrieved
  context. Treat as “no answer”, not a transport error.
- **Determinism:** `temperature=0`. Identical queries may still differ slightly
  when upstream retrieval ranking varies.
- **MCP vs LawBey web UI:** do **not** treat the Open WebUI chat UI as the
  source of truth. The UI can retrieve irrelevant chunks and decline even when
  the KB has the right acts. **This MCP endpoint is what Carta should integrate
  against.**
- **Rate limits:** 100 / hour and 1000 / day per partner key (in-memory; resets
  on redeploy). Suitable for pilot / agent usage, not high-QPS blast traffic.

---

## 7. Security notes for Carta

- Send the partner key only via the `Authorization` header. Never in URLs.
- The upstream LawBey API key is held server-side only and is never
  returned to callers.
- `context` is optional and must not contain PII — it is forwarded to the
  upstream model.
- All traffic is HTTPS (Fly.io forces HTTPS).

---

## 8. Support / escalation

- Outage / `5xx`: check `GET /health` first; contact LawBey ops.
- Wrong answers / `grounded: false` on clearly in-scope questions: retry once,
  then report the query + the `sources` / `citations` returned.
- `rag_unavailable`: KB index may be rebuilding — retry later; escalate if
  persistent.
- Adding statutes to the KB: LawBey ops task (no Carta change needed).
