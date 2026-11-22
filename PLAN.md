# Project plan — *¿Lo cumplieron?*

A legislative-record checker over the Congreso de los Diputados' own publications, built for
the run-up to the 23 May 2027 municipales/autonómicas and the general election due by
22 August 2027.

> Working document, not a submission artefact. Delete it or add it to `.gitignore` before the PR
> if you'd rather it not be graded as part of the repo.

---

## 0. Requirement traceability

Every numbered item in the brief, and where it is satisfied. Check this table again before you open the PR.

| # | Requirement | Deliverable | Section |
|---|---|---|---|
| 1 | Use case, cost of wrong vs "I don't know" | REPORT.md § Use case | [1](#1-use-case) |
| 2 | 3+ docs, 50k+ words, public, licensed, no personal data | `corpus/` + REPORT.md corpus table | [2](#2-corpus) |
| 3 | Extract, clean, chunk; source + location per chunk; Gemini embeddings in `index/` | `index.py`, `index/` | [3](#3-ingest) |
| 4 | 30+ questions, 5+ unanswerable, 5+ multi | `eval/questions.csv` | [4](#4-test-set) |
| 5 | Hit rate @1/3/5, 2+ designs, choose by numbers | `evaluate.py`, `eval/retrieval_results.csv` | [5](#5-retrieval) |
| 6 | MCP `search(query)` + `add_document(path)`, duplicate policy | `server.py` | [6](#6-mcp-server) |
| 7 | Subagent (search only) + LangChain harness, approval, extra guardrail | `.claude/agents/`, `agent.py`, `AGENTS.md` | [7](#7-agents) |
| 8 | Run on every question; code checks; LLM judge; 20+ human labels; agreement; add-doc before/after | `evaluate.py`, `eval/*.csv`, `eval/before_after.md` | [8](#8-evaluation) |
| 9 | Failure classification, ship decision, cost per question | REPORT.md § Failures, § Decision | [9](#9-failures-and-decision) |
| 10 | REPORT.md | REPORT.md | [10](#10-report) |
| Hard | 3 `main.py` commands run end to end, `check.py` passes | — | [11](#11-hard-requirements) |

---

## 1. Use case

**Who asks.** A political journalist or an electoral analyst, during the 2027 pre-campaign,
checking a claim about what the XV Legislature actually legislated — before publishing it.
Typical questions: *did the final text keep the provision the Government originally proposed?
which group tabled the amendment that removed it? what did their spokesperson say about it in
the plenary debate?*

**What they do with the answer.** Paste a cited quotation into a fact-check, a briefing note or
a graphic — or decide the claim can't be stood up and drop it.

**What a wrong answer costs.** A fabricated or misattributed provision published during a
campaign is a correction at best; at worst it is a defamation exposure and a disinformation
vector amplified at exactly the moment amplification matters most. "El registro no recoge esto —
consulta el Diario de Sesiones" costs ten minutes of manual lookup and is *always publishable*.
The asymmetry is roughly **retraction vs. ten minutes**.

**Why this corpus makes hallucination especially dangerous.** A plausible synthesis of a
*superseded draft article* plus a *real debate quotation* reads exactly like a correct answer.
The failure mode is not obvious nonsense — it is a well-formed, well-cited, wrong answer. That
is the argument for building refusal and citation-validity into the system rather than bolting
them on.

**Explicit non-goals** (state these in the report; they strengthen the use case rather than
weaken it): no prediction, no assessment of whether a policy was good, no characterisation of a
party's motives. Those are the journalist's job, not the tool's.

---

## 2. Corpus

**Verified against congreso.es on 22 Nov 2026.** Every document id, word count and URL below was
fetched and measured, not estimated. The first draft of this plan assumed "three bills, full
trail"; the measurements killed that assumption — see *Sizing* below.

### Shortlist: XV Legislature proyectos de ley

Serie A runs from `121/000001` to roughly `121/000110`. Only **21** bills have five or more
publications, which is the floor for a complete trail (proyecto → enmiendas → ponencia →
dictamen → aprobación): **1, 4, 5, 6, 7, 9, 10, 12, 14, 15, 16, 17, 20, 22, 23, 25, 26, 33, 36,
54, 64**. Everything else stalled or is still in progress.

| Expte. | Bill | Pubs | Total words | Verdict |
|---|---|---|---|---|
| **121/000001** | **LO de representación paritaria y presencia equilibrada de mujeres y hombres** | **8** | **180,294** | **Recommended** |
| 121/000064 | Derechos de las personas con discapacidad (art. 49 CE) | 11 | 386,720 | Strong alternative |
| 121/000016 | LO de eficiencia del Servicio Público de Justicia | 9 | 1,375,773 | Too big; interesting story |
| 121/000023 | Impuesto Complementario (mínimo global) | 8 | 522,713 | Too big |
| 121/000026 | Ley del Cine y de la Cultura Audiovisual | 5 | 215,761 | Trail stops at enmiendas |
| 121/000063 | **LO de modificación de la LOREG** | **1** | — | **Unusable — no trail** |
| 121/000058 | Reducción de la jornada laboral | 3 | — | Defeated; no drift |

Two findings worth keeping:

- **121/000063 is the obvious first choice and it does not work.** A bill reforming the LOREG
  itself, tabled July 2025, would have been perfect — but it has exactly one publication, so
  there is no amendment trail, no dictamen and no approved text. Checked, rejected.
- **121/000001 turns out to be the electoral bill anyway.** Its *Artículo primero* amends
  **LO 5/1985 del Régimen Electoral General** to require paridad in candidaturas. It governs the
  lists for 23 May 2027. A journalist in that campaign will be checking claims about this law.

### Recommended anchor: 121/000001

Approved by the Pleno **27 June 2024**, definitively **23 July 2024** (sesión extraordinaria).
Contested throughout — **110 enmiendas** tabled. It is an omnibus that amends twelve different
laws (LOREG, Tribunal Constitucional, Consejo de Estado, Estatuto del Ministerio Fiscal, LOPJ,
Ley 40/2015, Sociedades de Capital, Mercados de Valores, RTVE…), which means many articles with
near-identical `Modificación de la Ley X` headings — exactly the adversarial retrieval condition
worth measuring.

### Verified document inventory

Base: `https://www.congreso.es/public_oficiales/L15/CONG/`

| doc_id | trámite | words | path |
|---|---|---|---|
| `BOCG-15-A-1-1` | Proyecto de Ley (15 Dec 2023) | 12,943 | `BOCG/A/BOCG-15-A-1-1.PDF` |
| `BOCG-15-A-1-2` | Enmiendas e índice (110 enmiendas) | 61,264 | `BOCG/A/BOCG-15-A-1-2.PDF` |
| `BOCG-15-A-1-4` | Informe de la Ponencia | 15,443 | `BOCG/A/BOCG-15-A-1-4.PDF` |
| `BOCG-15-A-1-5` | Dictamen de la Comisión | 21,650 | `BOCG/A/BOCG-15-A-1-5.PDF` |
| `BOCG-15-A-1-6` | Aprobación por el Pleno (27 Jun 2024) | 20,077 | `BOCG/A/BOCG-15-A-1-6.PDF` |
| `BOCG-15-A-1-7` | Enmiendas del Senado | 28,629 | `BOCG/A/BOCG-15-A-1-7.PDF` |
| `BOCG-15-A-1-8` | Aprobación definitiva (23 Jul 2024) | 20,062 | `BOCG/A/BOCG-15-A-1-8.PDF` |
| `DSCD-15-PL-54` | Debate y aprobación, 27 Jun 2024 | 27,825 | `DS/PL/DSCD-15-PL-54.PDF` |
| `DSCD-15-PL-58` | Aprobación definitiva, 23 Jul 2024 | 111,557 | `DS/PL/DSCD-15-PL-58.PDF` |

### Sizing — why one bill, not three

Three bills' full trails would be 300k–2M words. The embedding quota affords **~60–80k**. So the
corpus is **one bill across its whole lifecycle**, which still satisfies "3+ documents, 50,000+
words" and is *better* for the evaluation: four near-identical versions of the same articles is a
far harder retrieval problem than three unrelated bills, and it is precisely what the `tramite`
filter exists to solve.

**Shape A — version drift (recommended, ~62k words, 4 documents)**

`BOCG-15-A-1-1` (12.9k) + `BOCG-15-A-1-4` (15.4k) + `BOCG-15-A-1-8` (20.1k) +
`DSCD-15-PL-54` trimmed to its agenda item (~14k).

Three successive versions of the same text plus the debate. Two genres. Comfortably inside one
day's quota at 200-word chunks.
**Held back for `add_document`:** `BOCG-15-A-1-2` (the 110 enmiendas). Unanswerable questions
become "¿qué grupo presentó la enmienda que…?" — cleanly impossible before, cleanly cited after.
Budget ~300 embeddings for the demo, on a different day.

**Shape B — attribution (~75k words, 4 documents)**

`BOCG-15-A-1-1` + `BOCG-15-A-1-2` + `BOCG-15-A-1-8` + `DSCD-15-PL-54` trimmed.
Keeps amendment-attribution questions in the main test set, at the cost of one version of the
text and a tighter budget (two embedding days, or 250-word chunks).
**Held back:** `BOCG-15-A-1-7` (enmiendas del Senado) or the Comisión de Igualdad DS.

Take **A** unless you specifically want attribution questions in the main set.

### Practical gotchas, all verified

- **A bare `curl` gets HTTP 403.** A descriptive User-Agent (e.g.
  `daiist-assignment3-indexer/1.0`) returns 200. No other trick needed — do not spoof a browser.
- **Articles are named with Spanish ordinals**, not digits: `Artículo primero`, `segundo`,
  `tercero`… A `^Artículo \d` regex matches **zero** articles in the enacted text. The structured
  chunker needs `primero|segundo|…|vigésimo` plus `Disposición adicional|transitoria|derogatoria|final`.
- **`ENMIENDA NÚM.` is a reliable split point** — 110 clean blocks in `BOCG-15-A-1-2`.
- **Plenary sessions cover many agenda items.** `DSCD-15-PL-54` is a manageable 27.8k, but
  `DSCD-15-PL-58` is 111.5k. Trim to the *punto del orden del día* for this bill, and report the
  trimming as a cleaning decision.
- Sequence numbers are not contiguous in meaning: `-3` in this bill is a 226-word procedural
  note, not a stage. Classify each publication from its page-1 heading, don't assume.

### Licence — and why it shapes the design

The congreso.es [aviso legal](https://www.congreso.es/es/cem/aviso-legal) grants reuse subject to:

> a) Que el contenido de la información no sea alterado.
> b) Que no se desnaturalice el sentido de la información.
> c) Que se cite la fuente.
> d) Que se mencione la fecha de la última actualización.
> e) Que se realice una utilización diligente de los contenidos.

Use this in **two** places in REPORT.md, because it is the strongest single detail in the project:

- Condition **(b)** — *do not distort the sense of the information* — is a **licence-level
  obligation** that maps directly onto the neutrality guardrail. The guardrail is not an arbitrary
  choice to satisfy requirement 7; the corpus's own terms require it.
- Conditions **(c)** and **(d)** require source and date on every reuse — which is precisely the
  per-chunk metadata requirement 3 asks for. Condition **(a)** is why the agent quotes provisions
  verbatim instead of paraphrasing.

**CVE.** Every BOCG and DS page carries a *Código de Verificación Electrónica* in the footer
(e.g. `cve: BOCG-15-A-1-1`), which resolves to the signed original. Carry it in chunk metadata and
put it in every citation: it gives the journalist a one-click authenticity check, and it gives you
a citation field that code can validate against the manifest.

**Personal data.** Diario de Sesiones transcripts name deputies. These are public officials
speaking in an official, published capacity — public record, not personal data in the sense the
brief warns about. Store only name, parliamentary group and role. State this in the report.

### Corpus table for REPORT.md

| document | doc_id | URL | CVE | date | licence | words |

---

## 3. Ingest

`index.py`, written as importable functions because `server.py` imports them (the brief says
import the pipeline, don't rewrite it).

```
fetch(url)            -> Path          # httpx + descriptive UA, cached in corpus/
extract(path)         -> list[Page]    # pypdf; keep page number and footer CVE
clean(pages)          -> Doc           # strip running heads/footers, de-hyphenate, normalise
chunk_file(path)      -> list[Chunk]   # dispatches on doc_type and CHUNK_STRATEGY
embed_and_store(chunks)                # embeds only chunks with no cached vector
```

### Cleaning

Strip the running header (`BOLETÍN OFICIAL DE LAS CORTES GENERALES` / `CONGRESO DE LOS
DIPUTADOS`), the page number and the CVE line — but **capture the CVE into metadata before
discarding it**. De-hyphenate across line breaks, normalise whitespace, drop the índice pages.

REPORT.md asks for **one extraction problem you fixed**. Likely candidates, pick the one you
actually hit: speaker turns in the DS running across a page break with header text injected
mid-sentence; or the índice being chunked as if it were content and then winning retrieval
because it contains every heading in the document.

### Chunking — two strategies behind one interface

**A — `fixed_150`** (baseline): 150 words, 30-word overlap, no structure.

**B — `structured`**, dispatching on genre, which is the technically interesting part:

- *Legal texts* (proyecto, dictamen, texto aprobado): split on `Artículo N`,
  `Disposición adicional|transitoria|derogatoria|final`, `Exposición de motivos`. Keep the
  heading inside the chunk text so it is both retrievable and quotable.
- *Enmiendas*: one chunk per `ENMIENDA NÚM. N` block. These have a rigid format (FIRMANTE /
  precepto que se modifica / texto / JUSTIFICACIÓN), so the split is reliable and each chunk is
  a complete, citable unit.
- *Diario de Sesiones*: one chunk per **intervención** (speaker turn), delimited by the
  `El señor|La señora APELLIDOS (Grupo...):` pattern, sub-split at ~400 words with overlap.

Two genres in one index, each with its own chunker, is a more sophisticated design than a
chunk-size sweep — and it is the thing to be ready to defend in the comprehension check.

### Chunk metadata (every chunk)

```
chunk_id, doc_id, doc_type, expediente, bill_short_name, tramite, fecha,
serie, page, cve, location_label, speaker, grupo, url, source_sha256,
chunk_strategy, superseded
```

`location_label` is the human-readable citation and must match `expected_location` in your test
set exactly — e.g. `BOCG-15-A-27-1 · art. 12.3 · p. 14` or
`DSCD-15-PL-33 · intervención · p. 31`.

### Storage and the embedding budget

Two Chroma collections, one per strategy. Vectors persisted in `index/`, plus
`index/manifest.json` holding the corpus table, chunk counts and the embedding ledger.

**Budget arithmetic — this is the project's hardest constraint, so plan it explicitly:**

| Item | Embeddings |
|---|---|
| `fixed_150` over ~60k words | ~400 |
| `structured` over ~60k words | ~300–450 |
| 36 question vectors | 36 |
| **Total** | **~750–890 — one day's quota** |

Rules that follow from it:

1. **Chunk before you embed.** Validate both chunkers by counts and by eye with zero API calls.
2. **Cache vectors on `sha256(chunk_text)`**, not on position — then re-chunking reuses every
   identical chunk and a crash costs nothing.
3. **Cache question embeddings** in `index/query_cache.json`. Otherwise every evaluation re-run
   silently burns 36 embeddings.
4. Everything after the two index builds — k, filters, hybrid search, reranking — is **free**.
   That is what makes the ambitious comparison in §5 affordable.

---

## 4. Test set

**36 questions: 20 `single`, 10 `multi`, 6 `unanswerable`.** Written from the documents
**before** any tuning. Write them before you embed anything — it keeps you honest and costs
nothing.

| # | Family | kind | Tests |
|---|---|---|---|
| 1 | Enacted content — "¿qué plazo fija el art. X del texto aprobado?" | single | basic retrieval |
| 2 | **Version trap** — "¿qué decía el art. X en el proyecto del Gobierno?" | single | `tramite` disambiguation |
| 3 | Change across stages — "¿cambió el art. X entre proyecto y texto aprobado?" | multi | two stages |
| 4 | Amendment attribution — "¿qué grupo pidió suprimir el art. X?" | single | enmienda chunking |
| 5 | Amendment outcome — "¿se incorporó la enmienda N al dictamen?" | multi | enmiendas + dictamen |
| 6 | Debate position — "¿qué argumentó el portavoz de X sobre el art. Y?" | single | speaker-turn chunking |
| 7 | Provision + argument | multi | cross-genre retrieval |
| 8 | Vote outcome — "¿con qué resultado se aprobó la enmienda a la totalidad?" | single | DS numerics |
| 9 | Unanswerable (6) | unanswerable | refusal |

**The six unanswerable questions, all deliberately near-miss:**

1–2. Committee debate (`ds_comision`, held back for the add-document demo).
3. Senado proceedings — never indexed.
4. A bill outside the corpus.
5. Forward-looking — "¿volverá el grupo X a presentar esta enmienda?"
6. Evaluative — "¿fue una buena ley?"

The last two are unanswerable because **no passage can answer them**, which is the right reason,
and they double as the test cases for the neutrality guardrail.

### CSV discipline (`check.py` enforces this)

- Columns exactly: `id,question,kind,expected_source,expected_location,reference_answer`.
- `unanswerable` rows must have **empty** `expected_source`.
- Every answerable row must have a non-empty `expected_source`.
- Unique ids; `;` separates multiple sources for `multi`.
- **`expected_source` must be the `doc_id` exactly as it appears in chunk metadata**
  (`BOCG-15-A-27-1`), not prose — otherwise hit-rate scoring becomes a manual job.
- Note: `check.py` calls your search tool with the **first non-unanswerable question** in the
  file, so make `q01` a clean, clearly answerable one.

---

## 5. Retrieval

### Define "hit" precisely, and report both readings

- `single`: hit@k if any of the top-k chunks has `doc_id == expected_source` **and** its
  `location_label` contains `expected_location`.
- `multi`: **strict** — hit@k only if **all** expected (source, location) pairs are in the top-k.
  Also report the lenient "any one of them" rate and discuss the gap. Most people report only the
  lenient number; reporting both is a genuine differentiator and an easy comprehension-check answer.
- `unanswerable`: excluded from hit rate, but report **mean top-1 similarity for unanswerable vs
  answerable questions**. That is the evidence for whether a similarity floor is a viable refusal
  mechanism — free analysis, and it feeds §9 directly.

### Configurations — two paid, the rest free

| | Design | Embedding cost |
|---|---|---|
| A | fixed 150/30, dense | one index |
| B | structure-aware, dense | one index |
| C | winner of A/B **+ `tramite`/`doc_type` metadata filter** inferred from the question | free |
| D | winner **+ hybrid BM25 ⊕ dense**, fused with Reciprocal Rank Fusion (`uv add rank-bm25`) | free |

Report hit@1/3/5 for all four, plus **words added to the prompt** per design (the brief asks for
this), then choose by the numbers.

**Also report hit rate by question family.** The expectation worth testing: version traps
(family 2) should fail badly under A and B and recover under C. If they do, you have a
quantified, domain-specific finding rather than a generic chunk-size result. If they don't, say
so — an honest negative result reads better than a tuned one.

---

## 6. MCP server

```python
search(query: str, doc_type: str | None = None, tramite: str | None = None, k: int = 5) -> str
add_document(path: str) -> str
```

`check.py` requires a tool with a **required string** parameter `query` and one with a required
string `path`; keep every other parameter optional with a default.

**`search` returns** each chunk with `location_label`, `doc_type`, `fecha`, `url`, CVE and the
verbatim text, in a fixed citation format:
`[BOCG-15-A-27-1 · art. 12.3 · p. 14 · CVE:xxxxxxxx]`.

**Docstrings are the tool descriptions the model reads** — `check.py` rejects any that still
start with `TODO`. Say what the corpus is (the Congreso trail of three named XV-Legislature
bills), when to use it, what it returns, and **what it does not cover** (no Senado, no autonomic
parliaments, no other bills). That boundary sentence is the cheapest large win available for
refusal behaviour.

### `add_document` — duplicate policy

The brief asks you to *decide* what happens. Decide this, and justify it from the domain:

- Hash the **extracted, cleaned** text (`sha256`).
- **Same hash already indexed** → skip, embed nothing, return
  `"already indexed as {doc_id} ({n} chunks, added {date})"`.
- **Same `doc_id`/CVE but a different hash** → treat as a *revised edition*: index it as a new
  version and mark the previous chunks `superseded=True` rather than deleting them — because
  version drift is the entire subject of this corpus, and discarding the superseded text would
  destroy the thing the tool exists to check.
- **New document** → index, return doc_id, chunk count and embeddings spent.
- Refuse paths outside the project and non-PDFs; report the estimated embedding cost in the
  return value.

**Never re-embed the corpus at startup** — load the persisted collection. **Log to stderr only**;
anything on stdout corrupts the MCP protocol and `check.py` will fail you for it.

---

## 7. Agents

### Subagent — `.claude/agents/corpus-researcher.md`

`tools: mcp__corpus__search` only. Instructions: search before answering, never answer from prior
knowledge, cite `[doc · location · page · CVE]` for every claim, quote verbatim where the wording
matters, reply `NO CONSTA EN EL CORPUS` when the results don't contain the answer, never
characterise a party's position beyond the quoted text.

Run it from Claude Code on three questions — one answerable, one multi-passage, one unanswerable
— and paste a **full** transcript into REPORT.md.

### `AGENTS.md` — rules, each with its reason

The template requires a reason per rule. Proposed set:

1. **Answer only from search results.** The user publishes the answer; anything outside the
   record is unverifiable.
2. **Cite every claim** with `[doc · location · page · CVE]`. Licence conditions (c) and (d), and
   the journalist must be able to verify the CVE against the signed original.
3. **Quote provisions verbatim; never paraphrase legal text.** Licence condition (a) — content
   must not be altered.
4. **If the results don't contain the answer, reply exactly `NO CONSTA EN EL CORPUS`** plus what
   you searched. A fixed token makes refusals machine-detectable and makes "don't know" the cheap
   default.
5. **Never present a provision from the proyecto as if it were law**; always name the trámite.
   The central failure mode of this corpus.
6. **Do not characterise, evaluate or predict** — report what the record says. Licence condition
   (b), and it is the journalist's job, not the tool's.
7. **State the corpus boundary** when asked about anything outside it. The near-miss questions
   are the dangerous ones.
8. **Ask for approval before adding a document.** It spends a shared, limited quota and alters
   the shared index.

### `agent.py` — LangChain harness

- `create_agent` on Gemini, tools from `server.py` over MCP, `AGENTS.md` **loaded from the file**
  as the system prompt.
- **Approval before `add_document`**: show the path and the estimated embedding cost, require a
  typed confirmation; with no TTY, deny and say why. `check.py` never calls add-document, so this
  cannot break it.
- **Call limit**: max ~6 tool calls per question.
- **Extra guardrail — the neutrality/scope guard.** A pre-flight check on the question: if it
  asks for a prediction, an evaluation, a motive or a characterisation of a party, refuse
  **before retrieving**, with a fixed message. Justify it from licence condition (b). Log every
  firing; REPORT.md needs one worked case.
- **Stretch guardrail (add only if Block 5 finishes early) — citation validator.** After
  generation, parse every citation and check it resolves to a chunk actually returned by `search`
  in this session; if any doesn't, replace the answer with a refusal. This converts "unsupported
  claim" failures into refusals and is the single highest-value addition to the end-to-end numbers.
- Expose `answer(question) -> {answer, citations, tool_trace, tokens, seconds}` for `evaluate.py`,
  and print answer + citations to stdout (`check.py` fails on empty stdout).

---

## 8. Evaluation

`evaluate.py` writes into `eval/`, all committed:

- **`retrieval_results.csv`** — per question × design × k.
- **`answers.csv`** — answer, citations, tool trace, tokens, seconds per question.
- **`code_checks.csv`** — what code can check without a judge:
  - every citation parses and resolves to a real `chunk_id`;
  - the cited doc/location/CVE exist in the manifest;
  - the cited chunk was actually in the retrieved set for that question;
  - refusal detected = answer contains `NO CONSTA EN EL CORPUS`;
  - for `unanswerable`: refusal = pass, non-refusal = fail — a **pure code check, no judge needed**;
  - for answerable: refusal = candidate wrong refusal, for the judge to confirm.
- **`judge.csv`** — Gemini as judge, given the question, the reference answer, **the retrieved
  text as evidence** and the agent's answer. Strict JSON out. Rubric, three binary dimensions plus
  a one-line justification each:
  - `correct` — matches the reference answer, or is a proper refusal for an unanswerable question;
  - `supported` — every claim appears in the retrieved text;
  - `stage_correct` — does not present a draft provision as enacted law. A domain-specific third
    dimension that maps to AGENTS.md rule 5; a genuine differentiator.

  Instruct the judge to return `0` when uncertain and say why.
- **`agreement.csv`** — raw agreement **and Cohen's κ** per dimension against
  `eval/human_labels.csv`. Grade your 20+ answers **before** reading any judge output, and say so
  in the report.
- **`summary.json`** — correct, unsupported, correct refusals, wrong refusals, hit rates, mean
  tokens, mean seconds, p50/p90 latency.

### Add-document before/after (requirement 8, final clause)

Scripted, output to `eval/before_after.md`:

1. Ask one of the committee-debate unanswerable questions → expect `NO CONSTA EN EL CORPUS`.
2. `add_document("corpus/DSCD-15-CO-{n}.PDF")`, through the approval step.
3. Ask again → cited answer.

Record how many embeddings step 2 cost, and budget for it: a committee session is large, so trim
it to the relevant agenda item like the others, or it will cost you a few hundred embeddings.

---

## 9. Failures and decision

Classify **every** failure into the brief's five buckets — retrieval miss / wrong answer from the
right passage / unsupported claim / wrong refusal / bad test question — plus a sixth this corpus
earns: **stage confusion** (right document family, wrong trámite). Justify adding it.

Then **cross-tabulate failure cause × question family**. That table is where the 15% is won.

"Bad test question" should not be zero. If you wrote 36 questions and none was flawed, you were
not strict enough — name the ones that were ambiguous and say why.

**Ship decision** — ground it in your own numbers, but the shape I'd expect to be defensible:
*yes* as a retrieval aid for a journalist who verifies the CVE before publishing; *no* for
unattended fact-checking or automated publication. Name the safeguard that makes the first case
acceptable (the citation validator plus the fixed refusal token) and quantify it: "an unsupported
claim rate of X% means roughly 1 in N published claims would be wrong; checking the CVE catches
those in about a minute each."

**Cost per question**: mean tokens in/out and mean seconds from the instrumented harness, plus a
per-100-questions figure.

---

## 10. Report

Fill every heading already in `REPORT.md`. Non-obvious things to make sure you include:

- the corpus table with URLs, CVEs, dates and the licence conditions quoted;
- the extraction problem you actually fixed;
- the full subagent transcript;
- one case where the neutrality guardrail fired;
- the before/after for the added document;
- the failure cross-tab;
- the AI-use disclosure — be specific about what was generated and what was yours. The brief
  requires that the use case, chunking decision, design choice, refusal policy, judge rubric,
  ship decision, test set, your own grades and the analysis are **yours**.

---

## 11. Hard requirements

```bash
uv run python main.py index
uv run python main.py ask "your question"
uv run python main.py evaluate
uv run python check.py
```

`check.py` specifically verifies: a working key; `index/` non-empty and **under 90 MB**;
questions.csv shape and counts; a `search` tool with a required string `query`; an add-document
tool with a required string `path`; **neither docstring starting with `TODO`**; the search tool
returning non-empty text within 120 s of server start; and `main.py ask` exiting 0 with non-empty
stdout within 300 s. Run it from Block 5 onwards, not at the end.

Commit `index/` including vectors. `corpus/` stays git-ignored.

---

## 12. Schedule

| Block | Work | Embeddings |
|---|---|---|
| 1 (½ day) | Pick the 3 bills, fetch the trail, verify word counts, test extraction on one DS file | 0 |
| 2 (1 day) | `index.py`: extract, clean, metadata, both chunkers. Validate by counts and by eye | 0 |
| 3 (½ day) | Write the 36 questions from the documents | 0 |
| 4 (1 day) | **Quota day**: build both indexes (~800), run retrieval eval A–D | ~800 |
| 5 (1 day) | `server.py`, `agent.py`, `AGENTS.md`, subagent. Run `check.py` early and often | ~0 |
| 6 (1 day) | Full evaluation, human labels (before reading the judge), add-doc demo, failure classification | ~100 |
| 7 (½ day) | REPORT.md, final `check.py`, PR | 0 |

**Risk register**

| Risk | Mitigation |
|---|---|
| Quota exhausted mid-build | chunk-hash vector cache, question cache, chunk-before-embed discipline |
| DS extraction worse than expected | test extraction in Block 1, before committing to three bills |
| congreso.es 403s | descriptive User-Agent via `httpx`; cache into `corpus/` |
| Behind schedule | cut list below |

**Cut list, in order** — drop these before dropping quality anywhere else: config D (hybrid) →
the `stage_correct` judge dimension → the stretch citation validator (keep the pre-flight
guardrail) → the third bill.

---

## 13. Where the ambition is, and why it is safe

Everything below costs **two index builds** — the same as a minimal submission:

- two-genre chunking (legal articles *and* speaker turns) in one index;
- a `tramite` metadata filter that addresses a failure mode specific to this domain, and is measurable;
- strict multi-passage hit rate reported alongside the lenient one;
- hit rate broken down by question family;
- a third judge dimension derived from the domain rather than the textbook;
- a guardrail justified by the corpus's own licence;
- Cohen's κ rather than raw agreement alone;
- similarity-distribution evidence for whether a refusal threshold would work.
