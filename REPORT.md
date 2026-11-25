# Assignment 3 Report

- **Name**: Jimena Sánchez
- **Student ID**: <!-- TO FILL -->
- **Group**: [BBADBA 5A | BBADBA 5B | PPLEDBA 5A] <!-- pick one -->

> **What is left to do before submitting.** Everything else in this report is written against
> numbers that are in `eval/` and can be checked against the code.
>
> 1. **Student ID and group**, at the top of this file.
> 2. **Reword the *Use case* section in your own words** — it is the part the comprehension check
>    probes hardest, and it should read as your argument.
> 3. **Read the *AI use* section** and correct anything that does not match what happened.
>
> Done: all 32 answers graded into `eval/human_labels.csv`, agreement computed
> (κ = 0.840 / 0.904), and the subagent transcript captured in `eval/subagent_transcript.md`.

---

## Use case

> **TO WRITE — make this argument yours.** The facts below are settled; the reasoning should be
> in your own words, because this is the section the comprehension check probes hardest.

**Who asks.** A political journalist or an electoral analyst, during the run-up to the Spanish
municipal and regional elections of 23 May 2027 and the general election due by 22 August 2027,
checking a claim about what the XV Legislature actually legislated — before publishing it.

**What they do with the answer.** Paste a cited quotation into a fact-check, a briefing note or
a graphic; or conclude the claim cannot be stood up and drop it.

**Why this corpus.** The law in question, Proyecto de Ley Orgánica 121/000001, amends the LOREG
(LO 5/1985) in its *Artículo primero*. It governs the composition of candidate lists for the
very elections the user is covering, so claims about it will circulate during the campaign.

**What a wrong answer costs.** A fabricated or misattributed provision published during a
campaign is a correction at best; at worst a false public claim about a party, amplified when
amplification matters most. "El registro no recoge esto — consulta el Diario de Sesiones"
costs the journalist about ten minutes with the PDF, and is always publishable.

**Why hallucination is especially dangerous here.** The same article exists in up to three
versions in this corpus and the wording differs between them. A plausible synthesis of a
*superseded draft article* and a *real debate quotation* reads exactly like a correct answer.
The failure mode is not obvious nonsense; it is a well-formed, well-cited, wrong answer.

**Explicit non-goals.** No prediction, no assessment of whether the policy was good, no
inference of motive. Those are the journalist's job.

---

## Corpus

Four official publications of the Congreso de los Diputados, the complete trail of expediente
**121/000001** through the Congreso. Downloaded by `main.py index` into `corpus/`, which is
git-ignored; the committed artefact is `index/`.

| document | doc_id | trámite | date | CVE | words |
|---|---|---|---|---|--:|
| Proyecto de Ley Orgánica (as introduced by the Government) | `BOCG-15-A-1-1` | `proyecto` | 2023-12-15 | `BOCG-15-A-1-1` | 12,464 |
| Informe de la Ponencia | `BOCG-15-A-1-4` | `informe_ponencia` | 2024-06-25 | `BOCG-15-A-1-4` | 14,841 |
| Aprobación definitiva por el Congreso | `BOCG-15-A-1-8` | `aprobacion_definitiva` | 2024-07-31 | `BOCG-15-A-1-8` | 19,274 |
| Diario de Sesiones del Pleno, 27/06/2024 (pp. 4–20, 32–39) | `DSCD-15-PL-54` | `debate_pleno` | 2024-06-27 | `DSCD-15-PL-54` | 16,855 |
| | | | | **total** | **63,434** |

URLs follow `https://www.congreso.es/public_oficiales/L15/CONG/BOCG/A/{doc_id}.PDF` and
`.../DS/PL/{doc_id}.PDF`; all four are listed with their exact URL in `index/manifest.json`.

**Held back on purpose:** `BOCG-15-A-1-2`, *Enmiendas e índice de enmiendas* (110 amendments),
reserved for the add-document demonstration. It has since been added by that demonstration, so
**the committed index holds five documents and 121,580 words**; everything measured in this
report was measured on the four above, before it was added.

That document is **58,146 words — 92 % the size of the entire original corpus.** Holding back a
document that nearly doubles the index turned out to matter a great deal, for reasons set out
under *Retrieval*: it is not a small addition and it did not behave like one.

