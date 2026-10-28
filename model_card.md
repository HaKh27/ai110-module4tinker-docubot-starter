# DocuBot Model Card

This model card is a short reflection on your DocuBot system. Fill it out after you have implemented retrieval and experimented with all three modes:

1. Naive LLM over full docs  
2. Retrieval only  
3. RAG (retrieval plus LLM)

Use clear, honest descriptions. It is fine if your system is imperfect.

---

## 1. System Overview

**What is DocuBot trying to do?**  
Describe the overall goal in 2 to 3 sentences.

> DocuBot answers developer questions about a sample codebase using only the project's own documentation (`docs/AUTH.md`, `API_REFERENCE.md`, `DATABASE.md`, `SETUP.md`). It supports three modes so we could compare a plain LLM, a non-LLM retrieval system, and a retrieval+LLM (RAG) pipeline on the same questions, to see where grounding in real docs actually changes the answer.

**What inputs does DocuBot take?**  
For example: user question, docs in folder, environment variables.

> A natural-language developer question (typed at the CLI prompt or pulled from `dataset.py`'s `SAMPLE_QUERIES`), the `.md`/`.txt` files in the `docs/` folder, and, for the LLM-backed modes, the `GEMINI_API_KEY` environment variable used to authenticate the Gemini client.

**What outputs does DocuBot produce?**

> Depending on mode: (1) naive mode returns raw Gemini text generated with no document context; (2) retrieval-only mode returns the raw matching snippets with their source filenames, or `"I do not know based on these docs."` if nothing matches; (3) RAG mode returns a Gemini-generated answer grounded in the retrieved snippets, or `"I do not know based on the docs I have."` if the snippets are empty or insufficient.

---

## 2. Retrieval Design

**How does your retrieval system work?**  
Describe your choices for indexing and scoring.

- How do you turn documents into an index?
- How do you score relevance for a query?
- How do you choose top snippets?

> `build_index()` builds a word → filenames inverted index (lowercased, punctuation-stripped tokens), but the actual `retrieve()` method doesn't use it — it brute-force scans every document, split into paragraphs on blank lines (`\n\n`). For each paragraph, `score_document()` lowercases the query, strips punctuation, **drops stopwords** (added this session — "the", "who's", "how", etc. — since they inflated scores without being real signal), and sums how many times each remaining query word appears as a substring in the paragraph. Paragraphs scoring at or above `MIN_RELEVANCE_SCORE` (1, after today's fix — see Section 5) are sorted by score and the top `top_k` (default 3) are returned.

**What tradeoffs did you make?**  
For example: speed vs precision, simplicity vs accuracy.

> Simplicity over precision throughout: substring counting instead of stemming/embeddings means "failing" never matches "Failure" and "codes" never matches "401/403/404". Paragraph-level chunking (splitting on blank lines) is simple but splits markdown headings from the body text that follows them, so related content can land in separate chunks and never combine into a single high-scoring match. We chose a low, stopword-filtered threshold (favoring recall) rather than a stricter one, since the RAG prompt itself is responsible for refusing to answer when the retrieved evidence is weak — better to over-retrieve and let the LLM's grounding rules filter it than to under-retrieve and dead-end at "I do not know" before the LLM even sees relevant text.

---

## 3. Use of the LLM (Gemini)

**When does DocuBot call the LLM and when does it not?**  
Briefly describe how each mode behaves.

- Naive LLM mode:
- Retrieval only mode:
- RAG mode:

> - **Naive LLM mode:** Always calls Gemini, and ignores the docs entirely — `naive_answer_over_full_docs()` even ignores the `all_text` argument it's given and just sends the raw question with a generic "You are a documentation assistant" prompt.
> - **Retrieval only mode:** Never calls Gemini. `answer_retrieval_only()` returns the raw retrieved snippets (or the refusal string) with no generation step at all.
> - **RAG mode:** Calls Gemini only if `retrieve()` found at least one snippet; if retrieval comes back empty, `answer_rag()` short-circuits to `"I do not know based on the docs I have."` without ever calling the LLM. Otherwise it sends the snippets plus the question to `answer_from_snippets()`.

**What instructions do you give the LLM to keep it grounded?**  
Summarize the rules from your prompt. For example: only use snippets, say "I do not know" when needed, cite files.

> From `answer_from_snippets()`'s prompt in `llm_client.py`: use only the information in the provided snippets; do not invent functions, endpoints, or configuration values; if the snippets aren't enough to answer confidently, reply with the exact refusal string `"I do not know based on the docs I have."`; and when it does answer, briefly name which files it relied on. The naive mode prompt has no such rules — it has no grounding instructions at all, which is the point of using it as the ungrounded baseline.

---

## 4. Experiments and Comparisons

Run the **same set of queries** in all three modes. Fill in the table with short notes.

You can reuse or adapt the queries from `dataset.py`.

| Query | Naive LLM: helpful or harmful? | Retrieval only: helpful or harmful? | RAG: helpful or harmful? | Notes |
|------|---------------------------------|--------------------------------------|---------------------------|-------|
| Where is the auth token generated? | Harmful — asked us to supply our own framework/code instead of answering | Helpful — surfaced the exact `generate_access_token` / `auth_utils.py` line | Helpful — correctly named `generate_access_token` in `auth_utils.py`, cited AUTH.md | Best case: retrieved snippet directly contained the answer |
| How do I connect to the database? | Harmful — invented generic `DB_HOST`/`DB_PORT`/`DB_USER` env vars that don't exist in this project | Helpful — returned the real `DATABASE_URL` variable from DATABASE.md/SETUP.md | Helpful — correctly explained `DATABASE_URL`, SQLite fallback, and Postgres note, with citations | Naive mode fabricated plausible-sounding but wrong config |
| Which endpoint lists all users? | Harmful-ish — guessed `GET /users`, close but wrong path (`GET /api/users`) | Partially helpful — top-3 snippets mention `get_all_users()` and "admins only" but never show the literal path | Harmful (refusal) — said "I do not know" even though the real endpoint existed further down the ranked list, outside `top_k=3` | Right doc, wrong chunk ranked highest — see Section 5 |
| How does a client refresh an access token? | Harmful — gave a generic OAuth2 refresh-token flow, not this app's actual simple `/api/refresh` call | Helpful — top result describes token-based auth and `generate_access_token`, though not the refresh step itself | Harmful (refusal) — said "I do not know"; the actual line (`Refresh the token... by calling /api/refresh`) ranked 8th, outside `top_k=3` | Same ranking/chunking gap as above |
| Who's the president? (guardrail test) | Harmful — answered with real-world political info, totally out of scope | Helpful — correctly returned "I do not know based on these docs" (after fixing the stopword-scoring bug) | Helpful — refused, as expected | Before the fix, retrieval returned unrelated AUTH/SETUP snippets because "the" alone cleared the relevance threshold |
| What are the error codes for failing cases? | Harmful — produced generic HTTP status code advice unrelated to this app | Partially helpful — after lowering the threshold, returns real section headings ("Failure Cases:", "## Error Formats") but not the actual 401/403/404 bullets | Not tested after the chunking gap was identified (left unfixed by choice) | Headings and their bullet lists get split into separate chunks, so neither alone reaches 2+ keyword matches |

**What patterns did you notice?**  

- When does naive LLM look impressive but untrustworthy?  
- When is retrieval only clearly better?  
- When is RAG clearly better than both?

> Naive LLM mode is most dangerous exactly when it sounds most confident: for the database and token-refresh questions it produced detailed, well-formatted, plausible-sounding answers (generic env var names, a textbook OAuth2 flow) that are simply wrong for this specific codebase, with no signal to the user that it's guessing. Retrieval-only is clearly better whenever the literal answer lives in a single paragraph in the docs (the auth-token and database questions) — it's not fluent, but it never fabricates. RAG is clearly better than both when retrieval surfaces the right paragraph, since it adds fluency and file citations on top of real grounding. But RAG is only as good as what gets retrieved: when the real answer was split across chunks or ranked below `top_k`, RAG's refusal rule made it *more* conservative than retrieval-only, which at least surfaced partial, related snippets a human could still read and use.

---

## 5. Failure Cases and Guardrails

**Describe at least two concrete failure cases you observed.**  
For each one, say:

- What was the question?  
- What did the system do?  
- What should have happened instead?

> **Failure case 1 — guardrail too loose (false positive).** Question: "Who's the president?" The system returned unrelated AUTH.md and SETUP.md snippets instead of refusing. Cause: `score_document()` counted every query word, including stopwords like "the", as a substring match; "the" appears many times in almost any paragraph, so the `MIN_RELEVANCE_SCORE = 2` threshold was satisfied by stopword noise alone, with zero actual topical overlap. It should have returned "I do not know based on these docs" immediately, since no document mentions a president. Fixed by filtering stopwords out of the query before scoring.

> **Failure case 2 — guardrail too strict, then under-informative (false negative, then partial fix only).** Question: "What are the error codes for failing cases?" The system answered "I do not know based on these docs" even though `API_REFERENCE.md` lists exactly this (401/403/404 under "Failure Cases:"). Cause: paragraphs are chunked on blank lines, which splits markdown headings from the content below them, so "Failure Cases:" (1 keyword match) and the bullet list underneath it (0 keyword matches, since "error"/"codes"/"failing" don't literally appear in the bullets) never combine in one chunk to reach `MIN_RELEVANCE_SCORE = 2`. Lowering the threshold to 1 stopped the outright refusal, but the top-ranked chunks returned are still just bare headings, not the actual codes — the deeper chunking issue was identified but intentionally left unfixed this session.

**When should DocuBot say “I do not know based on the docs I have”?**  
Give at least two specific situations.

> 1. The question is genuinely out of scope for the docs (e.g. "Who's the president?") — there is no topical overlap with any document, and refusing is the correct, safe answer. 2. Retrieval finds snippets, but none of them actually contain the specific fact asked for (e.g. a refresh-token question where the snippet only covers token *creation*, not *refresh*) — here refusing is safer than letting the LLM infer or extrapolate past what the snippets actually say, even though it means missing an answer that technically exists elsewhere in the docs.

**What guardrails did you implement?**  
Examples: refusal rules, thresholds, limits on snippets, safe defaults.

> - A stopword filter in `score_document()` so common words like "the", "who's", "how" can't by themselves push a paragraph over the relevance threshold.
> - `MIN_RELEVANCE_SCORE` in `retrieve()` (now 1, down from 2) as a minimum-relevance cutoff before a paragraph is considered a candidate snippet at all.
> - `top_k` (default 3) caps how many snippets are passed to the LLM, keeping the prompt small and focused.
> - An explicit "I do not know based on these docs"/"...docs I have" refusal string used by both retrieval-only (when `retrieve()` returns nothing) and RAG (short-circuits before calling the LLM if there are no snippets, and is also instructed inside the prompt to use that exact refusal when the snippets are insufficient).
> - The RAG prompt restricts Gemini to only the retrieved snippets and forbids inventing endpoints, functions, or config values.

---

## 6. Limitations and Future Improvements

**Current limitations**  
List at least three limitations of your DocuBot system.

1. Pure substring matching with no stemming or synonyms — "failing" never matches "Failure", "codes" never matches literal numeric codes like "401", so relevant content can be invisible to the scorer even when it's exactly what the user needs.
2. Chunking on blank lines separates markdown headings from the content that follows them, so a heading and its own bullet list can end up scored as unrelated, weakly-matching paragraphs instead of one strong match.
3. Fixed `top_k` (3) with no score-aware cutoff — a correct answer ranked 4th or lower (as happened with the refresh-token and "list all users" queries) is silently dropped, and RAG then refuses even though the docs actually contain the answer.
4. The inverted index built in `build_index()` is never actually used by `retrieve()`, which instead does a full brute-force scan of every paragraph on every query — wasted work with no retrieval benefit.

**Future improvements**  
List two or three changes that would most improve reliability or usefulness.

1. Chunk by markdown section (heading + the content under it) instead of by blank line, so headings and their bullets/body stay together as one scorable unit.
2. Add basic stemming (or swap to embedding-based similarity) so morphological variants like "failing"/"failure" or "codes"/"401" can match.
3. Make `top_k` or the score cutoff adaptive — e.g., include all paragraphs within some margin of the top score, rather than a hard-coded 3, so a correct but lower-ranked chunk doesn't get silently dropped.

---

## 7. Responsible Use

**Where could this system cause real world harm if used carelessly?**  
Think about wrong answers, missing information, or over trusting the LLM.

> The clearest real example from today: asked how to refresh an access token, naive LLM mode confidently described a generic OAuth2 refresh-token flow that doesn't match this codebase at all (the real mechanism is just calling `/api/refresh`). A developer who trusted that answer without checking could implement an incompatible auth flow, waste time debugging against a spec that was never real, or build something with different security properties than the actual system has (e.g. assuming refresh tokens exist when `AUTH_SECRET_KEY`-based short-lived tokens are what's actually used). More generally: any ungrounded or over-confident answer about authentication, permissions, or database connection details could lead to insecure configuration (e.g. inventing env vars, missing that `AUTH_SECRET_KEY` must be set) or broken integrations if a developer copies it verbatim without checking the docs.

**What instructions would you give real developers who want to use DocuBot safely?**  
Write 2 to 4 short bullet points.

- Never trust naive LLM mode for anything project-specific — it has zero grounding and will fabricate confidently (e.g. invented `DB_HOST`/`DB_PORT` env vars that don't exist in this project). Use it only as a sanity check against general-knowledge conventions, never as the source of truth.
- Treat a RAG "I do not know" as "retrieval didn't find it," not "the docs don't have it" — as seen with the refresh-token and "list all users" queries, the real answer can exist in the docs but rank below `top_k` and never reach the LLM. If RAG refuses, try retrieval-only mode and skim more snippets yourself before concluding the docs are silent.
- Always check the cited filename(s) in a RAG answer against the actual doc before acting on it, especially for anything security- or config-related (auth secrets, database URLs, permission checks).
- If you change or add documentation, re-run the sample queries in all three modes afterward — small edits to heading/paragraph structure can silently change which chunks score well enough to be retrieved.

---
