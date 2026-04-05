# DocuBot Model Card

This model card reflects the DocuBot system after implementing retrieval and testing all three modes.

---

## 1. System Overview

**What is DocuBot trying to do?**

DocuBot is a documentation assistant that answers developer questions about a codebase by searching local markdown files. It supports three answer modes so developers can compare how grounding affects answer quality and reliability.

**What inputs does DocuBot take?**

DocuBot takes a natural language question from the developer and reads `.md` and `.txt` files from the `docs/` folder. Modes 1 and 3 also require a `GEMINI_API_KEY` environment variable to call the Gemini language model.

**What outputs does DocuBot produce?**

Mode 1 produces a free-form LLM answer with no connection to the actual project docs. Mode 2 returns raw text sections from the most relevant documents. Mode 3 returns a concise LLM-generated answer that cites only the retrieved sections. All modes return a refusal message when no relevant content is found.

---

## 2. Retrieval Design

**How does your retrieval system work?**

Each document is split into sections by markdown headings before any scoring happens. Each section is then scored by counting how many times each non-stop-word query term appears in that section text. The top three sections with a score of at least 2 are returned. If nothing reaches that threshold, retrieval returns an empty list and the system refuses to answer.

**What tradeoffs did you make?**

The system is simple and easy to reason about, but it only matches exact words. It cannot handle synonyms or paraphrasing. A query like "how do I connect to the database" may miss sections that use the word "configuration" instead of "connect." Heading-based splitting respects document structure but produces uneven section lengths, so a long section is still returned in full even if only one sentence is relevant.

---

## 3. Use of the LLM (Gemini)

**When does DocuBot call the LLM and when does it not?**

Naive LLM mode always calls Gemini with only the developer question and no project docs attached. The model answers from general training knowledge. Retrieval only mode never calls the LLM and returns raw section text directly. RAG mode calls retrieval first, then passes the top sections and the question to Gemini. If retrieval returns nothing, the LLM is never called at all.

**What instructions do you give the LLM to keep it grounded?**

The RAG prompt tells Gemini to use only the information in the provided snippets, not to invent functions, endpoints, or configuration values that are not in the snippets, to reply exactly "I do not know based on the docs I have." when snippets are not sufficient, and to mention which files it relied on when it does answer.

---

## 4. Experiments and Comparisons

The same three questions were tested across all three modes.

| Query | Naive LLM | Retrieval Only | RAG | Notes |
|-------|-----------|----------------|-----|-------|
| Where is the auth token generated? | Harmful. Gave a long generic answer about JWT servers in general, not about this codebase at all. | Helpful. Returned the exact Token Generation section from AUTH.md as the top result. | Most helpful. Gave a one-sentence answer citing `generate_access_token` in `auth_utils.py` from AUTH.md. | RAG wins here. Retrieval was accurate and RAG turned it into a readable answer. |
| How do I connect to the database? | Harmful. Returned hundreds of lines about PostgreSQL, MongoDB, and Java JDBC with no connection to this project. | Partially helpful. The third snippet from DATABASE.md was relevant, but the top two results were from AUTH.md and API_REFERENCE.md, which had nothing to do with the database. | Safe refusal. Said "I do not know based on the docs I have." because the retrieved snippets did not actually answer the question. | Retrieval failed for this query. RAG's refusal was the safest outcome even though it means the question went unanswered. |
| Which endpoint lists all users? | Risky. Guessed that the answer was probably `GET /api/users` or `GET /users`, which happens to be correct, but this was a guess based on REST conventions, not the actual docs. | Missed the answer. Returned the install dependencies section from SETUP.md as the top result, then a generic API_REFERENCE header, then database query helpers. The actual `GET /api/users` section was not retrieved. | Safe refusal. Said "I do not know based on the docs I have." because the retrieved snippets did not contain the endpoint. | The retrieval scoring failed to find the right section. RAG refused safely, but the correct answer exists in the docs and was not found. |

**What patterns did you notice?**

Naive LLM looks impressive because its answers are long, well-formatted, and confident, but every answer was about generic software concepts instead of this specific project. This is the most dangerous mode because it sounds authoritative while being completely ungrounded.

