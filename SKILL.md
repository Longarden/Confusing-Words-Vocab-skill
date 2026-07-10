---
name: vocab
description: Personal concept glossary — captures terms you didn't know, written in YOUR OWN understanding (not dictionary copy-paste), into a markdown store, and republishes the whole glossary as a fixed-URL Artifact page. Triggers on "add to my vocab", "vocab this", "add this term to my glossary", "show my vocab", "단어장에 추가해", "몰랐던 용어 정리해", "단어장 보여줘", "/vocab". Also load this skill when the user has just learned a new concept or term in conversation, to offer saving it to their glossary.
---

# vocab — a glossary of what *you* didn't know, in your own words

Concepts you learn in AI conversations evaporate when the session ends. This skill captures the terms the user didn't know — recorded as **the user's own final understanding** (generation effect: self-reconstructed definitions stick far better than copied ones) — into a markdown store, and republishes the entire glossary as one self-contained Artifact web page whose **URL never changes**, so the user can bookmark it once and browse it anywhere.

## config.json (created on first run, never committed)

| key | lifecycle |
|-----|-----------|
| `storePath` | Ask the user on first run. Offer two candidates: a file inside their notes vault (e.g. Obsidian) if they mention one, or `~/vocab.md`. Do not try to auto-detect vaults. Korean users may prefer `단어장.md`. |
| `artifactHtmlPath` | If missing or `null`, default to `<this skill's folder>/vocab-glossary.html` and write that value into config. |
| `artifactUrl` | Create as `null`. After the FIRST successful Artifact publish, save the returned URL here. |

`config.example.json` in this repo shows the schema. `config.json` itself is personal and gitignored.

## Trigger policy (proposal mode)

Besides explicit calls, **offer proactively**: when the user asks about a concept they don't know and reaches an understanding, offer once — in the user's language — at a natural pause: "Want me to add ○○ to your vocab?" ("방금 그 개념 단어장에 넣을까?").
- If declined, do not re-offer the same term this session.
- Never interrupt mid-task; offer at paragraph/session boundaries.
- Never add silently. Writes without an offer happen only on explicit user calls.

## Steps

### A. Add ("add to my vocab", "단어장에 추가해")

1. Resolve `storePath` from config (first-run flow above).
2. Scope: if the user names specific term(s) ("add X to my vocab"), add only those; otherwise scan the whole session. Extract **concept/terminology questions only** (skip code-implementation Q&A). Per entry:
   - **definition**: one textbook-style sentence — concise and accurate.
   - **my-understanding**: the user's own words first — if the user reconstructed it ("so it's basically ~, right?"), use that sentence. If they never restated it, Claude writes it and appends `(summarized)` (Korean: `(요약)`).
   - **example**: one concrete example; prefer one that actually came up in the session over a generic one.
   - **confusion**: what the user misunderstood or struggled with, if anything (works as an error-notebook). Omit the field entirely when none.
3. Append to the store using the entry format below. **Duplicate/synonym terms: merge into the existing entry** — replace my-understanding with the latest understanding, update the date, and remove a previous confusion if this session resolved it (keep if not). Duplicates = same term, spelling variants, translations, or obvious synonyms of an existing entry; when unsure whether two terms are the same concept, ask the user instead of guessing. For `subject`, reuse an existing subject from the store whenever one fits — invent a new slug only for a genuinely new area (prevents tag fragmentation that degrades the filter).
4. Republish the Artifact (Step C).
5. Recap in chat: store absolute path + added/updated terms + Artifact URL.

### B. Show ("show my vocab", "단어장 보여줘")

Default: republish the Artifact (Step C) AND print a short table (term × subject × date) in the terminal. Skip the republish only if the user asks for a terminal-only view.

### C. Artifact republish

1. Load the `artifact-design` skill first if it is available; skip the load (do not fail) if it is not.
2. Parse the whole store and generate a single self-contained HTML glossary page:
   - Top: live search input (filters across term, definition, my-understanding, example, confusion) + subject filter chips + a "⚠ confused only" filter.
   - Body: term cards (term, subject tag, date, definition, my-understanding, example; confusion as a warning callout; `related` links as plain text chips).
   - The visible entry count must reflect the current filter (e.g. "3/12 terms") and be announced to screen readers (aria-live).
   - Escape `</` sequences inside embedded JS data (write `<\/`) so store content can never break out of the script block. Apply `overflow-wrap: anywhere` on cards so long tokens (URLs, identifiers) cannot force horizontal scroll.
   - No external resources (CSP). Theme: a single light (white) theme is a fine default — if you support dark mode too, verify text-over-highlight contrast in BOTH themes (a half-height highlighter mark behind dark text becomes unreadable on dark backgrounds). A sort toggle (alphabetical/recent) is optional.
   - Write UI strings in the store's dominant language.
3. Always write the HTML to `artifactHtmlPath` (same path every time).
4. **URL persistence is mandatory**: if `artifactUrl` exists in config, pass it as the `url` parameter of the Artifact tool — republishing from a different conversation WITHOUT `url` mints a brand-new URL. After the first publish, save the returned URL into config before the recap.
5. favicon: `"📖"`, fixed.
6. **Fallback**: if the Artifact tool is unavailable in this environment, save the HTML to `artifactHtmlPath` only and print its absolute path in the recap (openable in any browser).

## Entry format (store .md)

```markdown
## <term>
- subject: <slug, e.g. networking | linear-algebra | os | claude-code>
- date: <YYYY-MM-DD, updated on merge>
- definition: <one textbook-style sentence>
- my-understanding: <user's own reconstruction; append (summarized) if Claude wrote it>
- example: <one concrete example, session-sourced preferred>
- confusion: <what the user misunderstood; omit the field when none>
- related: <[[vault note links]], optional>
```

The store file starts with a `# Vocab` (or `# 단어장`) header and one guide line; entries append newest-last. Entries with a `confusion` field get a ⚠ badge and are targeted by the "confused only" filter — that filter is the highest-value review mode before an exam.

## Non-goals

- No dictionary copy-paste — the user's understanding is the first-class citizen.
- No full session transcripts or Q&A dumps — term-level entries only.
- No silent automatic writes — proposal or explicit call only.
- No Anki/spaced-repetition integration (use a dedicated tool if needed).

---

## 한국어 요약

세션에서 몰랐던 용어를 **내가 이해한 문장**으로 마크다운 단어장에 누적하고, 전체를 URL 고정 Artifact 페이지로 재발행하는 스킬. "단어장에 추가해" / "단어장 보여줘"로 호출하고, 개념 문답이 보이면 Claude가 먼저 "넣을까?"라고 제안함(조용한 자동 추가 금지). `confusion`(헷갈림) 필드가 있는 용어는 ⚠ 필터로 모아 시험 직전 복습에 사용.
