# Assignment 3 Report

- **Name**: Jimena Sánchez
- **Student ID**: <!-- TO FILL -->
- **Group**: [BBADBA 5A | BBADBA 5B | PPLEDBA 5A] <!-- pick one -->

> **How to finish this report.** Sections marked **TO WRITE** need numbers that do not exist
> yet, or judgement that has to be yours. Everything else describes what the code does and can
> be checked against it. Rewrite any of it in your own words — the comprehension check is
> written without your code, so the parts you cannot defend out loud are the parts to redo.

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
reserved for the add-document demonstration.

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

> **TO WRITE — this is yours.** `eval/worksheet.md` and `eval/sections.csv` inventory the corpus
> so that `expected_source` and `expected_location` match the metadata exactly. Describe how you
> wrote the questions, give the counts by `kind`, and show two examples — ideally one `multi` and
> one `unanswerable`, with a sentence on why the unanswerable one is a near miss rather than an
> easy reject.

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

> **TO WRITE.** Paste the table from `eval/retrieval_summary.csv`, state the words added to the
> prompt per design, say which design you chose and why the numbers justify it. Report hit rate
> by question family too — the interesting claim to test is that version-trap questions fail
> under A and B and recover under C. If they do not, say so: a negative result honestly reported
> is worth more than a tuned one. Also report the top-1 similarity margin between answerable and
> unanswerable questions, which `evaluate.py` prints, and say whether a refusal threshold would
> have worked.

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

> **TO WRITE.** Paste one real `search` result, and one full transcript of the subagent running
> in Claude Code — ideally one where it refuses, since that is the behaviour under test.

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

> **TO WRITE.** Run `uv run python evaluate.py --stage demo`, then paste the unanswerable
> question, the refusal before, and the cited answer after, from `eval/before_after.md`.

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

> **TO WRITE.** One case where a guardrail fired, with the question and the output. The scope
> guard is the easiest to demonstrate; the citation check is the more interesting one if you
> catch a real instance.

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

> **TO WRITE.** Grade 20+ answers yourself in `eval/human_labels.csv` **before** reading
> `eval/judge.csv`, and say in the report that you did. Then report raw agreement and Cohen's
> kappa per dimension from `eval/agreement.csv`, and the headline results: correct, unsupported,
> correct refusals, wrong refusals. Where you and the judge disagree, say who you think was right
> and why — that paragraph is worth more than the kappa itself.

---

## Failures

`evaluate.py` proposes a cause for every failure and leaves a `causa_final` column for your own
call. The categories are the five in the brief plus one this corpus earns:

`retrieval_miss` · `wrong_from_right_passage` · `unsupported_claim` · `wrong_refusal` ·
`bad_question` · **`stage_confusion`** (right document family, wrong trámite)

> **TO WRITE.** Counts by cause, cross-tabulated against question family, and what you would fix
> first. Justify adding `stage_confusion` as a sixth category, or fold it back into one of the
> five if the data does not support it. `bad_question` should not be zero — if none of your
> questions turned out flawed, you were not strict enough; name the ones that were ambiguous.

---

## Decision

> **TO WRITE — this is the judgement the assignment is really testing.** Would you let a
> journalist rely on this, for what, and with what safeguard? Ground it in your own numbers
> rather than in intent: an unsupported-claim rate of X% means roughly 1 in N published claims
> would be wrong, and checking the CVE catches those in about a minute each. State the cost per
> question in tokens and seconds from `eval/summary.json`, and give a per-100-questions figure.
> Say plainly what you would *not* let it be used for.

---

## AI use

> **TO WRITE — check this is accurate before submitting; it is a disclosure about your own
> process, not mine.**

Claude Code (Claude Opus) was used to write the code in `index.py`, `server.py`, `agent.py`,
`evaluate.py` and `AGENTS.md`, to investigate which XV Legislature bills had a complete
parliamentary trail, and to generate `eval/worksheet.md` and `eval/sections.csv` as aids for
writing the test set.

The following were not delegated: the choice of use case and corpus; the decision to index one
bill's full lifecycle rather than three bills; the chunking strategy; the retrieval design
chosen from the numbers; the refusal policy in `AGENTS.md`; the judge rubric; the test set in
`eval/questions.csv`; the human grades in `eval/human_labels.csv`; and the analysis and ship
decision in this report.

<!-- Adjust the two paragraphs above to match what actually happened. -->
