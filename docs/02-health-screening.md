# Workflow 2: AI Pet Health & Risk Screening

## The Business Problem

When a pet's health or behavior information comes in, nobody at Pawsome
Stays is systematically reviewing it before the pet is approved for a stay.
A pet with a real safety concern — aggression history, medication with
strict timing, a serious allergy — could otherwise slip through without
staff ever being alerted to take extra precautions. Manually reading every
submission doesn't scale, and it's inconsistent from one staff member to
another.

This workflow automates the first-pass judgment call: is this pet safe to
approve automatically, or does a real person need to look at this before the
stay is confirmed?

## What This Workflow Does

1. **Receives a health submission** — a webhook simulating a pet health/
   behavior form (pet name + health notes) being submitted or updated.
2. **Looks up the pet** in Airtable to resolve its record and confirm it
   exists on file.
3. **Sends the health notes to an AI model (Groq, `llama-3.3-70b-versatile`)**
   with a strict system prompt instructing it to classify risk as Low,
   Medium, or High, with a short reason, returned as JSON only.
4. **Parses the AI's response defensively** — the raw response is just text,
   and AI output isn't always perfectly well-formed, so this step validates
   the JSON structure and the allowed risk values before trusting it.
5. **Branches on the outcome:**
   - **Parsing failed** → defaults to `"Pending Review"`, routed to manual
     review. The workflow never assumes success on ambiguous output.
   - **Parsed successfully, Low risk** → auto-approved, `Risk Level` updated
     directly in Airtable.
   - **Parsed successfully, Medium/High risk** → flagged for manual review,
     `Risk Level` updated to reflect the AI's assessment.
6. **Notifies a manager by email** whenever a pet is flagged for review,
   including the AI's risk level, its stated reasoning, and the original
   health notes submitted — so the reviewer has full context, not just a
   generic alert.

## Architecture

_[Insert screenshot of the full n8n canvas here]_

```
[Webhook] → [Find Pet] → [IF: Pet Found?]
                             ├─ True → [Groq Risk Classification] → [Parse AI Response] → [IF: Parse Succeeded?]
                             │                                                                ├─ True → [IF: Is Low Risk?]
                             │                                                                │             ├─ True  → [Auto-Approve Pet]
                             │                                                                │             └─ False → [Flag for Manual Review] → [Notify Manager]
                             │                                                                └─ False → [Flag for Manual Review] (shared)
                             └─ False → [pet not found — error handling, pattern shared with Workflow 1]
```

## Key Design Decisions

**AI failure never defaults to approval.**
This is the central safety principle of the workflow. If the AI call fails,
times out, or returns text that doesn't match the expected JSON shape, the
pet is **not** auto-approved — it's routed to `"Pending Review"` and flagged
for a human. The only way a pet gets auto-approved is if the AI explicitly
and successfully returned `"Low"`. Uncertainty always resolves toward caution,
never convenience.

**AI output is parsed defensively, not trusted blindly.**
The AI is instructed (via the system prompt) to return only strict JSON, but
real model output isn't guaranteed to comply every time. The parsing step
wraps `JSON.parse` in a try/catch and additionally checks that the returned
`riskLevel` is one of the three valid values — a model could return
syntactically valid JSON with an invalid or hallucinated risk label, and this
step catches that too, not just outright parse failures.

**Both "needs review" branches share one path.**
A Medium/High classification and a parse failure both ultimately mean the
same thing operationally: "a human needs to look at this." Rather than
duplicating the Airtable update and notification logic twice, both branches
converge into one shared `Flag for Manual Review` → `Notify Manager` path,
each carrying its own `riskLevel` and `reason` values into the same nodes.
This mirrors the shared-response pattern used in Workflow 1.

**Low temperature (0.2) for the AI call.**
Since this is a classification task where consistency matters more than
variety, a low temperature setting was used to make the model's risk
judgments more deterministic and repeatable across similar inputs.

**A known limitation: risk classification boundaries are fuzzy.**
During testing, near-identical framings of a "mild allergy" case were
classified differently across runs — once as Low, once as Medium, once as
High — depending on how the consequences were worded (e.g. adding "could
require an emergency vet visit" pushed a case from Low to High). This is an
honest characteristic of LLM-based classification, not a bug: the model is
sensitive to phrasing and severity language. In a real deployment, this is
exactly why Medium/High (and failed-parse) cases are *never* auto-approved —
the human review step exists specifically to catch and correct this kind of
ambiguity.

## Error Handling

- **AI call or parsing failure**: caught explicitly by the `Parse AI
  Response` code node's try/catch, with a safe default (`"Pending Review"`)
  that always routes to manual review rather than silently failing open.
- **Pet not found**: same pattern as Workflow 1 — the original request is
  preserved for follow-up rather than dropped.
- **Infrastructure/API failures** (e.g. Airtable or Groq unreachable):
  handled by the shared, centralized Error Workflow used across this
  project. _[Link to Error Workflow docs once built]_

## Tech Stack

- **n8n** (self-hosted)
- **Airtable** — `Pets` table (health notes source + risk level storage)
- **Groq** (`llama-3.3-70b-versatile`) — AI risk classification, called via
  a generic HTTP Request node (no dedicated n8n node exists for Groq, so
  its OpenAI-compatible REST API is called directly)
- **Gmail** (OAuth2) — manager notification emails

## Example Payloads

**Incoming webhook request:**
```json
{
  "petName": "Rocky",
  "healthNotes": "History of biting staff during previous stay, requires muzzle for handling, on daily seizure medication that must be given at exact times, will attack other dogs on sight and must be kept fully isolated at all times"
}
```

**Raw AI response (`choices[0].message.content`):**
```json
{"riskLevel": "High", "reason": "Pet has a history of biting staff and requires special handling and isolation procedures."}
```

**Parsed and normalized (output of `Parse AI Response`):**
```json
{
  "riskLevel": "High",
  "reason": "Pet has a history of biting staff and requires special handling and isolation procedures.",
  "parseSucceeded": true
}
```