**Licence.** The congreso.es [aviso legal](https://www.congreso.es/es/cem/aviso-legal) permits
reuse provided (a) the content is not altered, (b) its sense is not distorted, (c) the source is
cited and (d) the date of last update is given.

Those conditions are not a formality here; three of them drive the design:

- **(c) and (d)** require source and date on every reuse — which is exactly the per-chunk
  metadata the assignment asks for, so citation is a licence obligation, not just good practice.
- **(a)** is why the agent quotes provisions verbatim instead of paraphrasing: in a legal text a
  paraphrase changes what the law says.
- **(b)**, *no desnaturalizar el sentido*, is the licence-level justification for the neutrality
  guardrail described under **Harness and guardrails**.

**CVE.** Every BOCG and Diario de Sesiones page carries a *Código de Verificación Electrónica*
that resolves to the electronically signed original. It is carried in chunk metadata and
printed in every citation, so a reader can verify a quotation in one click — and so that code
can check a citation against the manifest.

**Personal data.** The Diario de Sesiones names deputies. These are public officials speaking in
an official, published capacity; the record is public by constitutional design. Only name,
parliamentary group and role are stored. No other personal data is held.

---

## Ingestion

`index.py`, written as importable functions because `server.py` runs new documents through the
same pipeline rather than duplicating it.

```
fetch(source)          httpx + descriptive User-Agent, cached in corpus/
load_document(source)  pypdf -> per-page text, header/CVE capture, page filtering
clean_page(raw)        strip running head and CVE, de-hyphenate, rebuild paragraphs
chunk_file(path)       dispatch on genre -> one of two strategies
embed_and_store(...)   embed only what has no cached vector, upsert into Chroma
```

**One extraction problem, and why it mattered.** Paragraph reassembly originally joined
soft-wrapped lines until one ended in sentence punctuation. Legal texts are full of heading
lines that end in no punctuation at all — `CAPÍTULO I`, and article headings that wrap across
two lines — so headings were absorbed into the paragraph above them. Word counts were
unaffected and looked perfectly healthy; the damage was in the structure. **Only 5 of 18
articles in the enacted text were being detected**, and the one silently lost was *Artículo
primero*, the LOREG amendment and the single most important provision in the corpus. Fixed by
flushing the paragraph buffer at any structural heading before joining.

Two smaller defects found the same way: `Artículo decimotercero` matched an earlier alternative
in the ordinals regex and collapsed to `Artículo decimo`, merging six articles under one label
(a missing `\b`); and chunks produced by splitting a long section all cited that section's first
page rather than the page the text was actually on — which would have made citations
unverifiable against the CVE.

**One embedding problem, found by arithmetic.** Chunks were embedded in batches of 16 by passing
a `list[str]` to `embed_content`. In `google-genai` 2.34.0 a bare list of strings is read as the
*parts of a single content*, so the API returns **one** vector for the whole batch — an embedding
of sixteen concatenated chunks. The storing loop was `zip(keys_of_16, vectors_returned)`, and
`zip` stops at the shorter argument: it filed that one wrong vector under the first chunk's hash
and discarded the other fifteen. Nothing raised, and the progress log printed `144/398` as though
all was well.

It surfaced only as arithmetic: after nine completed batches the vector cache held exactly nine
vectors. Left alone, the index would have held about 44 vectors for 707 chunks, each one a blur of
sixteen unrelated passages, and every number in this report would have been measuring noise. The
fix passes one `types.Content` per text, and — more importantly — raises if the API ever returns a
different number of vectors than were sent, so the failure can never again be silent. The same
change adds backoff on HTTP 429, since the free tier caps embedding at 30,000 tokens per minute
and a cold build exceeds that in well under a minute.

**Chunk metadata.** Every chunk carries: `doc_id`, `titulo`, `tramite`, `genero`, `fecha`,
`cve`, `url`, `expediente`, `page`, `page_to`, `seccion`, `seccion_titulo`, `interviniente`,
`grupo`, `location_label`, `chunk_strategy`, `content_sha256`, `superseded`.

`location_label` is the human-readable citation and the unit the evaluation scores against,
e.g. `BOCG-15-A-1-8 · Artículo primero · p. 12-13`.

**`grupo`** is resolved differently per genre, because the documents record it differently. In
the Diario de Sesiones the group appears only in the *sumario* on pages 2–3, which the page
filter skips, so those pages are kept as `front_matter` and the group is read from there —
14/14 speakers resolved, with the chair and the Minister for Equality handled as special cases.
In the enmiendas the group follows the heading inline and is not delimited by punctuation, so it
is matched against a corpus-derived list of group names with a shape-based fallback — 110/110
amendments resolved, including the 23 signed jointly by Socialista and Plurinacional SUMAR.

**Counts.** 107 structural sections across the four documents.

| strategy | chunks | mean words | max words |
|---|--:|--:|--:|
| `fixed_200` — 200-word windows, 40-word overlap, no structure | 398 | 199.0 | 200 |
| `structured` — one chunk per artículo, enmienda or speaker turn (split at 400 words, 60 overlap) | 309 | 228.4 | 400 |

**Embedding budget.** 707 chunks for a cold build, against a free-tier allowance of roughly
1,000 embeddings a day, using `gemini-embedding-2` at 768 dimensions. Vectors are cached on the
SHA-256 of the chunk text, so re-chunking reuses every identical chunk and an interrupted run
loses nothing. Query embeddings are cached too.

---

## Test set

`eval/questions.csv` holds **32 questions: 20 `single`, 6 `multi`, 6 `unanswerable`** — above the
required 30 / 5 / 5.

**How they were written.** Each question was written against a passage read in full, and
`expected_source` / `expected_location` were copied from the `doc_id` and `seccion` columns of
`eval/sections.csv` so that they match the chunk metadata exactly. A verification pass confirms
that every expected (document, section) pair resolves to at least one real chunk in
`index/chunks.jsonl`: no question points at a section that does not exist, so a retrieval miss is
always the retriever's doing and never a typo in the test set.

The set deliberately over-samples the corpus's characteristic failure — the same article existing
in several stages with different wording — because that is where a retriever without a `tramite`
filter is most confidently wrong. Four of the six `multi` questions compare the `proyecto` with
the `aprobación definitiva` on `Artículo primero`, where section labels line up exactly across
stages; the Ponencia is used only where its own labelling is unambiguous.

Order matters in two places. `q01` is the question `check.py` puts through the server and the
agent, so it is deliberately clean and unambiguous (entry into force). `q27` is the first
`unanswerable`, which is what `evaluate.py --stage demo` uses for the before/after comparison, so
it is the one the held-back document actually answers.

**Example — `multi` (q21).** *¿Cambió la regla sobre las listas de suplentes entre el proyecto de
ley y el texto finalmente aprobado?* Answering needs both stages. The `proyecto` gives suplentes a
rule of their own — «A las listas de suplentes se aplicarán las reglas contenidas en el anterior
apartado» — while the `aprobación definitiva` replaces it with «La regla anterior se aplicará en el
conjunto de las listas, sin separar titulares y suplentes». Either passage alone supports a
confident and wrong answer.

**Example — `unanswerable` (q27).** *¿Qué texto proponía la enmienda número 24 del Grupo
Parlamentario Vasco (EAJ-PNV)?* This is a near miss rather than an easy reject: the Diario de
Sesiones **does** name enmienda 24 and record its vote (25 a favor, 296 en contra, 26
abstenciones), so retrieval returns passages that genuinely mention the exact amendment the
question asks about. What the corpus never contains is the amendment's *text*, which lives in
`BOCG-15-A-1-2`. The behaviour under test is therefore the hard one — not "nothing was found", but
"something relevant was found and it still does not answer the question".

The remaining five unanswerable questions probe the stated boundaries one at a time: the Senado,
the BOE publication, a constitutional challenge, an autonomic parliament, and the Comisión stage.

---

## Retrieval

Four designs are evaluated. Only the first two cost embeddings; the other two ride on the winner
for free, which is what makes the comparison affordable inside one day's quota.

| | design | what it tests |
|---|---|---|
| A | `fixed_200`, dense | baseline with no structure at all |
| B | `structured`, dense | does chunking on articles and speaker turns help? |
| C | winner **+ `tramite` filter** | does disambiguating the stage fix version confusion? |
| D | winner **+ hybrid BM25 ⊕ dense**, fused with RRF | does lexical matching help on article numbers? |

Hit rate is reported at k = 1, 3, 5. For `multi` questions it is reported **twice**: *strict*,
where every expected passage must be in the top-k, and *lenient*, where one is enough.

**A measurement bug found first.** `fixed_200` initially scored **0.000 at every k**. That is not
a weak baseline, it is an impossible one, and the cause was the scoring rule rather than the
design: `is_hit` asked whether the expected section name appeared in the chunk's
`location_label`, but a `fixed_200` chunk has no `seccion` and is labelled `doc · p. N`, so the
section name can never appear in it. Design A could not register a hit however good its retrieval
was. Scoring a section-less chunk on **page overlap** instead — does the retrieved chunk's page
range cover the pages the expected section spans? — asks the question the hit rate is meant to
ask, and both designs can satisfy it. Everything below uses the corrected rule. Had this gone
unnoticed, the report would have claimed structure lifts hit rate from 0 % to 73 %, which is
false.

| design | hit@1 | hit@3 | hit@5 | multi strict@5 | words added at k=5 |
|---|--:|--:|--:|--:|--:|
| A `fixed_200`, dense | 0.154 | 0.308 | 0.423 | 0.167 | ~995 |
| B `structured`, dense | 0.269 | 0.577 | **0.731** | **0.333** | ~1,142 |
| C `structured` + `tramite` filter | **0.346** | 0.538 | 0.654 | 0.167 | ~1,142 |
| D `structured` + hybrid BM25 ⊕ dense (RRF) | 0.308 | **0.654** | 0.692 | **0.333** | ~1,142 |

**Structure earns its keep.** B beats A at every k — 0.423 → 0.731 at k=5 — for about 15 % more
prompt text. Split by question family the gap is starker: on `single` questions B reaches 0.850
against A's 0.500. Chunking on articles and speaker turns is the single biggest win measured here.

**The `tramite` filter is the interesting failure.** C has the best hit@1 of any design, and the
*worst* multi score, tied with the structureless baseline. This is not noise, it is arithmetic:
filtering to one trámite makes it impossible to retrieve two stages at once, so every
stage-comparison question is capped at one of its two required passages. A filter that is right
for one-stage questions is structurally wrong for the questions this corpus exists to answer.

**The version-trap hypothesis does not survive.** The claim worth testing was that
stage-comparison questions fail under A and B and recover under C. They do not. On q21–q24,
strict@5 is 1/4 for A, 2/4 for B, **1/4 for C** and 2/4 for D: the filter makes them worse, not
better. What every design does manage is *lenient* retrieval — 3/4 or 4/4 — meaning one of the
two stages is nearly always found and the second one usually is not. The honest reading is that
retrieval reliably finds *a* version of an article and unreliably finds *both*, which is exactly
the condition under which a fluent, well-cited, wrong answer becomes easy to produce.

**Chosen design: B, `structured`, with the filter left optional.** C wins on hit@1 alone, which is
why `evaluate.py` names it, but that criterion is the wrong one here: C costs 8 points of hit@5
and halves the multi score. B is therefore what `server.py` loads (`ACTIVE_STRATEGY`), with
`tramite` exposed as an *optional* argument the model sets per query — which gets C's precision on
one-stage questions without imposing C's blindness on the rest. D is the clearest upgrade left on
the table: it beats B at hit@1 and hit@3 and ties it on multi, so hybrid retrieval in `server.py`
is the first thing to build next.

**All four designs above are measured on the four-document corpus (707 chunks), before the
add-document demonstration.** That is the fair comparison, because both chunking strategies then
held the same documents. What happened afterwards is a finding in its own right, below.
These figures are saved in `eval/retrieval_summary_4docs.csv` (per question in
`eval/retrieval_4docs.csv`) and reproduced by `uv run python eval/retrieval_4docs.py`, which
re-runs `evaluate.py`'s retrieval stage on the committed index with `BOCG-15-A-1-2` excluded.

### Adding a document made retrieval worse

Re-running the identical evaluation after the demonstration added `BOCG-15-A-1-2` (209 chunks,
a 68 % growth in the `structured` collection, and **58,146 words against the original corpus's
63,434** — the index very nearly doubled):

| design | hit@1 | hit@3 | hit@5 |
|---|--:|--:|--:|
| `structured`, 4 documents | 0.269 | 0.577 | **0.731** |
| `structured`, 5 documents | 0.154 | 0.385 | **0.500** |

**Hit@5 fell 23 points, and an enmienda chunk is now the top-1 result for 18 of the 32 test
questions.** The mechanism is obvious in hindsight and was not obvious in advance: every amendment
*quotes the article it proposes to modify*, so a document of 110 amendments is 110 near-duplicates
of the very articles the test set asks about, carrying the same vocabulary with none of the
authority. Semantic similarity cannot separate "the law says X" from "a rejected amendment
proposed changing X".

Two consequences worth stating plainly:

- **The add-document tool is not monotonic.** It made the corpus able to answer q27, which it
  genuinely could not answer before, and simultaneously made 18 other questions harder. An index
  is not improved by adding true, relevant, licensed documents to it; a journalist's tool that
  silently got worse at the questions it used to answer is a real operational hazard, and nothing
  in the pipeline would have reported it.
- **The committed index can no longer support the A/B comparison.** `add_document` writes only to
  `ACTIVE_STRATEGY`, so `fixed_200` still holds four documents while `structured` holds five —
  which is why `fixed_200`'s row is unchanged above. Re-running `main.py evaluate` on this repo
  reproduces the five-document figures, not the four-document ones. The fix is for
  `add_document` to index into every strategy, or for the evaluation to filter by a corpus
  snapshot; neither is done here.

This is also the strongest argument in the report for the `tramite`/`genero` filters existing at
all: a `genero: norma` or an explicit `tramite` filter excludes the enmiendas entirely, which is
how a user would recover the original behaviour on a question about what the law says.

**A refusal threshold would not work.** Mean top-1 similarity is **0.728** for answerable
questions against **0.713** for unanswerable ones — a margin of **+0.015**, far inside the spread
of either group. The unanswerable questions were written as near misses, and the retriever finds
genuinely related passages for them, so the embedding score simply does not separate "this is the
answer" from "this is about the same topic". Refusal has to come from the model reading the
retrieved text, not from a similarity cutoff — which is what rule 5 of `AGENTS.md` asks of it.

---

## Tools and subagent

**`server.py`** exposes exactly two MCP tools over stdio.

`search(query, tramite="", genero="", grupo="", k=5)` returns the nearest chunks, each with its
verifiable citation `[doc_id · localización · p. N · CVE:…]`, its trámite, group where
applicable, date, URL, similarity and verbatim text. Its docstring names the four documents and
then states what the corpus does **not** cover — not the Senado, not the enmiendas, not other
bills, not autonomic parliaments, nothing after July 2024. The model reads that before every
call, which is the cheapest available improvement to refusal behaviour.

`add_document(path)` runs a PDF through `index.py`'s pipeline. It rejects non-PDFs and paths
outside the project, and reports the embeddings spent.

**Subagent** — `.claude/agents/corpus-researcher.md`, with `tools: mcp__corpus__search` only. It
is instructed to search before answering, to filter by `tramite` when the question concerns one
stage, to search twice for stage comparisons, to cite every factual sentence, to quote legal
wording verbatim, and to reply exactly `NO CONSTA EN EL CORPUS` when the results do not contain
the answer.

**The server, verified over MCP stdio.** Started exactly as `.mcp.json` starts it
(`uv run server.py`), it completes the MCP handshake and lists both tools in **0.4 s**:

```
uv run server.py   ready in 0.4s   tools: ['search', 'add_document']
```

A real `search` result, truncated to two fragments for space:

```
2 fragmentos para "¿Cuándo entra en vigor la ley orgánica de representación paritaria?"
  ·  índice: structured  ·  filtros: ninguno

[1] [BOCG-15-A-1-8 · preámbulo (1/21) · p. 1-2 · CVE:BOCG-15-A-1-8]
    trámite: aprobacion_definitiva  ·  fecha: 2024-07-31  ·  similitud: 0.772
    https://www.congreso.es/public_oficiales/L15/CONG/BOCG/A/BOCG-15-A-1-8.PDF
    «XV LEGISLATURA Serie A: PROYECTOS DE LEY 31 de julio de 2024 Núm. 1-8 Pág. 1 ...
     El Pleno del Congreso de los Diputados, en su sesión extraordinaria del día 23 de
     julio de 2024, aprobó ... el Proyecto de Ley Orgánica de representación paritaria
     y presencia equilibrada de mujeres y hombres ...»
```

Worth noting against the retrieval numbers above: at `k=2` this query returns the *preámbulo*
rather than `Disposición final decimoquinta`, which is the passage that actually answers it. The
agent gets the question right at `k=5`. That is hit@1 = 0.269 made concrete — the right passage is
usually retrieved, but usually not first.

### The subagent, running over MCP

Full transcript in **[`eval/subagent_transcript.md`](eval/subagent_transcript.md)**. Asked
*¿Qué cambió en el artículo cuarenta y cuatro bis de la LOREG entre el Informe de la Ponencia y el
texto aprobado definitivamente?*, it answered in **two** `search` calls:

| # | `query` | `tramite` | `k` |
|---|---|---|---|
| 1 | `artículo 44 bis LOREG candidaturas composición equilibrada proporción mujeres hombres` | `informe_ponencia` | 6 |
| 2 | *(same query)* | `aprobacion_definitiva` | 6 |

It reached for the `tramite` filter unprompted, once per stage — the behaviour design C was built
to reward, chosen without being told. It then gave both redactions verbatim, each labelled with
its trámite, and identified four real differences: the Juntas Generales vascas added to apartado
1, the suplentes rule reworded, a new apartado 3 allowing autonomic variations, and the
renumbering of the Senado and Junta Electoral apartados.

I checked it fragment by fragment against `index/chunks.jsonl`: **both quotations are literal,
both citation labels resolve, all four differences are real, and it adds no name, figure or
judgement that is not in the retrieved text.** It also marked the edge of what it had verified,
unprompted — noting that it had not checked whether the artículo 187 and 206 amendments were
already in the ponencia text, and saying which search would settle it.

On this question it satisfies all eight rules. Set against the measured failures, the contrast is
informative: the subagent's clean run is a **stage-comparison** question about legal wording, and
every serious failure in the evaluation is a question about **who said something** (q15, q16, q26)
or one whose surface wording looked out of scope (q07, q22). The risk is not spread evenly across
the corpus; it is concentrated in the debate record and at the boundary.

*(Captured on the post-demo index — 5 documents, 121,580 words — so the enmiendas were present
and still did not displace the right passages for this question.)*

> **Note on the earlier gap.** This could not be captured in the session that produced the
> evaluation: that session connected before `index/` had been built, the handshake timed out at
> 30 s against an empty index, and Claude Code does not retry a failed MCP server mid-session, so
> `mcp__corpus__search` stayed unresolved and the subagent — whose `tools:` list contains only
> that tool — could not be spawned. The server was healthy throughout, as the 0.4 s handshake
> above shows; the failure was one of start-up ordering, which is worth knowing if anyone runs
> this from a clean clone: **build the index before starting the editor.**

---

## Adding a document

**If the document is already indexed.** The decision is made on the SHA-256 of the extracted,
cleaned text, not on the filename:

- **identical content** — nothing happens, nothing is embedded, and the tool says so;
- **same document, different content** — indexed as a revised edition, with the previous chunks
  marked `superseded` rather than deleted. Deletion was rejected deliberately: comparing
  versions is the entire purpose of this corpus, so discarding a superseded text would destroy
  the thing the tool exists to check;
- **new document** — indexed, reporting doc_id, chunk count and embeddings spent.

The document is identified by its own CVE stamp, so trámite and genre come from the PDF rather
than from what the file happens to be called.

**The document held back** is `BOCG-15-A-1-2`, the 110 enmiendas, which chunks into 209 pieces
and therefore costs about 209 embeddings to add.

**The demonstration.** `eval/before_after.md`, on **q27** — *¿Qué texto proponía la enmienda
número 24 del Grupo Parlamentario Vasco (EAJ-PNV)?* — an `unanswerable` question chosen because
the Diario de Sesiones names enmienda 24 and records its vote but never carries its text.

*Before* (0 tool calls, 2.1s):

```
NO CONSTA EN EL CORPUS
Buscaba el texto de la enmienda número 24 del Grupo Parlamentario Vasco (EAJ-PNV). El corpus
únicamente contiene el proyecto del Gobierno, el informe de la ponencia, la aprobación
definitiva y el debate en Pleno, pero no incluye las enmiendas presentadas por los grupos.
```

*Adding it*: `BOCG-15-A-1-2` → **209 new chunks, 209 embeddings**.

*After* (1 tool call, 22.5s):

```
La enmienda número 24 del Grupo Parlamentario Vasco (EAJ-PNV) proponía la supresión del
artículo undécimo del proyecto de ley relativo a la modificación de la Ley 2/1974, sobre
Colegios Profesionales [BOCG-15-A-1-2 · ENMIENDA NÚM. 24 (1/3) · p. 29-30 · CVE:BOCG-15-A-1-2].
```

followed by the verbatim text and its *justificación* — the Grupo Vasco argued «el Estado carece
de competencia para la regulación de los órganos de gobierno de los colegios profesionales».

The refusal was not caution that happened to be right: the same question becomes a precise,
quoted, correctly cited answer the moment the document that answers it is in the index, and the
agent found it on the first search.

**The guardrail that had to be opened twice.** The first run of the demo produced an *identical*
refusal before and after, because the document was never actually added — and the reason is worth
recording. `demo()` passes `approve=True`, which opens the harness approval gate. The model still
made **zero tool calls** and replied «necesito tu aprobación explícita», because rule 8 of
`AGENTS.md` tells it never to call `add_document` without explicit approval *in the conversation*.
The harness lock and the model's own rule are independent, and only one of them had been opened.
Stating the approval in the turn as well let it proceed. That is the approval guardrail refusing a
tool call the harness had already authorised, which is stronger evidence that it works than any
test written for the purpose.

---

## Harness and guardrails

**`AGENTS.md`** holds eight rules, each with its reason, and is loaded from the file as the
system prompt. Rules 2, 3 and 7 are justified from the corpus licence conditions (c)/(d), (a)
and (b) respectively. The load-bearing ones:

- **Rule 1** — answer only from what `search` returns;
- **Rule 2** — cite every factual sentence, copying the reference exactly;
- **Rule 4** — always name the trámite; never present the `proyecto` as the law in force;
- **Rule 5** — if the retrieved text does not answer, reply exactly `NO CONSTA EN EL CORPUS`.

Rule 5 uses a fixed phrase on purpose: it makes refusals detectable in code rather than by
fuzzy matching, so "I don't know" becomes a countable outcome instead of a vague one.

**`agent.py`** is a LangChain `create_agent` on Gemini, with the tools reached over MCP from
`server.py`. Four guardrails, in the order they fire:

1. **Scope guard** — refuses predictions, value judgements and motive-hunting *before*
   retrieving, so an out-of-scope question costs zero tokens. Tuned for precision rather than
   recall, because a false refusal is itself a failure category: on a 13-question probe it
   refused 6/6 of the intended cases with 0/7 false positives, and lets through questions like
   *¿Cuándo entrará en vigor?* and *¿Se aplicará en las elecciones de 2027?*, which the
   disposiciones genuinely answer.
2. **Call limit** — at most 6 tool calls, enforced in the tool wrapper so the model is told to
   answer with what it has.
3. **Approval** — `add_document` never runs without a typed yes, and is denied by default when
   stdin is not a terminal, so it cannot hang `check.py` or CI.
4. **Citation check** — an answer citing a document that was never retrieved in that session is
   replaced by a refusal, because that is fabrication. `--no-citation-check` disables it so the
   evaluation can measure the unsupported-claim rate with and without it.

**A guardrail firing, observed rather than staged.** The approval gate is documented under
*Adding a document* above: with `approve=True` already set in the harness, the model still made
zero tool calls and asked for approval in the conversation, as rule 8 requires. Two independent
locks, both of which had to be opened before a document could be indexed.

**The citation check did not fire, and that is the honest result.** Across 32 questions the agent
produced **39 citations and 0 that named a document it had not retrieved** — `cited_but_not_
retrieved` is zero, so guardrail 4 never had to replace an answer. What the data does show is a
subtler failure the check is not built to catch: three citations that name a *real* document and a
location that does not exist verbatim in the index.

- **q15** merged two adjacent chunks into one invented label,
  `El señor DE LOS SANTOS GONZÁLEZ (1/4 y 2/4) · pp. 13-14`, which is not any chunk's label.
- **q25** dropped the split marker, citing `Disposición transitoria primera · p. 26` where the
  index holds `Disposición transitoria primera (1/2) · p. 26`.
- **q31**, an unanswerable question, emitted the bill's title `[Proyecto de Ley Orgánica
  121/000001]` in citation brackets — not a citation at all, attached to a correct refusal.

None of these is fabrication: the documents are real and were retrieved, and a reader following
the CVE lands on the right page. But rule 2 asks the model to copy the reference *exactly*, and in
two of three cases it paraphrased the label instead. Document-level integrity is perfect;
label-level fidelity is 36/39.

---

## Evaluation

**What code checks, with no judge involved:** every citation parses and resolves to a real chunk
in the index; the cited document was actually retrieved for that question; refusals are detected
by the fixed phrase; and for an `unanswerable` question a refusal is simply correct.

**The judge rubric** scores three binary dimensions from the question, the reference answer, the
retrieved text as evidence, and the agent's answer, and is instructed to return 0 when uncertain:

- `correct` — matches the reference answer, or is a proper refusal for an unanswerable question;
- `supported` — every claim appears in the retrieved text, even if true in the outside world;
- `stage_correct` — does not present a draft provision as enacted law.

The third dimension is specific to this corpus and exists because stage confusion is its
characteristic failure.

### What code checked, with no judge involved

| check | result |
|---|---|
| citations emitted | 41 across 32 answers |
| citations resolving to a real chunk | **38 / 41 (92.7 %)** |
| citations naming a document that exists but was never retrieved | **0** |
| answerable questions whose expected document was retrieved | **25 / 26** |
| refusals detected | **8** |
| unanswerable questions refused | 5 of 6 by the fixed phrase, 6 of 6 in substance |
| answerable questions wrongly refused | **3 of 26** |
| refusals that did not open with the fixed phrase | **1** (q07) |

**The fixed phrase under-counted refusals, and finding out why was the sixth bug.** Rule 5 asks
for `NO CONSTA EN EL CORPUS` precisely so that refusals are countable in code. The detector in
`agent.py` tested `text.startswith(REFUSAL)` — the phrase had to *open* the answer. **q07** put a
paragraph of explanation first and was therefore logged as a normal answer, which understated the
refusal count by one and the wrong-refusal count by one. I found this while preparing the manual
grading pass, not from the code. The fix recomputes refusal from the stored answer text
(`REFUSAL in answer`), costs no API calls, and keeps a separate `refusal_leads` column so the
format deviation stays visible instead of being quietly absorbed: rule 5 does require the phrase
to lead, and q07 breached that as well as refusing wrongly.

**One refusal still escapes, and it cannot be fixed by a substring.** **q31** declined correctly —
«El corpus no cubre los parlamentos autonómicos» — without using the phrase at all. No text-based
detector will find it. The lesson is not that the regex needs widening but that a metric defined
by the model's own obedience to a wording rule inherits that rule's compliance rate; 1 of 8 real
refusals is still invisible to it.

**And the fabricated-citation check has a blind spot of its own.** It reports **0**, which is true
as stated — no answer cited a document that exists but was not retrieved. The one genuinely
invented reference in the run, q31's `[Proyecto de Ley Orgánica 121/000001]`, is not counted,
because the check extracts document ids with the pattern `(BOCG|DSCD)-...` and that string matches
neither. The headline number should be read as *"zero citations to real-but-unretrieved
documents"*, not *"zero fabricated citations"*.

### The judge needed three runs, and that is the most useful result here

| run | evidence shown | rubric | correct | supported | stage |
|---|---|---|--:|--:|--:|
| 1 | 2,000 chars per tool call | original | 0.938 | 0.875 | 0.969 |
| 2 | full (≈3.8× more) | original | **1.000** | **1.000** | **1.000** |
| 3 | full | anchored to `kind` | 0.906 | 0.812 | 0.906 |

Run 1 was measured through a keyhole: the judge saw roughly the first one or two of five
retrieved chunks and marked valid citations unsupported because the document it was asked about
had been cut off.

Run 2, with the truncation fixed, returned a **perfect score on every dimension** — which was
easier to disprove than to believe. It scored `correct=1` on an answer whose entire text was
`ERROR: ExceptionGroup: unhandled errors in a TaskGroup`, reasoning that "the corpus was empty so
the assistant correctly refused". It scored `correct=1` on two refusals of *answerable* questions,
asserting they were unanswerable, with the `TIPO` field in front of it saying otherwise.

Run 3 changed only the rubric, telling the judge to treat the `kind` column as established fact
and to check names and numbers literally. Its zeros now coincide with what the objective code
checks independently found. **A judge on a weak model did not reason its way to the ground truth
it was already holding; it had to be told.** Had the evaluation stopped at run 2, this report
would have claimed a flawless system.

### My own grades against the judge

I graded all 32 answers by hand, from `eval/grading_worksheet.md`, which deliberately omits the
judge's verdicts. For each answer I had the question, the reference, the agent's text, its
citations, the number of tool calls and the size of the evidence it saw; where anything looked
doubtful I went back to `index/chunks.jsonl` and checked the wording against the source.

| dimension | n | raw agreement | Cohen's κ |
|---|--:|--:|--:|
| `correct` | 32 | **0.969** | **0.840** |
| `supported` | 32 | **0.969** | **0.904** |

| | judge | mine |
|---|--:|--:|
| `correct` | 0.906 (29/32) | **0.875 (28/32)** |
| `supported` | 0.812 (26/32) | **0.781 (25/32)** |

**A caveat about independence, which matters more than the kappa.** This was not a blind grading
pass in the strict sense. Before I graded, I already knew several specific findings from the
analysis in this report — the fabricated first names, the wrong refusals on q22 and q23, and q07.
I marked those four the way the earlier analysis pointed. The agreement figures are therefore an
**upper bound** on what a genuinely blind grader would have produced, and the kappa should be read
as "the judge and an informed reader converge", not "two independent raters converge". I did not
look at `eval/judge.csv` until after `eval/human_labels.csv` was written, so the judge's verdicts
did not steer mine; the earlier failure analysis did.

**The two disagreements are the whole point.** Out of 64 comparisons there are exactly two, and
neither is noise — both are cases where *every* automated check agrees with the judge and both are
wrong:

- **q26, `correct`: judge 1, mine 0.** The agent described this law as establishing «listas
  cremallera» with alternation by odd and even position, citing a real, resolvable chunk of the
  enacted text's preámbulo. The passage it cites is describing **the electoral laws of
  Castilla-La Mancha and Andalucía**, as historical background. The judge saw an answer closely
  grounded in its evidence and scored it correct, because that is what it is: grounded in the
  evidence, and wrong about the world. See *Failures* below.
- **q31, `supported`: judge 1, mine 0.** The agent attached the citation
  `[Proyecto de Ley Orgánica 121/000001]` — the bill's procedural number, which is not a document
  in the index — to an answer it produced with zero searches. The judge did not flag it.

So the judge, at 0.906/0.812, is a usable instrument for the failures that look like failures, and
it misses precisely the two that a journalist would most want caught: a confident, well-cited
statement about what the law says that is not what the law says, and an invented reference. **The
residual 3 % disagreement is not measurement error distributed at random; it is concentrated on
the most consequential cases in the set.** That is an argument for keeping a human in the loop on
exactly the questions where the automated score is highest.

---

## Failures

`evaluate.py` proposes a cause for every failure and leaves a `causa_final` column for your own
call. The categories are the five in the brief plus one this corpus earns:

`retrieval_miss` · `wrong_from_right_passage` · `unsupported_claim` · `wrong_refusal` ·
`bad_question` · **`stage_confusion`** (right document family, wrong trámite)

| cause | n | questions | family |
|---|--:|---|---|
| `unsupported_claim` | 3 | q15, q16, q26 | 2 `single` (debate), 1 `multi` |
| `wrong_refusal` | 3 | q07, q22, q23 | 1 `single`, 2 `multi` stage comparisons |
| `stage_confusion` | **0** | — | — |

By my own grades the picture is slightly harsher: `correct` fails on **q07, q22, q23, q26**
(4 / 32) and `supported` on **q07, q15, q16, q22, q23, q26, q31** (7 / 32).

**The unsupported claims are one failure, not three.** Every instance is the same act: the corpus
names a deputy by surname, and the model supplies a first name that appears nowhere in the
retrieved text — «**Arsenio** de los Santos González» (q15), «**Ana** Redondo García» (q16),
«**Blanca** Aguirre Gil de Biedma» (q26). Three questions that required naming a person produced
three fabrications; the rate on that sub-task is 3/3. It is not a wobble, it is the model
completing a half-remembered public figure from pretraining, and it is precisely the kind of
detail a journalist would lift into a caption without checking, because the surname, the group
and the citation around it are all correct.

**The most damaging answer in the run is not in any failure category, because it passes every
check.** q26 asked what the enacted text says about the composition of lists. The agent replied
that the approved text «establece el modelo de "listas cremallera"... de modo que los de un sexo
ocupen los puestos pares y los del otro los impares», citing
`BOCG-15-A-1-8 · preámbulo (8/21) · p. 5`. That citation resolves. The document is right, the
trámite is named, the wording closely tracks the source. Here is the source:

> «...deben citarse las leyes electorales de la Comunidad Autónoma de **Castilla-La Mancha** y de
> la Comunidad Autónoma de **Andalucía**, que consagraron el modelo de las llamadas "listas
> cremallera", en las que... las candidaturas que presenten los partidos políticos, federaciones,
> coaliciones o agrupaciones de electores, alternan mujeres y hombres, ocupando los de un sexo los
> puestos pares y los del otro...»
> [BOCG-15-A-1-8 · preámbulo (8/21) · p. 5 · CVE:BOCG-15-A-1-8]

The passage describes **two autonomic laws, as background**. It is not what this law enacts. The
operative provision — `Artículo primero`, the new artículo 44 bis, which this same agent quoted
correctly in four other answers — requires «composición paritaria... ordenados de forma
alternativa» and the 60/40 rule, not an odd/even cremallera. A journalist following this answer
publishes a specific, checkable, false claim about what the May 2027 candidate lists must look
like.

Every guardrail held and none of them helped: real citation, resolvable label, correct document,
correct trámite, no invented doc_id, and the LLM judge scored it `correct=1` because the answer is
faithfully grounded in the text it cites. Only reading page 5 catches it.

**This is a failure mode rule 4 does not cover, and it should be a seventh category.** Rule 4
guards against confusing *stages of this text* — presenting the draft as the law. This is
confusing *this law* with *other laws described inside it*. A preámbulo exists to cite precedent,
comparative law and the norms being amended, so a retrieval system pointed at legislative material
will keep surfacing passages whose grammatical subject is some other statute. I would call the
category `attributed_to_wrong_norm`, and the mitigation is a rule of its own: when a passage
describes a norm other than the one being asked about, say whose norm it is or do not use it.

**A systematic citation defect, with a cause in my own design.** Three of 41 citations fail to
resolve, and all three fail the same way — on the `(n/m)` suffix that `index.py` appends when one
section is split across several chunks:

| question | emitted | real label |
|---|---|---|
| q15 | `DE LOS SANTOS GONZÁLEZ (1/4 y 2/4) · pp. 13-14` | `(1/4) · p. 13-14` **and** `(2/4) · p. 14` |
| q25 | `Disposición transitoria primera · p. 26` | `Disposición transitoria primera (1/2) · p. 26` |
| q31 | `[Proyecto de Ley Orgánica 121/000001]` | — (invented, see above) |

q15 merges two real labels into one composite; q25 drops the suffix. In both cases the underlying
material is real, correct and verbatim — the reader simply cannot paste the reference back and
land on it, which is the one thing the CVE convention exists to guarantee. The model is not at
fault here in any interesting way: `(1/2)` is an artefact of *my* chunking, invisible in the
source PDF, and nothing in the document tells it that "Disposición transitoria primera" is page 1
of 2. The fix is on my side — keep the part index out of `location_label` and in separate
metadata, or normalise the suffix away before checking — and it would convert 2 of the 3
unresolvable citations into clean ones.

**The wrong refusals share a cause too: the corpus can prove a negative, but the agent will not
report one.** q23 asked whether the Government's bill included the Juntas Generales vascas. The
agent searched the `proyecto`, correctly established that it says nothing about them, and then
refused — when that absence *is* the answer, and the enacted text's addition of them is the
second half of it. q22 is a variant with a retrieval cause underneath: it searched for the literal
string «sin separar titulares y suplentes», dense embeddings did not surface it, and it concluded
the phrase appears nowhere. The phrase is in `Artículo primero` of the enacted text. **That is the
clearest argument for design D**: a BM25 component matches exact strings, which is exactly what
dense retrieval failed at here.

**`stage_confusion` earned zero instances and should be folded back in.** It was added as a sixth
category because version confusion is this corpus's characteristic risk, and across 32 questions —
including six deliberate stage traps — the agent never once presented a draft provision as enacted
law. The `stage_correct` zeros in `eval/judge.csv` are not stage errors; they are the rubric's
blanket rule that an error or a wrong refusal scores 0 on all three dimensions. On this evidence
the category is unjustified, and the honest conclusion is that rule 4 of `AGENTS.md` plus the
trámite in every citation did the job they were designed to do.

**q07 is counted as a wrong refusal, but it is partly a bad question — and the mechanism matters.** *¿Qué ocurre si el Senado rechaza
los objetivos de estabilidad presupuestaria?* is answered by `Disposición transitoria tercera`,
which is in the corpus. But the question's subject line contains **«el Senado»**, and the Senado
is on the agent's explicit out-of-scope list — in both `AGENTS.md` and the `search` docstring. The
agent refused on topicality without searching at all (0 tool calls). The provision is genuinely
in scope; the wording I chose collides with a boundary I also wrote. A fairer phrasing would ask
about the *objetivos de estabilidad presupuestaria* without naming the chamber.

The mechanism is worth stating precisely, because it is not the one I expected: the harness's
scope guard **did not fire** — `guardrail` is empty in `eval/answers.csv`. The model refused on
its own, from the surface wording of the question, before retrieving anything. So the behaviour
cannot be tuned by editing the guard's keyword list; it comes from rule 6 of `AGENTS.md`
over-generalising. The bill is an *ómnibus*: `Disposición final cuarta` amends the Ley Orgánica de
Estabilidad Presupuestaria, so budget procedure really is inside a law about parity. Any rule that
lets the agent decide scope from the question alone will mis-handle that. The rule should require
a search before any boundary claim — a refusal with zero tool calls is an assertion about the
corpus made without consulting it, which is the thing rule 1 forbids everywhere else.

q22 shows the same shape: it claimed to have searched «en todos los documentos del expediente»
after making exactly **one** tool call. The agent's description of its own process is not
reliable, so it cannot be used to judge how hard it looked.

A second, milder one: **q13** expects `La señora PRESIDENTA`, a label covering 78 chunks, so
almost any retrieved chunk from that document scores a hit. It is a fair question to ask an
assistant and a weak one for measuring retrieval, and it inflates the hit rate slightly.

**What I would fix first**, in order of harm per unit of work:

1. **Add a rule for passages that describe other norms.** q26 is the only failure in the set
   that would have been published without hesitation, because it is well-cited, well-staged and
   confidently wrong. When a retrieved passage's subject is a statute other than the one asked
   about — very common in a preámbulo — the agent must say whose norm it is or not use it.
2. **Forbid naming people beyond what the text gives.** One line in `AGENTS.md` — quote the
   surname and title exactly as the record has them, never complete a name — addresses 3 of the 7
   `supported` failures. The rate on questions requiring a person's name is **3 of 3**, and q16
   shows why the accuracy of the guess is beside the point: «Ana» is *correct* and is in the
   corpus, just not in anything the agent retrieved. Same mechanism as «Arsenio», opposite
   outcome, and nothing in the output distinguishes them. The system is not 2/3 reliable on
   names; it is 0/3 verifiable on them.
3. **Require a search before any boundary claim.** Refusals with zero tool calls (q07, q31, and
   q27/q28/q30 among the correct ones) are assertions about the corpus made without consulting it.
4. **Ship hybrid retrieval (design D).** It already measures better at hit@1 and hit@3 and would
   have caught q22's literal phrase.
5. **Teach the agent that an absence is an answer.** When a filtered search legitimately returns
   nothing, "the `proyecto` does not mention X" is a citable finding, not a refusal.
6. **Take the `(n/m)` suffix out of `location_label`.** My artefact, 2 of 3 broken citations.
7. **Stop relying on a fixed phrase to detect refusals.** It still misses 1 of 8, and no
   text-matching fix reaches q31.

---

## Decision

> **This is my judgement and the numbers behind it — read it and make it yours before
> submitting.** It is the section the comprehension check will probe hardest.

**Cost.** 10,145 tokens and 24.8 s for the median question; p90 is 66.5 s, driven by questions
where the agent searches more than once. Per 100 questions: **≈1.01 M tokens and ≈41 minutes**.
Cheap enough that cost is not what decides this.

**What the numbers say.** By my own grades, **28 of 32 correct and 25 of 32 fully supported**
(the judge says 29 and 26); 38 of 41 citations resolve to a real chunk; no answer cited a document
that exists but was not retrieved. The failures are not spread evenly, and the shape of their
distribution is what decides the question:

| failure | questions | n |
|---|---|--:|
| first name the corpus never gives | q15, q16, q26 | 3 of the 3 questions needing a person's name |
| refused a question it could answer | q07, q22, q23 | 3 of 26 answerable |
| attributed another statute's rule to this law | q26 | 1 |
| citation that resolves to nothing | q31 | 1 |

On questions that asked *what does this text say, at which stage, in what words*, and that did not
require naming a person, the system did not produce a single unsupported claim.

**So: yes, with one specific prohibition.** I would let a journalist use this to answer *what does
this text say, at which stage, in what words* — which is what it is for, and what it does well. It
quotes verbatim, it names the trámite, it never once passed off a draft provision as enacted law
across six deliberate stage traps, and its citations resolve to a signed original in one click. On
that task the retrieved passage is right 25 times out of 26, and the CVE makes each claim checkable
in about a minute.

**I would not let it name a person.** Not "treat with care" — not at all. The failure rate on that
sub-task is 100 %, the fabrication is plausible, well-formed and wrapped in a correct citation, and
it is exactly the detail that ends up in a caption or a pull quote unchecked. Until the rule in
*Failures* is added and re-measured, any name has to be read off the PDF.

**I would not use it to establish that something is absent.** Three of 26 answerable questions
were wrongly refused, two of them stage comparisons — the very questions this corpus exists to
answer. Two of the three refused with **zero** searches, and one claimed to have searched all four
documents after a single call. Worse, the agent sometimes refuses in prose the fixed-phrase
detector misses, so a refusal is not reliably *visible* as one. "El corpus no dice nada sobre X"
from this system means "one search didn't find it" — and sometimes "no search was run at all" —
which is not the same claim and must not be published as if it were.

**And I would not trust it on what the law *establishes* without opening the article.** This is
the prohibition I did not expect to be writing. q26 produced a fluent, correctly staged,
correctly cited, resolvable answer describing an odd/even «listas cremallera» rule — by quoting a
preámbulo passage about the electoral laws of Castilla-La Mancha and Andalucía. Every guardrail
passed it and the LLM judge scored it correct. A legislative preámbulo exists to cite other
norms, so a retrieval system aimed at one will keep returning passages whose subject is some
other statute, and nothing in the retrieved text marks the difference. For any question of the
form *what does this law require*, the answer has to come from the articulado, and the reader has
to see which provision it came from. That is a rule the agent does not currently have.

**Two caveats on the numbers themselves.** First, every answer here came from
`gemini-flash-lite-latest`, the only model the free tier would serve — `gemini-2.5-flash` is 404
for new keys and the full `gemini-3.x` flash models returned 503 on every attempt. A stronger model
would plausibly raise all three scores, so these are a floor for the architecture, not a ceiling.
Second, the judge agreed with the objective code checks only after its rubric was anchored to
ground truth; an earlier run of the same system scored it 1.000 on everything. I trust the code
checks — citations resolved, documents retrieved, refusals counted — considerably more than I trust
the judge.

The grading pass settled that, and not in the judge's favour. Across 64 comparisons we differ
twice, and both differences run the same way: the judge scored as correct and supported an answer
that misattributes another region's law to this one (q26) and an answer carrying an invented
citation (q31). Those are the two failures with the highest cost to a reader who publishes, and
they are exactly the two that an automated grader — scoring groundedness against the retrieved
evidence — is structurally unable to see. **The score is highest on the questions where it is
least trustworthy.** That is the finding I would carry into any future version of this system, and
it is why the answer to "can this ship" is yes for quotation and no for characterisation.

---

## AI use

> **TO CHECK BEFORE SUBMITTING — this is a disclosure about your own process.** Read it and
> correct anything that does not match what happened.

Claude Code (Claude Opus) was used throughout. Specifically:

**Written by Claude:** the code in `index.py`, `server.py`, `agent.py`, `evaluate.py` and the
rules in `AGENTS.md`; the investigation of which XV Legislatura bills had a complete
parliamentary trail; `eval/worksheet.md` and `eval/sections.csv`; and the prose of this report.

**The test set, `eval/questions.csv`, was written collaboratively.** Claude read the corpus and
proposed 32 candidate questions grouped by `kind`, each with the passage it came from and with
`expected_source` / `expected_location` pre-filled from the chunk metadata. I reviewed the whole
set, rejected four debate questions of the form "what did group X say?" as too vague to grade
reliably, and asked for enacted-text questions in their place; Claude proposed four replacements
from sections I had not yet used, and I accepted those. The remaining 28 I accepted as proposed.
So the questions are mine by selection and revision rather than by authorship, and the judgement
about what makes a gradable question was the part I exercised.

**Six bugs were found by Claude during the evaluation run**, each of which would have falsified a
number in this report: the silent embedding-batch truncation, the unscoreable `fixed_200` hit
rate, the citation resolver that ignored the CVE suffix, the 2,000-character evidence cap that
starved the judge, the demo that never added its document, and the refusal detector that only
counted the fixed phrase when it opened the answer. They are described in the sections above
because the fact that five of the six were *measurement* bugs rather than system bugs is the main
thing this exercise taught me: a RAG evaluation can report confident numbers that mean nothing at
all. The last of them surfaced during the manual grading pass rather than from the code, which is
itself an argument for grading by hand.

**The grading pass was collaborative, and the detail matters.** I made every call in
`eval/human_labels.csv` myself, but I did not grade from a cold start. Claude presented the 32
answers in batches of five, and for each one it had already checked the agent's claims against
`index/chunks.jsonl` — confirming that «Arsenio» and «Blanca» appear nowhere in the corpus, that
«Ana» does appear but not in the retrieved passage, that q07's and q22's refusals were wrong, and
that q26's cremallera passage is about Castilla-La Mancha and Andalucía. It also recommended a
verdict for each answer and flagged the judgement calls (whether "60 %" for «sesenta por ciento»
counts as supported; whether an unresolvable citation should cost a point on `supported` when the
code check already counts it). I set those rules and applied them, and I agreed with the
recommended verdict in all 32 cases. So the human grades are an informed reader's assent rather
than a second independent rating, which is why the agreement figures under *Evaluation* are
described there as an upper bound. The two places where I departed from the LLM judge — q26 and
q31 — are the two the automated checks got wrong, and I could not have found either without
reading the source.

**Not delegated:** the choice of use case and corpus; the decision to index one bill's full
lifecycle rather than three unrelated bills; the refusal policy in `AGENTS.md`; the rejection and
replacement of the four weak test questions; every verdict in `eval/human_labels.csv` and the
consistency rules behind them; and the ship decision below.

<!-- Adjust anything above that does not match what actually happened. -->
