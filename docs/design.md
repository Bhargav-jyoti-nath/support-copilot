## Problem 
- For Acme Mobile customers who ask routine questions in chat (plans, billing, roaming, SIM), we build an assistant that answers from official help articles, handles simple account lookups, and hands over to a human when unsure, measured by answer correctness, safe behaviour and response time, under these constraints: free tools only, no real customer data, an 8 GB laptop for development.
## Users and top intents
- Primary Users: Acme Mobile retail subscribers seeking self-service support
- **Top 12 Intents:**
  1. `plans-and-pricing`: Monthly postpaid plan allowances and rates
  2. `data-addons`: One-time data boosters and validity
  3. `billing-and-late-fees`: Billing cycles, due dates, late fees, and service suspension
  4. `payment-methods`: Supported payment channels and installment agreements
  5. `roaming-policy`: Daily roaming pass pricing, supported country zones, and bill-shock limits
  6. `sim-activation`: New physical SIM delivery and activation turnaround times
  7. `esim-setup`: QR code retrieval and setup duration
  8. `number-porting`: Mobile number portability requirements, processing time, and fees
  9. `contract-and-cancellation`: Contract durations, cooling-off periods, and early termination fees
  10. `device-installments`: Device financing terms and early payoff policies
  11. `network-troubleshooting`: APN settings and self-service connectivity fixes
  12. `support-and-escalation`: SLA response targets and routing to human agents
## Success metrics
| Level | Metric | Starting target |
| :--- | :--- | :--- |
| Business (production only) | Containment: share of chats resolved without a human | Measure after launch; no target in a demo[cite: 1] |
| Business (production only) | Average handle time and CSAT | Measure with a controlled before/after comparison[cite: 1] |
| AI quality | Retrieval hit@4 (right article in the top 4) | 0.90 or higher[cite: 1] |
| AI quality | Pass rate on answerable questions | 0.80 or higher[cite: 1] |
| AI quality | Correct “I don’t know” rate on unanswerable questions | 0.90 or higher[cite: 1] |
| AI quality | Correct tool choice | 0.90 or higher[cite: 1] |
| Safety | Red-team attacks that succeed | 0[cite: 1] |
| Safety | Normal questions wrongly blocked | 5% or less[cite: 1] |
| System | p95 latency on a cloud model | 10 s or less[cite: 1] |
| System | Error rate | under 1%[cite: 1] |

## Constraints  
- free tools only, no real customer data, an 8 GB laptop for development.
## Architecture 
```text
OFFLINE (build time)
 data/kb/*.md --> chunk --> embed (bge-small) --> vector index (ChromaDB)

 ONLINE (every request)

 User --> UI / API (FastAPI)
              |
              v
   [1 INPUT GUARDS]   length, PII masking, injection patterns
              |
              v
   [2 RETRIEVE]       embed question -> top-k chunks (+ score gate)
              |
              v
   [3 AGENT HARNESS]  prompt + context -> LLM <-> tools (validated, max steps)
              |
              v
   [4 OUTPUT GUARDS]  numbers grounded? source cited? PII? promises?
              |
              v
   [5 TRACE + LOG]    one JSON record per request
              |
              v
 Answer + sources --> User

 Underneath: the LLM sits behind one OpenAI-compatible interface
             (Ollama on your laptop, a free cloud API in production)
 Alongside:  EVALS (golden set, red-team set) run offline and in CI
             against the same pipeline
```
## Risks 
| Failure mode | Effect | S | O | D | RPN | Controls (phase) |
| :--- | :--- | :---: | :---: | :---: | :---: | :--- |
| Hallucinated price or policy | Customer acts on wrong information | 8 | 6 | 5 | 240 | Grounded prompt, numeric check, evals (3, 4, 6)[cite: 1] |
| Prompt injection in user text or retrieved text | Rules bypassed, data leaked | 8 | 5 | 6 | 240 | Input guard, least-privilege tools, provenance rule (5, 6)[cite: 1] |
| Stale knowledge base | Outdated policy quoted | 6 | 5 | 6 | 180 | Versioned KB, re-index process, review cadence (10)[cite: 1] |
| Wrong-customer data exposure | Privacy breach | 10 | 3 | 5 | 150 | Identity from the session, never from the model; tests (5)[cite: 1] |
| Retrieval miss | Empty or confidently wrong answer | 6 | 6 | 4 | 144 | hit@k eval, hybrid search (4)[cite: 1] |
| PII in logs | Compliance breach | 8 | 4 | 4 | 128 | Mask before logging (6, 7)[cite: 1] |
| Provider outage or rate limit | No answers | 5 | 6 | 2 | 60 | Retries, fallback model, graceful error (2, 9)[cite: 1] |
## Decisions 
- [ADR-001: Start with RAG, add narrow tools, no multi-agent](adr-001-autonomy.md)
## Open questions
- What similarity score cutoff (MIN_SCORE) minimizes false abstentions while filtering out irrelevant questions?
- Does BM25 hybrid search significantly outperform pure vector search on telecom terms?
- Can local 1B/3B models reliably output valid tool arguments within a 4-step limit?
