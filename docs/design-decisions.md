# Design Decisions & Lessons Learned

## 1. Hybrid: RAG to find, tools to compute

ERP data is mostly structured. Pure vector RAG is bad at "what is the balance of account X on date Y",
because the number has to be computed from thousands of rows, not "found". So:

- **Finding** (the right account, similar journals, the user guide) → hybrid retrieval.
- **Computing** (balances, reports) → call the ERP's existing reporting functions.

As a result, the assistant's numbers always match the ERP pages. This is tested: the general-ledger
tool's output is compared directly against the service function for the account with the most entries.

## 2. Tools call existing services

The alternative was letting the LLM write SQL. It was rejected because:

- Business rules (parent accounts can't be used, debit = credit, closed periods, automatic journal
  numbers, atomic posting) already live in the services. Duplicating them means two sources of truth.
- Free-form SQL from an LLM is hard to secure per role.

## 3. Two phases for every write

`prepare` (validation + preview, no side effects) is separate from `execute` (after the user clicks).
The validated payload (account ids resolved from codes, etc.) is stored in `AssistantAction`, so what gets
executed is exactly what the user saw on the card. Nothing is re-read from the LLM's text.

Safeguards:
- An atomic action claim (`updateMany ... where status = PENDING`) prevents duplicate journals on double clicks.
- Another user's actions cannot be confirmed.
- Proposals expire after 24 hours.
- Receipt attachments must be URLs actually uploaded in the same conversation.
- Deleting/voiding journals is deliberately **not** available via chat.

## 4. Index in a separate database

`rag_db` can be dropped and rebuilt at any time. Chat history doesn't mix with the transactional DBs,
and the production Postgres image doesn't need to be replaced for pgvector.

## 5. Local, optional embeddings

`multilingual-e5-small` (384 dimensions) runs inside the Node process. Benefits: no per-token cost, and
ERP text is not sent to an embedding service. If the model fails to load, search silently falls back to
full-text + trigram with no user-facing error. This mode is also used for tests and CI.

## 6. Account documents include the parent path

The chart of accounts has many identically named sub-accounts under different projects/teams.
Account documents are written like this:

```text
Akun 73045xxxxx BIAYA BBM KENDARAAN. Tipe: Beban/Biaya.
Induk: 7000000000 BEBAN USAHA › 7300000000 BEBAN USAHA <TEAM> › 73045xxxxx KK OPERASIONAL <TEAM> <CITY>.
Akun detail (sub akun) — bisa dipakai di baris jurnal.
```

(The indexed text is in Indonesian, the language of the ERP: "Account … Type: Expense. Parent: … Detail
account (sub-account) — can be used in journal lines.")

The system prompt requires the LLM to ask the user when more than one candidate is plausible.

## 7. Bugs found while building (and how they were caught)

| Bug | Symptom | Fix | How it was caught |
|---|---|---|---|
| Reindex pagination mixed `updatedAt` ordering with an `id` cursor | Only 2,088 of 21,014 journals indexed | Explicit pagination mode (ordered by `id`) | Counting documents per source after reindex |
| Incremental cursor used only `updatedAt` | Rows with identical timestamps (bulk imports) could be skipped at batch boundaries | `(updatedAt, id)` cursor | Code review before testing |
| OR-prefix full-text | "kas ho" → "kasbon" dominated results | AND path weighted 1.3, OR 0.6 | Manual query testing |
| Partial tool-call history | API rejected the conversation if the process died mid-turn | History sanitization + unit test | Adversarial review |
| Login rejected inventory users for `/accounting/asisten` | Assistant page wouldn't open for the Inventory role | Exception scoped to the assistant path | Per-role Playwright E2E |
| Account codes rendered right-aligned | Code column looked like amounts | Number heuristic: plain codes of ≥7 digits stay left-aligned | E2E screenshots |

## 8. Testing without LLM access

During development, the LLM API was unreachable from the build environment. The workaround:

- **Integration tests** use a mock `ChatFn` with scripted tool calls. Everything behind it (Zod
  validation, ERP services, audit, index, confirmation) runs for real against Postgres with migrated data.
- **Browser E2E** uses a small HTTP server that mimics the `/chat/completions` endpoint. The production
  LLM client is used as-is, with `LLM_BASE_URL` pointed at the mock server.

**Not yet** tested: the real model's answer quality (account selection, tone). This needs to be checked
with the team's real questions once the API key is configured.