Retrieval only is the most transparent mode. When retrieval works well, as it did for the auth token question, the raw section text contains the exact answer. When retrieval fails, the wrong sections are returned and the user has to notice that themselves, which is easy to miss.

RAG combines the best of both when retrieval works. It produces a short, readable answer and cites the source file. When retrieval fails, RAG refuses rather than hallucinating, which is the safest possible behavior. However, RAG depends entirely on retrieval quality. If retrieval returns the wrong sections, RAG either refuses or answers the wrong question.

---

## 5. Failure Cases and Guardrails

**Describe at least two concrete failure cases you observed.**

Failure case 1: The query "How do I connect to the database?" returned AUTH.md's Token Generation section as the top result. The words "the" and other overlapping terms caused unrelated sections to outscore the relevant DATABASE.md section. The system returned three snippets, only one of which was relevant, and RAG correctly refused because the snippets were not enough to answer confidently.

Failure case 2: The query "Which endpoint lists all users?" completely missed the `GET /api/users` section in API_REFERENCE.md. The word "lists" matched a section about database query helpers and the word "endpoint" matched the generic API Reference header instead. The correct answer existed in the docs but was never retrieved.

**When should DocuBot say "I do not know based on the docs I have"?**

DocuBot should refuse when the query is about a topic that does not appear in the documentation, such as payment processing or deployment pipelines not described in the docs. It should also refuse when no section scores above the minimum threshold, meaning the match is too weak to trust. These two situations both occurred during testing and the guardrail handled them correctly.

**What guardrails did you implement?**

A score threshold of 2 requires that at least two meaningful query terms appear in a section before it is returned. Retrieval returns an empty list when nothing meets this threshold. Both answer modes check for an empty list before doing anything else and return the refusal message immediately. The RAG prompt also instructs Gemini to refuse when the snippets are not sufficient, which provides a second layer of safety after retrieval.

---

## 6. Limitations and Future Improvements

**Current limitations**

The retrieval system only matches exact words. If the query uses a different word than the document uses, the relevant section may never be found. This happened with "connect to the database" failing to rank the DATABASE.md connection section highly.

The scoring counts word frequency across the entire section. A long, general section that mentions query words several times can outscore a short, focused section that answers the question directly. This caused SETUP.md to appear in results for unrelated queries.

RAG can only be as good as retrieval. When retrieval returns wrong sections, RAG either refuses or has bad evidence to work with. There is no way for the LLM to reach into the docs itself when retrieval misses.

**Future improvements**

Replacing word frequency scoring with vector similarity would allow the system to match "connect" with "configuration" and "endpoint" with "route," which would fix most of the retrieval failures observed during testing.

Breaking sections into smaller paragraph-level chunks would prevent long, loosely relevant sections from ranking above short, precise ones. This would especially help queries like "which endpoint lists all users" where the answer is a single heading and two lines.

Adding a feedback signal when the LLM refuses would help identify which queries the retrieval system consistently fails on so those patterns can be fixed systematically.

---

## 7. Responsible Use

**Where could this system cause real world harm if used carelessly?**

The naive LLM mode is the biggest risk. It generates confident, detailed answers that have no connection to the actual codebase. A developer following that output could use incorrect environment variable names, configure security settings wrong, or call endpoints that do not exist. In an onboarding context this could cause real security misconfiguration without the developer realizing the answer was fabricated.

Retrieval only mode can also mislead a user who does not read carefully. When wrong sections are returned, as happened with the database query, the results look plausible but do not answer the question.

**What instructions would you give real developers who want to use DocuBot safely?**

Never use naive LLM mode for setup instructions, security configuration, or any question where an incorrect answer would cause real problems. Always use RAG or retrieval only for those cases.

Treat retrieval only output as raw evidence that still needs to be evaluated. Read the full section and confirm it actually answers your question before acting on it.

Keep the docs folder up to date. DocuBot can only be as accurate as the documentation it indexes. An outdated doc will produce outdated answers with no warning.

When DocuBot says it does not know, trust that response and look at the source files directly rather than rephrasing the question to force an answer.

---
