# Docket: engineering notes

What we learned building and running Docket: what worked, what didn't, how the second opinion behaves on real
traffic. Dated and factual; numbers from the running system or the published eval. The short version is in the
README's "What I learned"; this is the longer one.

## 1. The second opinion (TypeSafe Jev)

### How it works
After the checks and the risk gate, a separate decision model is asked two questions about the document:
`payment_risk` (**low** / **moderate** / **high**) and `pay_without_review` (yes / no). It sees a short structured
state: the document type, how many values were verified / uncertain / wrong / missing, which arithmetic and format
checks passed or failed, every risk gate with its result and codes, the risk score and the decision so far. **Never**
the document's text, names, amounts or bank details.

- Asked on **every** document. 2.5 s timeout + one retry; answers validated (probabilities 0–1 summing to ≈ 1, the
  claimed choice really the most likely); fail-open (no key / error / invalid answer → the gate's decision stands).
- It can only make a decision **more careful**: `high` with probability ≥ 0.6 on a document the gate would pass sends
  it to a person. It never passes a document the gate stopped.
- Agreement: on a passing document it agrees when the risk is low or it would pay; on a stopped document it agrees
  when the risk isn't low or it wouldn't pay.

### What it does on real traffic (all Docket documents read so far, mostly sample batches, 2026-11-02)

| | Count |
|---|---|
| Documents with an answer | 169, **0 failures** (fail-open never needed), median answer **187 ms** |
| Agrees with the gate | 110 (65 %) |
| Sees it differently on a document that passed (noted, the gate decides) | 57 |
| High risk on a document that passed (would have sent it to a person) | **0** |
| Risk band on documents the gate stopped | high 32 · moderate 38 |
| Risk band on documents that went straight through | low 23 · moderate 76 · high 0 |

**Reading it honestly.** It never saw high risk where the gate saw none, and every high-risk call it made was on a
document the gate had already stopped: it agrees on the serious cases. It is conservative on clean ones (a third of
passing documents get a "moderate"), which is why a moderate opinion is recorded but changes nothing. It sees the
gate's own codes, so it can't find what the gate missed; its value is a documented, independent sanity check on the
final call. The same overlay runs in Front Desk (leads) and Dossier (applicant documents).

### The review queue, measured (2026-11-02)
The queue used to sort by "worst reading first". Replayed on the 122 documents waiting (synthetic scenarios with a
known answer), that put the money-at-risk documents (duplicates, bank-detail changes, PO mismatches) at an average
position of **95 of 122**, one in the top 20: a forged bank change is read perfectly, so it sank. 44 of those 45
documents also score **under** the gate's review line (60): a hard gate stops them, not the score.

| Order tried | Fraud in top 20 | Fraud's average position | nDCG |
|---|---|---|---|
| Worst reading first (before) | 1 | 95 | 0.75 |
| Jev's P(high) alone | 12 | 57 | — |
| Risk score | 14 | 34 | 0.88 |
| **A gate failed › a wrong value › the rest, then score, then Jev** (now) | **14** | **33** | **0.88** |

Jev as the last tie-breaker is neutral to slightly better; placed before the score it made the ranking worse. So the
deterministic verdict orders the queue and the second opinion only breaks ties. The queue now shows three labelled
groups: money at risk, values to fix, other.

### One record across the three products
Every second opinion (Docket documents, Dossier cases, Front Desk leads) now carries the same core fields: `used`,
`model`, `ms`, `risk` (low / moderate / high), `p_risk`, `agreement` (agree / disagree / strong_disagree, against the
code's decision) and `effect` (none / noted / sent_to_person). In Front Desk the old agreement field said "agree" unless
Jev had changed a route, which never happened; it is now measured with the same rule as here.

## 2. Evals

- **Evals find your bugs, not the model's.** The first evaluation routed 43–64 % correctly. Almost every miss was our
  code: currency symbols glued to amounts, glyphs that split one printed line into two, the literal word "null".
  Fixed in code, routing went to 100 % on that set.
- **The eval's harness found a hang in the generator:** the "due before issue" date loop could never end on some days.
  Choices now come from candidate lists, tested on every day of a year.
- **A prompt fix the eval found:** for scans the cheap model sometimes returned all nulls, because the rules said
  "quote from the document text" and a scan has none. A rule for scanned pages (read the image, quote as printed)
  made it 3 / 3.
- **Published numbers only.** For a day the one-pager, Upwork and this README showed 99.5 % fields / 100 % routing
  from a local run nobody could check; the published eval (what the app and the site show) was 99.2 % / 98 %. Every
  public number now reads the published run, and a local run is labelled as such.

### Choosing the models again (2026-11-02)
The same 25 synthetic invoices through every candidate, plus four Dossier packs:

| Model (via OpenRouter) | Fields right | Routed right | Answered | Median |
|---|---|---|---|---|
| Gemini 3.5 Flash-Lite (first) | 99.5 % | 96 % | 25/25 | 3.5 s |
| DeepSeek V4.1 Flash (second, cheapest) | 100 % | 100 % | 25/25 | 4.2 s |
| GPT-6 Luna | 99.0 % | 92 % | 25/25 | 7.3 s |
| Gemini 3.8 Flash | 100 % | 88 % | 25/25 | 10.0 s |
| Gemini 2.5 Flash-Lite | 100 % on what it read | 90 % | **10/25** | 6.6 s |

- **The cheapest listed model is not the cheapest to run.** Gemini 2.5 Flash-Lite shares one upstream pool on
  OpenRouter and answered "rate-limited upstream" for 15 of 25 invoices, even run alone. A model that doesn't answer
  costs a person's time.
- **DeepSeek V4.1 Flash read invoices best, but stays second.** On applicant packs it writes an ID holder's name as
  one line ("ELEANOR GRACE BRENNAN") where the passport prints surname and given names apart; the grounding rule then
  marks the name red and a genuine applicant waits for a person. Flash-Lite quotes what is printed.
- **Pin the host and switch thinking off.** DeepSeek runs on ~40 hosts. With "minimal" reasoning one host took ~10 s;
  with thinking off ~2 s. The chain now names the hosts measured first (US, no training) and lets the rest follow.
- **Dropped:** free models (20 requests a minute, 429s, broken JSON), a coding-tool token plan (22–55 s per document,
  and its terms exclude application backends), and a direct provider API that costs 4–8× more per input token.

## 3. Checks and fraud

- **The first sender must not define the truth.** An early version trusted the first IBAN it saw for a supplier, so
  whoever got there first was approved forever. Trust in a bank account now comes only from a person or the
  imported vendor list.
- **US invoices had no bank check at all.** The gate knew only IBANs; US vendors pay by ACH / wire, so the commonest
  US invoice fraud ("we changed banks") went unseen. Routing + account numbers now go through the same
  approved / seen / changed gate, with the ABA checksum in code.
- **Scans are read twice.** The model reads the image; OCR on our server reads it without seeing the model's answer.
  A value is green only when both agree digit for digit, so a clean scan can go straight through like a PDF.
- **Exact amounts, safely** (2026-11-02, found on a real statement in Dossier, same engine): a model quoting
  "10450.00" for a printed "10,450.00" was a fuzzy near-miss. It is now an exact match, but only inside one printed
  word in a well-formed amount; a security review showed "12.34" and "5.00" side by side would otherwise pass as
  "12,345.00".

## 4. Email intake

- Forwarding keeps the evidence: a supplier's mail forwarded by a mailbox rule still verifies its DKIM signature, so
  the sender check works on forwarded invoices.
- Look-alike sender domains (northwlnd.co.uk for northwind.co.uk) always stop; a new or personal sender domain adds
  risk (configurable).
- A "[reference]" in a subject must not let a stranger add a file to someone else's case (references are guessable):
  only the business's own signed mail, or the same signed sender, may do that.

## 5. App and operations

- **Samples bury real work.** A demo batch adds 14 documents per click; the review queue held 6 real documents under
  121 samples. The queue now opens on real documents (Real / Samples / All).
- **n8n test events:** a webhook test ping reached a workflow with no branch for it and returned 500; every recipe now
  has a no-op branch for test events.
- **Repository hygiene:** the first web commit included a package-manager store (~36k files); removed and ignored,
  it stays in history until a deliberate rewrite.

## 6. Principles that held

- The model reads, code decides; a value turns green only when it is found on the page and passes every rule.
- Fail closed on money, fail open on the optional (an unavailable second opinion changes nothing; an unreadable value
  goes to a person).
- Measure before claiming; publish only what the app itself can show.
- Every security review finding becomes a test.
