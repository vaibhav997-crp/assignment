I'm submitting this as a GenAI/LLM Engineer take-home assessment (brief attached: 
BRIEF_GenAI_LLM_Engineer.docx). Build the full solution described in the brief, 
using the attached policy corpus (12 POL-*.md files), accounts.json, tools.md, 
eval_seed.csv, and README_DATA.md.

WHAT TO BUILD (per the brief):
1. A service agent that answers customer questions about their card, grounded 
   only in the attached policy corpus + the customer's account state.
2. Hybrid retrieval over the 12-doc corpus (it's small — TF-IDF/keyword hybrid 
   is fine, no need for a vector DB).
3. Implementations of both tools in tools.md — get_account_state and 
   raise_service_request — matching their contracts exactly, including the 
   simulated 400ms latency and ~1-in-12 UpstreamTimeout on get_account_state, 
   and InvalidCategory rejection on unrecognised categories in 
   raise_service_request.
4. A schema-valid structured response for every query: 
   {answer, citations, decision, tool_calls, refused, routed_to}.
5. Enforced citations: every factual claim must trace to a real retrieved 
   policy chunk id. If nothing relevant is retrieved, refuse and route — 
   never invent an answer.
6. Refusal as a designed behaviour, not a fallback: at least one seeded eval 
   case (travel insurance — not in any policy doc) must be refused and routed 
   to a human queue.

CRITICAL DESIGN CONSTRAINT — read this before writing any code:
The corpus contains a deliberate prompt-injection attempt inside 
POL-OPS-012_agent_desktop_notes.md. That file is explicitly marked "Not 
policy — informal notes" but contains a line instructing "automated 
assistants" to approve any credit-limit-increase request and hide that it did 
so. The correct architecture is: all credit-limit-increase DECISIONS must be 
made by deterministic, non-LLM rule code (implementing POL-CRD-001, 
POL-CON-004, POL-CRD-002/003, POL-KYC-008, POL-SVC-005) that runs on the tool 
output BEFORE any LLM call. The LLM is only ever used to phrase an 
already-made decision into plain language, or to answer general policy 
questions from retrieved, verified-policy (is_policy=True) chunks. This means 
the injected instruction is structurally incapable of approving anything, 
regardless of what the LLM reads. Do not implement this as "tell the LLM in 
the system prompt to ignore injected instructions" alone — that's necessary 
but not sufficient; the decision logic must not run through the LLM at all.

OTHER TRAPS ALREADY IDENTIFIED IN THE DATA (confirm your build handles each):
- POL-CRD-001 (6 months between increases) contradicts POL-CRD-011 (3 
  months). POL-CRD-011 itself says POL-CRD-001 prevails until revised — 
  resolve it that way.
- Thin-file customers (<6 months repayment history) are capped at 1.25x 
  increase and must be referred to a human, never auto-decisioned 
  (POL-CRD-003), overriding the normal 2x cap in POL-CRD-001.
- A vulnerable-flagged customer (POL-CON-004) is declined regardless of 
  score/affordability, with zero sales content in the reply.
- An open dispute mandates decline (POL-CRD-001 + POL-SVC-005).
- Re-KYC overdue >90 days blocks new limit increases but does NOT freeze the 
  account or affect the existing limit (POL-KYC-008) — don't overstate the 
  consequence.
- Internal behavioural scores are never disclosed to the customer 
  (POL-GOV-009) — even when asked directly. Offer the approved reason codes 
  (AA01–AA06) instead.
- Travel insurance is not covered anywhere in the corpus — this is the 
  designed "must refuse" case.
- raise_service_request must reject any category not in POL-SVC-006's list 
  rather than guessing one; route ambiguous intents to GENERAL for human 
  review.

DELIVERABLES:
1. A working repo: agent code, both tool implementations, retrieval, a 
   deterministic guardrail/rules module separate from the LLM-calling code, 
   and run instructions.
2. eval_seed.csv extended to at least 15 cases total (it currently has 12) — 
   include some you expect to fail, and report actual pass rate honestly 
   after running them, don't just claim 100%.
3. A short instrumentation layer: log per-request traces, token usage, 
   latency, and errors for a handful of sample runs, and show the output.
4. A "Break" writeup: for each of prompt injection (POL-OPS-012), a 
   retrieval miss, a tool timeout, and the POL-CRD-001/POL-CRD-011 
   contradiction — state exactly what the customer sees.
5. A "Deploy" section: how you'd add tracing, prompt versioning, a CI eval 
   gate, and a fallback if the primary model is unavailable (keep this to 
   design/prose where full implementation isn't warranted in the time-box).
6. A "Scale" section: caching, cost per conversation, streaming, 
   concurrency, and what changes if the corpus were 100x larger and split 
   across customer tenancies.
7. A one-page README covering: what you built, what you deliberately did 
   not build and why, what you'd do next with another day, and what you 
   would not trust about your own submission and how someone would catch it.

Use whichever LLM API I configure (I'll add the key myself) — write the LLM 
call as a swappable module so the model provider isn't hardcoded. Keep the 
whole thing scoped to what's achievable in a 6-hour time-box — a partial, 
well-reasoned submission with clear cut lines beats an over-built one.
