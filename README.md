# auspex

> Your chat archives know things you don't.

Drop your chat exports in. Get back a map of your social network — who's drifting from whom, whose tone toward whom shifted, what people in your life are really like vs. what they say about themselves. Click any person, see who they actually are. Click any line between two people, see how that relationship is moving. Re-run after new chat arrives and a feed surfaces what changed.

Everything is local. The text never leaves your machine. No cloud, no telemetry, no remote API.

![auspex UI on the built-in demo corpus — radar feed on the left, force-directed network in the center, evidence-cited sidebar on the right](assets/example.png)

*Screenshot above is the demo corpus (3 synthetic people, no actual LLM passes run). Real runs populate themes, predictions, cited quotes, pair-edge warmth, and the insight feed.*

### what the radar feed actually surfaces

After each run, the radar diffs against the previous snapshot and ranks every change by computed urgency. A real feed at 30 people in your network reads something like:

```
RADAR · 12 of 31 insights this run · sorted by urgency
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
●  alice → bob       tone shifted: curious → dismissive (warmth −0.71)
●  charlie           new high-stakes claim: "i'm burned out"  [unverifiable]
●  alice ↔ bob       relationship_cooling · interaction −68% vs baseline
◐  bob ↔ charlie     alliance_forming · shared topics +18% · warmth +0.41
◐  alice             theme 'avoids confrontation' [FADING → ACTIVE]
◐  bob               cognitive_shift · integrative complexity ↑↑ +47%
○  diana → alice     asymmetric_investment · 18 acts vs 3 return
○  charlie           confidence_jump · 'analytical mindset' 0.71 → 0.91
○  bob               new_self_claim "i don't get jealous"   [inconsistent]
·  eve ↔ frank       new_pair · 7 recent exchanges
·  alice             new_theme 'romanticizes effort' (12 msgs · conf 0.62)
·  charlie           cognitive_shift · self-monitoring ↓ −15%
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
●  critical  ≥ 0.75      ◐  high  ≥ 0.55      ○  medium  ≥ 0.35      ·  low
```

Each line clicks through to the supporting evidence — the actual messages that triggered the diff. **None of these are LLM opinions.** Confidence is computed from counted falsification probes. Warmth is computed from directional tone histograms. Asymmetric investment is computed from reply/initiation counts. The LLM's job is to extract structured signal; the math decides what's important.

> Most LLM personality tools are horoscopes with a tech stack. This one isn't.

### how it compares to the alternative

```
                       typical LLM personality tool    │    auspex
─────────────────────────────────────────────────────────┼───────────────────────────────
   why does it think X?            shrug                │    every claim → msg_id citation
   is the confidence real?         model types a number │    (s+1)/(s+3·f+2) from probes
   handles self-claims how?        folds them in        │    separate reconciliation stream
   detects what changed since…?    re-run, read again   │    ranked diff feed, urgency-scored
   issues an IQ or MBTI?           yes, confidently     │    refuses, hardcoded
   models the network?             nope, individuals    │    real pairwise dynamics
   constant validity check?        none                 │    Phase 4 adversarial probes
   sends data anywhere?            cloud API            │    100% local, no telemetry
   incremental on new data?        no, redo from zero   │    fingerprint short-circuits
   ─────────────────────────────────────────────────────┼───────────────────────────────
   what you get                    smooth prose         │    a falsifiable measurement
```

## What it tells you

- Who in your network is drifting away from you (or getting closer).
- How your closest friends actually talk to each other when you're not the topic.
- What people in your life are really like, vs. what they say about themselves.
- What's changing week-to-week — relationships cooling, tones shifting, alliances forming.
- Where each person sits relative to your whole network on cognitive style, openness, conscientiousness, and the rest.

## Quickstart

```bash
ollama pull gpt-oss:20b               # or qwen2.5:7b, llama3.1:8b, aya-expanse:8b — your choice
cp -r lexicons.example lexicons        # seed the gitignored translation files
cp config.example.json config.json     # then edit: self_name + handles + aliases
# drop chat exports into data/   format: YYYY-MM-DD HH:MM | sender | message

cargo build --release
OLLAMA_MODEL=gpt-oss:20b ./target/release/auspex data/*.txt
# server starts at http://localhost:8765/ automatically when the pipeline finishes
```

### Just opening the UI (skip the pipeline)

If the pipeline has already produced `graph.html` and you just want to reopen the UI to navigate existing results — no LLM, no scans, no re-eval — pass `--serve`:

```bash
./target/release/auspex --serve     # or  -s
```

This starts the embedded HTTP server on `http://localhost:8765/` against the existing `graph.html`. Useful for: reopening the UI after closing the browser, sharing a single port for multiple browser tabs/sessions, accessing the chat panel without re-running the pipeline. Will exit immediately with an error if `graph.html` doesn't exist yet.

### Re-runs (adding new exports)

Drop additional chat exports into `data/` and re-run the full command. auspex's incremental pipeline does only the work that's actually new:

```bash
./target/release/auspex data/*.txt    # pipeline + serve
```

Phase 0 classifies only the new messages, Phase 1 scans only new substantives, downstream phases reuse cache when upstream didn't change. For people whose corpus fingerprint matches the previous run, the entire pipeline short-circuits — their profile is loaded from cache, no LLM work happens. Then the insight engine diffs against the prior snapshot and surfaces only what's genuinely new.

## How to

Recipes for the things you'll actually need to do. Theory lives in the primer below; this section is task-oriented.

### Export chat history from your platforms

The pipeline expects one message per line in `data/*.txt`, format:

```
2026-04-12 18:32 | alice | hey did you see the climbing video i sent
2026-04-12 18:33 | you   | yeah that v7 is unreal
```

Whitespace around the pipes is fine. Timestamps are optional — if missing, the chronological-recency features degrade gracefully but everything else works. Each file can be one person, one channel, or a whole archive; the pipeline groups by sender name regardless.

Per-platform recipes:

**Discord** — use [`DiscordChatExporter`](https://github.com/Tyrrrz/DiscordChatExporter) (CLI or GUI). Export each channel as plain text or CSV, then reshape:

```bash
# from DiscordChatExporter's CSV export (--format Csv):
awk -F',' 'NR>1 { gsub(/"/,"",$0); print $1" | "$2" | "$3 }' channel.csv > data/channel.txt
```

**iMessage** — install [`imessage-exporter`](https://github.com/ReagentX/imessage_exporter), export to TXT, then:

```bash
imessage-exporter -f txt -o /tmp/imsg
# imessage-exporter uses "Apr 12, 2026  6:32:14 PM\nFrom Alice\nbody"
# reshape with a small awk or python script; example python:
python3 -c "
import re, sys, glob
for path in glob.glob('/tmp/imsg/*.txt'):
  with open(path) as f: text = f.read()
  # parse blocks ... write to data/<name>.txt in 'YYYY-MM-DD HH:MM | sender | text' format
"
```

**WhatsApp** — phone → chat → ⋮ → "Export chat" → "Without media". The export format is `[12/04/26, 18:32:14] Alice: hey`. Reshape:

```bash
# turn [DD/MM/YY, HH:MM:SS] sender: msg into 'YYYY-MM-DD HH:MM | sender | msg'
sed -E 's|^\[([0-9]{2})/([0-9]{2})/([0-9]{2}), ([0-9]{2}):([0-9]{2}):[0-9]{2}\] ([^:]+): (.*)$|20\3-\2-\1 \4:\5 | \6 | \7|' chat.txt > data/chat.txt
```

**Telegram** — Desktop → ⋮ → "Export chat history" → JSON. Then:

```bash
jq -r '.messages[] | select(.text != "") | "\(.date[:16] | sub("T";" ")) | \(.from) | \(.text | if type=="string" then . else (map(if type=="string" then . else .text end) | join("")) end)"' result.json > data/chat.txt
```

**Signal** — desktop client doesn't export directly; use [`signal-backup-tools`](https://github.com/bepaald/signalbackup-tools) on an Android backup, dump to CSV, then reshape with the same awk pattern as Discord above.

After reshaping, spot-check with `head data/<file>.txt` to confirm the format matches the example at the top of this section.

### Add a new language to the lexicons

```bash
# copy any existing language file as a template
cp lexicons/self_ref/en.txt lexicons/self_ref/de.txt

# edit it — one entry per line, # for comments, blank lines OK
$EDITOR lexicons/self_ref/de.txt

# repeat for the other categories: other_ref, certainty, hedging,
# positive, negative, analytical, and intents/<intent_name>/

# next run will log: "lexicon: self_ref loaded (N entries from langs: de, en)"
```

The filename stem (`de`, `fr`, `es`, whatever) is only a diagnostic label — the loader merges every `.txt` file under each category. Nothing is hardcoded about which languages are supported.

### Switch to a different LLM model

```bash
ollama pull aya-expanse:8b               # or qwen2.5:7b, llama3.1:8b, gpt-oss:20b, granite3-dense:8b
OLLAMA_MODEL=aya-expanse:8b ./target/release/auspex data/*.txt
```

Switching models mid-corpus is safe — the cache is keyed by message ID, not by model. Already-classified messages stay; new messages get the new model. Quick guidance:

- **Multilingual corpus**: `aya-expanse:8b` or `qwen2.5:7b`. Built for cross-language coverage.
- **English-heavy + want depth**: `gpt-oss:20b` for synthesis prose quality.
- **Structured-output reliability**: `granite3-dense:8b` — IBM tuned it specifically for function-calling and JSON.
- **Speed-first, accept some quality loss**: `llama3.1:8b`.
- **Avoid for this tool**: `llama3.2:3b` and below (multilingual coverage too weak), `phi3:mini` (JSON reliability degrades fast at small sizes).

### Use the in-browser chat panel

Open `http://localhost:8765/`. Press **Ctrl+K** to toggle the chat panel.

| Shortcut | Action |
|---|---|
| `Ctrl+K` / `Cmd+K` | toggle chat panel |
| `Ctrl+/` / `Cmd+/` | focus the "jump to person" search |
| `Esc` | close pair panel → sidebar → chat (peels one layer at a time) |
| `F` | fit graph to view (when not focused in an input) |
| scroll wheel | zoom graph |
| drag empty space | pan graph |

Example queries that lean into the data:

- *"what's changing with alice lately?"*
- *"who in my network is most likely to drift away next, and why?"*
- *"which of bob's self-claims look inconsistent with how he actually behaves?"*
- *"compare alice and bob as collaborators — who's more reliable, who's more creative?"*
- *"what active themes does my network share?"*
- *"show me anyone whose tone toward me has shifted in the last quartile"*

Don't ask things that require information outside the corpus (current events, what someone is doing right now, anything not in the chat). The system prompt explicitly tells the model not to invent historical figures or generic answers — but a question with no grounding will still produce a thinner reply.

### Reset and start over

Selective wipes — pick what you want to redo:

| Command | What gets redone |
|---|---|
| `rm -rf profiles/` | Phase 0–6 from scratch for everyone (most expensive) |
| `rm -rf insights/` | Snapshots clear; next run treats everything as "new" for the insight feed |
| `rm -rf index/` | Re-embed every message (expensive — minutes-to-an-hour depending on corpus size) |
| `rm graph.html` | Just regenerate the UI from existing cached profiles (cheap, no LLM) |

Nuclear option — wipe everything generated, keep your source data + lexicons + config:

```bash
rm -rf profiles/ insights/ index/ graph.html
```

Do **not** wipe `data/` (your chat exports), `lexicons/` (your word lists), or `config.json` (your identity). Those are inputs; the tool can't reproduce them.

### Troubleshoot when output looks broken

| Symptom | Likely cause | Fix |
|---|---|---|
| Everyone's classifications come back `low-info` | Model too small or wrong language family | Switch to `aya-expanse:8b` (multilingual) or `gpt-oss:20b` (depth) |
| `is_meta_cognitive` and `is_multi_perspective` are always `false` across all profiles | Model is satisficing on those booleans (known failure on smaller models even with prompt nudges) | Switch to `gpt-oss:20b` or `granite3-dense:8b`; check a sample classification file under `profiles/<name>_classifications.json` to confirm |
| Phase 0 takes much longer than expected | `OLLAMA_NUM_CTX` set too high for your VRAM, or model is swapping to CPU | Lower context with `OLLAMA_NUM_CTX=8192`, or pick a smaller model |
| Chat panel shows `Error: Failed to fetch` | Embedded server isn't running, OR you opened `graph.html` via `file://` instead of through the server | Run `./target/release/auspex --serve` and open `http://localhost:8765/` |
| Themes read like horoscope on someone with thin data | Under ~500 substantive messages isn't enough signal for stable themes | Trust the cited quotes more than the synthesized paragraphs; consider excluding people with too little data from the pipeline (`messages.len() < 10` is the floor; raise it if you want stricter) |
| A clearly distressing self-claim doesn't fire as `high_stakes_claim` | Claim was tagged `ironic` or `hyperbole` by Phase 0 | Check the self-claim panel for the claim; verdict will show `not-literal`. Edit Phase 0 prompt if you disagree with the classification |
| All edges look the same color/thickness | Pair interactions need at least 3 adjacent messages between two people to compute meaningfully | Pairs with sparse data get dropped automatically; only people with substantial mutual chat history form visible edges |

### Update existing chat data without wiping caches

Drop new exports into `data/`, re-run:

```bash
./target/release/auspex data/*.txt
```

Phase 0 classifies only the new messages (incremental). Phase 1 scans only new substantives (covered set tracked via `support_ids`). Downstream phases reuse cache when upstream didn't change. For people whose corpus fingerprint matches the previous run, the entire pipeline short-circuits and the saved Profile is loaded from `profiles/<name>_profile.json`. The insight engine then diffs against the prior snapshot and surfaces only what's actually new. This is the intended steady-state usage — add data, re-run, read the radar.

## How it works

```
            chat exports (data/*.txt)
                       │
                       ▼
                  parse_files
                       │
         ┌─────────────┴─────────────┐
         ▼                           ▼
    fastembed                    by_sender
    index                            │
                                     ▼
                          phase 0  classify (19 fields per message, w/ corpus context)
                                     │
                                     ▼
                          phase 1  observation extraction (cited quotes, verified)
                                     │
                                     ▼
                          phase 2  cluster → themes
                                     │
                                     ▼
                          phase 3  deepen + emit falsification specs
                                     │
                                     ▼
                          phase 4  probe each spec → ValidatedTheme + computed confidence
                                     │
                                     ▼
                          phase 5a cognitive markers (lexical + Phase 0 booleans)
                          phase 5b Big Five citation panels (no scores)
                          phase 5c self-claim reconciliation
                          phase 5  synthesis (no IQ, no MBTI)
                                     │
                                     ▼
                          phase 6  predictions (intent clusters × validated themes)
                                     │
                       Profile (provenance through every step)
                                     │
            ┌──────────┬─────────────┴─────────────┐
            ▼          ▼                           ▼
       cross-person  pair                       insight diff
       z-score       interactions               vs snapshot
            │          │                           │
            └────┬─────┴────┬──────────────────────┘
                 ▼          ▼
              graph.html   insights/feed.json
                 │
                 ▼
            embedded HTTP server :8765/
                 │
                 ▼
                you
```

Re-running with zero new data: pipeline does zero LLM work, just loads cached profiles and re-renders. With new data: Phase 0 classifies only the new messages, Phase 1 scans only new substantives, downstream phases invalidate exactly when they need to, the insight engine diffs against the prior snapshot and emits only what's actually new.

## This tool exists because most of the alternatives are bad

The current state of "AI personality analysis" is one of two failure modes:

1. **One-shot LLM horoscopes.** Dump messages into a prompt, get smooth paragraphs back. No provenance. No way to ask *why*. No way to know if it changed since last week. Confidence numbers typed by the same model that wrote the analysis. The tech-stack-wrapper version of a Buzzfeed quiz.
2. **Black-box "predictive" hiring/dating/leadership tools** that quietly assign IQ-from-chat or MBTI-from-tweets and won't tell you what evidence got used. Trade secret. Trust us.

This rejects both. Every claim cites verbatim quotes by stable `msg_id`. Every confidence number is computed from counted evidence, not LLM mood. The pipeline runs adversarial probes against its own conclusions before reporting them. It refuses to issue an IQ score because chat-lexicon-to-IQ has no defensible grounding — and that refusal is hardcoded in the synthesis prompt, not a side note.

This is a profiling tool that's honest about being one.

## What you actually get

Run the binary against your chat exports. After a few hours of LLM time you have:

- **Per-person psychological profiles** — identity, anxiety, social style, growth, vulnerability paragraphs. Every claim points back at a verbatim quote in the message log.
- **Validated themes**, where "validated" means the LLM was asked to specify what behavior would *contradict* the theme, the index was searched for that behavior, candidates were probed one-at-a-time, and confidence was computed by formula from probe outcomes. Falsification, not vibe.
- **Cognitive style markers** — abstract-noun rate, conditional rate, lexical complexity, integrative complexity, domain breadth, self-monitoring. Z-scored *across your corpus.* "Higher than the people around you" is the kind of statement the tool makes. "IQ 132" is the kind it refuses.
- **Big Five citation panels** — top-5 quoted messages per dimension. No score. Just the evidence the model would have used. You decide what it means.
- **Self-claim reconciliation** — when someone writes "i'm an INTJ" or "I have IQ 145," that doesn't fold into their trait profile. It's pulled into a separate stream and reconciled against actual behavior with a verdict of `consistent / inconsistent / unverifiable / not-literal`.
- **A real social network graph** — edges from actual interaction (reply latency, who-addresses-whom, tone-toward-specific-person, topic overlap, mentions), not behavioral-style similarity. Thickness = intensity. Color = warmth (green/red/neutral).
- **A ranked insight feed** that diffs the current state against the previous run. Insight types: `new_theme`, `theme_status_change`, `confidence_jump`, `cognitive_shift`, `new_self_claim`, `claim_verdict_change`, `high_stakes_claim`, `relationship_cooling`, `relationship_warming`, `tone_shift_toward`, `asymmetric_investment`, `alliance_forming`, `new_pair`. Each carries an urgency score computed from delta magnitude × baseline confidence × novelty.

Open `http://localhost:8765/`. Click a node, see the sidebar with everything. Click an edge, see a baseline-vs-recent pair comparison with directional tone histograms. Ctrl+/ to jump to a person. Ctrl+K to ask the chat panel a question — the prompt explicitly tells the model: *don't refuse, don't punt to historical figures, answer from this data, name the people in this corpus by name.*

## What a real insight looks like

A pair insight, surfaced automatically on a re-run after new chat arrived:

```
{
  "insight_type": "tone_shift_toward",
  "person": "alice",
  "summary": "alice → bob: tone shifted curious → dismissive (warmth -0.34 → -0.71)",
  "details": "Direction-of-addressee tone changed in the recent quartile. Negative-direction shifts often precede cooling.",
  "urgency": 0.70,
  "first_seen": "1747...",
  "related_claim": "bob"
}
```

A theme on someone's profile:

```
"Quietly resentful when plans are made without them"
  msgs: 23
  confidence: 0.71 (computed from 23 supporting + 2/9 falsification probes confirmed)
  recent_share: 0.52  [ACTIVE]
  support quotes (each clickable to source):
    "ok i guess if everyone else already decided"
    "no its fine whatever you want is fine"
    "wasnt asked but sure ill come"
```

## The seven choices that make it different

1. **Provenance through every phase.** Every observation carries `support_ids`. Every theme's evidence is verifiable. Quotes shown in the UI are *not* near-similarity matches — they're the actual messages the LLM cited, verified by substring match before being saved.
2. **Real falsification, computed confidence.** Phase 3 emits explicit falsification specs. Phase 4 probes for them. Confidence is `(support + 1) / (support + 3·falsified + 2)` — diagnostic-weighted, derived from probe counts. The LLM never types the number.
3. **Self-claims live in a separate stream.** Behavior beats self-report. Saying you're introverted doesn't make the profile read introverted. The claim is logged, reconciled, and surfaced as its own panel — you see both signals, the tool doesn't conflate them.
4. **Temporal split.** Every marker is computed twice (full corpus + recent quartile). Every theme carries `recent_share` + a status of `active` / `stable` / `fading`. The synthesis prompt is told to write active themes in present tense and fading themes in past. *What is changing* is a first-class output, not an afterthought.
5. **Cross-person z-scoring.** Within your corpus, not against some imaginary population. "More analytical than 80% of your network" is honest. "Analytical, IQ ~130" is not.
6. **Pairwise dynamics.** The graph is a real network model. Per-pair: directional reply counts, median latency, initiation balance, tone distribution when addressed-specifically, topic overlap, mentions, baseline vs recent. The "Network" in "Network Intelligence" is not aspirational.
7. **Incremental + same-origin.** Snapshot-based diff. Embedded HTTP server proxies `/api/*` to local Ollama so the chat panel is same-origin. No CORS dance. No separate process to launch. Re-runs after adding new exports do *only* the work that's actually new — including a fast-path that skips the whole pipeline for people whose corpus fingerprint matches the previous run.

## A primer for newcomers (read this before relying on output)

This section explains *what kind of measurement this is*, *why the pipeline is structured the way it is*, and *how to read the output without over-interpreting it.* It is long on purpose. If you skip it, you'll probably misuse the tool.

### 1. What this actually measures (and what it doesn't)

auspex measures **textual signatures of communicative behavior over time.** That is not the same thing as measuring "personality."

When you read a profile that says someone is "highly analytical, perfectionistic, avoidant of direct conflict," what's literally being claimed is: *over the chat messages this person sent, these patterns appear with high regularity, in ways an LLM probed against contradicting evidence and didn't find much.* That's it.

It's tempting to read those paragraphs as claims about who the person *is*. They are not. They are claims about how the person *writes in this medium*. The two overlap, but the gap is the point.

Published research on inferring Big Five personality from text gets correlations of roughly **0.3 to 0.5** against gold-standard self-report inventories. That means: even with the best models, even with academic-grade methodology, even on English corpora with thousands of words per subject, what you measure from text only agrees with what someone would report about themselves about half the time. Your friend group's chat is not better than that. It's probably worse — chat register is casual, multilingual, full of jokes and quoted memes, and missing entirely whole modes of communication (tone of voice, body language, what someone does when they're alone, what they choose not to write).

So when you open auspex's sidebar and read a paragraph about someone's "anxiety patterns," the right mental model is: *here is one disciplined view of their messaging behavior in this archive, framed in clinical-adjacent language, with citations.* That's useful. It's not ground truth. The construct-validity ceiling — the maximum agreement with reality — sits somewhere around 0.4 even in perfect conditions, and you don't have perfect conditions.

This is why most "AI personality" tools are dangerous: they hide this ceiling, present outputs in confident prose, give a single score, and let users round it to certainty. auspex shows you the citations, refuses to issue point scores for IQ or MBTI, and forces every claim to be inspectable.

### 2. Why provenance is the foundation, not a nice-to-have

The most important architectural commitment is: **every claim auspex makes can be traced back to specific messages.**

This sounds boring until you realize what its absence means. A normal LLM "analysis" pipeline does this: stuff messages into a prompt, get paragraphs back, display paragraphs. If you ask *why does the model think this person is perfectionistic?*, you cannot answer. The model doesn't remember which messages drove which sentence. The output is unfalsifiable — you cannot point at a specific piece of evidence and say "that's misread" because the chain from evidence to conclusion is hidden inside the model's forward pass.

auspex treats unfalsifiable claims as failures. Every observation extracted in Phase 1 carries `support_ids`: a list of the actual message IDs the LLM cited. The IDs are verified by substring-matching the quoted phrase against the cited messages before saving — so hallucinated quotes get dropped, not displayed. Phase 3 carries the `support_ids` forward into themes. Phase 4 validates against those same messages. The synthesis paragraphs in the sidebar reference theme names, and theme names reference quotes, and quotes reference msg_ids, and msg_ids reference timestamps and senders.

This is why you can click a quote in the UI and see the source message in context. It's also why caches invalidate correctly: when new messages come in, the system can compute exactly which observations might be affected (the ones whose support set overlaps with what changed) instead of redoing everything blindly.

The practical consequence: if a claim in a profile feels wrong to you, you can find the messages it's drawn from. You can decide for yourself whether the read is fair. That's the only way an analytical tool deserves your trust.

### 3. Falsification, not validation

Most "AI" systems validate themselves: they generate a claim, then evaluate the claim using the same model that generated it, get a high confidence, and present the result. This is circular. It tells you the model's outputs are internally consistent. It does not tell you the outputs are correct.

auspex tries to do the opposite. After Phase 1 extracts observations and Phase 2 clusters them into themes, Phase 3 asks the LLM a critical question: *what specific behavior would CONTRADICT this theme?* The model is forced to articulate, in words, what evidence would weaken or refute its own claim.

Phase 4 then takes those falsification specifications, searches the embedding index for candidate disconfirming messages, and probes each candidate one-at-a-time with a strict yes/no judgment: *does this single message exhibit the falsifying behavior?* The counts of confirmed falsifications get fed into a formula:

```
confidence = (n_support + 1) / (n_support + 3·n_falsified + 2)
```

This is a smoothed proportion — supporting evidence in the numerator, falsifying evidence weighted 3× in the denominator (because finding a clear counter-example is more diagnostic than finding one more example). The "+1 / +2" terms are a Laplace prior: with zero evidence the formula returns 0.5, which is honest uncertainty rather than a fake "no opinion."

The point is: **the model never types the confidence number.** A 0.71 means "23 supporting messages, 2/9 falsification probes confirmed" — actual counted evidence. A 0.40 means there's roughly as much disconfirming evidence as supporting evidence; treat the theme as a working hypothesis, not a verdict.

This methodology is borrowed from the philosophy of science (Karl Popper's argument that a theory you can't disprove isn't really a theory) and from Bayesian statistics (the prior + likelihood update). It is more rigorous than what most ML systems do. It does not make the conclusions *true*; it makes them *checkable*.

### 4. Self-claims vs behavioral signal: the most important separation

Here is the most pernicious failure mode in LLM personality tools, and the one auspex is most aggressive about avoiding.

When someone writes "I'm an INTJ" or "I have an IQ of 145" or "I'm just a really analytical person" in a chat, a naive pipeline will read those statements, include them in the context window during synthesis, and produce a profile that confirms the self-description. The profile says the person is INTJ; the evidence cited is them saying they're INTJ. Circular. Worthless.

This is how you get LLM-generated personality reports that everyone nods along to, because the model is essentially reflecting your stated identity back at you with new vocabulary.

auspex breaks the circle. Phase 0 explicitly classifies every message and detects when it's a *self-statement*. Self-statements are then **pulled out of the trait-extraction pipeline entirely** and routed into a separate reconciliation stream (Phase 5c). Each self-claim is logged, the dimension it claims about is identified (intelligence, profession, mood, identity, etc.), and the claim is reconciled against the person's *actual behavioral evidence* in the rest of their messages. The verdict is one of:

- **`consistent`** — behavior matches the claim
- **`inconsistent`** — behavior contradicts the claim
- **`unverifiable`** — no behavioral evidence either way
- **`not-literal`** — the claim was made ironically or as hyperbole

This means a profile cannot say "this person is intelligent because they say they are." It can say "this person *claims* high intelligence, and their behavioral evidence on that dimension is `consistent` / `inconsistent` / `unverifiable`." Both signals are visible. The tool refuses to silently merge them.

If you find yourself reading an auspex profile and feeling that it's "spot on" — good. If you find yourself reading one and feeling defensive because the verdict is `inconsistent` for something you say about yourself often, that's the tool working as designed. Self-perception and observed behavior are not the same thing, and a tool that pretends they are is doing you no favors.

### 5. Temporal awareness: people aren't constants

A person who was in a depressive episode in early 2024 and is largely fine now should not be profiled as "exhibits depressive rumination" with no qualification. That description averages out their current state. It's also not actionable — the most important question is usually not "what is this person on average across all the time you've known them" but "what is this person *now* and what is *changing*."

auspex splits every measurement into two windows: **the full corpus (baseline) and the most recent quartile by timestamp.** Cognitive markers are computed over both. Big Five citation panels are populated from both. Every theme tracks `recent_share` — the fraction of supporting messages that fall in the recent quartile — and gets a `temporal_status`:

- **`active`** — recent_share ≥ 0.40 (the pattern is currently load-bearing)
- **`stable`** — between 0.10 and 0.40 (consistent across time)
- **`fading`** — recent_share ≤ 0.10 (mostly old evidence; the person may have moved past this)

The synthesis prompt is explicitly told: *write active themes in present tense, write fading themes in past tense.* If a profile says "they have moved past their period of avoidant conflict-handling," that means recent_share is low for that theme — not just that the LLM felt poetic.

This matters because the highest-value thing the tool produces is the **insight feed** — a ranked list of what's *changing* run-over-run. Relationship cooling. Theme reactivation. Tone shift toward a specific person. Cognitive marker drift. The static profile is a snapshot; the temporal layer is where the actual intelligence lives.

### 6. Cross-person calibration (z-scoring)

Cognitive marker numbers — abstract rate, conditional rate, integrative complexity, lexical complexity, domain breadth, self-monitoring — would be **meaningless as absolute scores**. "0.34 integrative complexity" doesn't mean anything. Compared to what? Compared to a published corpus? You don't have one. Compared to humans in general? Chat language doesn't generalize.

The only honest reference frame for these numbers is *the other people in your own corpus.* After every person's profile is computed, auspex z-scores each marker across the population: subtract the mean, divide by the standard deviation. The result is in standard-deviation units relative to your own network.

So a profile that reads `z_integrative: +1.8` means: *this person scores 1.8 standard deviations higher on integrative complexity than the average member of your network.* That is a defensible statement. "Their IQ is 137" is not.

This also means: **the same person looks different in different networks.** Someone who is +1.5σ in their family group chat might be -0.3σ in a research lab chat. That's correct. The tool measures relative position; relative position depends on who you compare against.

Practical implication: don't take the cognitive markers as fixed properties. They're a comparison against the specific people in your specific corpus.

### 7. Pairwise dynamics: the network is the unit of analysis

If you only look at individual profiles, you are missing most of what's actually going on. Real life happens *between people*, not inside them. A pair-level measurement captures things that no individual-level summary can:

- **Reply latency asymmetry**: Alice replies to Bob in 2 minutes. Bob replies to Alice in 6 hours. Who is invested in whom?
- **Tone targeted at specific people**: Charlie's general tone is "warm." But Charlie's messages addressed-to-Diana-specifically are dominantly "frustrated" and "dismissive." Charlie has a problem with Diana — invisible if you only read Charlie's individual profile.
- **Topic overlap**: Bob and Alice talk about climbing constantly. Bob and Eve never share a single topic cluster. Bob/Alice are forming a sub-community. Bob/Eve are not.
- **Initiation balance**: Diana starts 80% of conversations with Eve. Eve never initiates. One-sided emotional investment.
- **Mention pattern**: Alice talks *about* Bob frequently in conversations with Charlie. *What* is she saying — warmly? as complaint? as comparison? (Phase 0 captures tone per mention).

auspex computes all of this per pair, with a baseline-vs-recent split. The graph in the UI is a real network model: edges derived from these pairwise interaction patterns, with thickness representing intensity and color representing warmth (green/red/neutral). You can click any edge to see the full directional breakdown — A→B vs B→A — for both windows.

This is also where the **insight engine** lives. Insights like `relationship_cooling`, `tone_shift_toward`, `asymmetric_investment`, and `alliance_forming` are diff-based: they fire when the pairwise state changes meaningfully between runs. These are the most actionable outputs the tool produces — the kind of thing that genuinely informs how you behave in your network — because they describe *what changed in the relationship*, not just *what each person is like*.

### 8. The insight engine philosophy

The insight feed is not a list of facts about your network. It is a list of **deltas worth your attention**, ranked by computed urgency.

The reasoning: a static profile is a low-bandwidth artifact — you read it once, get a sense, move on. A delta against your previous state is a high-bandwidth artifact — it tells you what to actually look at *right now*. If nothing in your network changed since the last run, the insight feed is empty, and that's correct. The tool is not trying to invent intelligence where none exists.

Each insight carries:

- A **type** (one of ~13 categories: new_theme, theme_status_change, confidence_jump, cognitive_shift, new_self_claim, claim_verdict_change, high_stakes_claim, relationship_cooling, relationship_warming, tone_shift_toward, asymmetric_investment, alliance_forming, new_pair)
- A **summary line** — what changed, in one sentence
- **Details** — a few sentences of context
- **Evidence** — the cited messages that triggered it
- An **urgency score** between 0 and 1, computed from delta magnitude × baseline confidence × novelty

The urgency score is what determines feed ordering. High-stakes claims (suicidal ideation, "I'm done with this," abrupt life-decision language) score around 0.85. Relationship-cooling on a pair with substantial prior interaction scores around 0.55-0.75. Minor cognitive marker drift scores around 0.30.

The "Radar" panel in the UI is just the top-30 insights, color-coded by urgency band: red (critical, ≥0.75), gold (high, ≥0.55), blue (medium, ≥0.35), gray (low, below 0.35). Click any insight and the person's sidebar opens with the evidence.

**This is the actual product.** The static profiles are an intermediate representation. What you should be opening auspex for, after the first run, is *to see what changed.*

### 9. How to read each section of the output

Concrete guidance for interpreting what the UI shows you:

**Cognitive markers (z-scored bars in the sidebar).** A bar showing z = +1.5 means this person is 1.5 standard deviations above the corpus mean on that marker. Roughly speaking, +1 is "noticeably above average for your network," +2 is "remarkably above," -1 is "noticeably below." Don't read these as IQ proxies. They are descriptive of one cognitive-style axis at a time, with all the language and register caveats from sections 1 and 6.

**Big Five citations.** Each dimension shows the top 5 messages that exemplify it most strongly *for this person*, plus an indicator of how many candidate messages there were. **There is no score.** Read the quotes. Decide for yourself whether they amount to "high openness" or just "they happened to write five messages with abstract-ish vocabulary." This is the most honest framing of Big Five from text — show the evidence, refuse the number.

**Self-claim panel.** Each claim is tagged with its dimension, its register (serious / ironic / hyperbole), and a verdict against behavioral evidence. Pay attention to mismatches: a `claim_register: serious` paired with `verdict: inconsistent` is real signal — the person sincerely believes something about themselves that their behavior contradicts. That's interesting. A `not-literal` verdict on a hyperbolic claim is the tool correctly refusing to treat exaggeration as a literal claim; ignore it.

**Theme confidence.** A theme with confidence 0.70+ means the falsification probes mostly came back negative — the pattern survived adversarial probing. Below 0.50, treat as a tentative hypothesis. Below 0.30, the theme is essentially refuted; it's only still in the list because the surface evidence count was high enough to surface it.

**Pair edges.** Edge thickness encodes interaction intensity. Edge color encodes pairwise warmth: green for net-warm tone, red for net-cold, blue/gold for neutral (gold for edges touching you). Click an edge for the directional breakdown — A→B and B→A are shown separately, because they're often different.

**Temporal status badges.** `[ACTIVE]` next to a theme name means it's load-bearing in the recent quartile. `[FADING]` means it's mostly old evidence. These badges should drive your reading: an active theme is current-state; a fading theme is historical context.

**Insight urgency colors.** Red = pay attention now. Gold = worth knowing. Blue = noise-level but recorded. Gray = nearly noise; included for completeness.

### 10. Good uses and bad uses

What the tool is genuinely useful for:

- **Self-knowledge.** Profile yourself first. Read the themes against your own behavioral evidence. Notice where the verdicts surprise you. Notice which active themes describe how you are *now* versus how you think of yourself.
- **Understanding the texture of your friend group.** With cross-person z-scoring, you can see who occupies which role in your network — who's the highest-affect, who's the most cognitively diverse, who's the most other-focused. These are descriptive observations about your social ecology, not judgments.
- **Spotting drift in relationships.** This is the killer feature. You re-run after a few weeks of new data. The insight feed surfaces `relationship_cooling` between two people you didn't realize were drifting apart, or `alliance_forming` between two people whose interaction has spiked. You see it before you would have noticed it.
- **Reflective practice.** Periodic re-runs give you a record of how your network state is changing over time. The accumulating insight feed becomes a kind of social-life journal — not a substitute for actual journaling, but a quantitative complement.

What the tool is *not* useful for, and you should refuse to use it for:

- **Hiring decisions.** Construct validity is too low. Behavioral patterns in chat do not generalize to job performance.
- **Diagnosing mental health.** auspex will detect language patterns that overlap with depression markers, anxious attachment markers, etc. — but those overlaps are not diagnoses. Refer to a clinician.
- **High-stakes predictions about specific actions.** "Will this person quit their job" type forecasts are speculative even with good data and a good model. Treat predictions as hypotheses worth considering, not commitments.
- **"Reading" people you barely know.** The pipeline depends on having a substantial corpus per person. With <100 substantive messages it produces low-confidence output that should be ignored.
- **Manipulation.** This should be obvious. The tool surfaces signal about people in your network. What you do with that signal is on you. Using it to engineer conversations in bad faith is not the use case it's built for.

### 11. The construct-validity wall, restated

If you take only one thing from this primer, take this: **auspex measures textual signatures of communicative behavior, not personality or character.** The correlation ceiling between chat-derived traits and self-report inventories is, optimistically, around 0.4. The correlation between chat-derived traits and *who someone actually is offline* is unknown and probably lower.

Every confident-sounding paragraph the tool produces is bounded by that ceiling. Read with skepticism. Reject claims that don't match the cited evidence. Trust the deltas more than the static labels — what's *changing* in a person's messaging is more reliably real than what someone *is*, because change is anchored in observable temporal signal.

The tool's discipline — provenance, falsification, separated streams, z-scoring, no IQ — is designed to make that ceiling visible and to prevent overclaiming past it. Use it accordingly.

## Module layout

```
src/
├── main.rs       347   entry point, CLI, pipeline wiring
├── config.rs      33   Config + load_config
├── parse.rs       80   parse_files, alias resolution, msg_id hashing
├── lexicons.rs   142   loader for lexicons/ (word lists + intent phrases)
├── types.rs      400   all serializable data structs
├── math.rs        58   cosine_sim, vec_mean, kmeans
├── index.rs      137   embedding index + on-disk persistence
├── persist.rs     19   save_json / load_json for per-profile cache
├── llm.rs         73   Ollama JSON + text wrappers
├── metrics.rs    658   intent / behavioral / interaction / pairwise computation
├── pipeline.rs  2210   phases 0–6, profile_person, cross-person calibration
├── insight.rs    610   snapshots + generate_insights + generate_pair_insights
├── html.rs       786   graph.html template (D3 + radar + pair panel + chat)
└── http.rs       254   pure-std HTTP server + Ollama proxy
```

Dependency direction is acyclic. Every module is `pub(crate)`-only — no public API surface, no semver shenanigans, no library/binary split. It's a tool, not a framework.

## Configuration

### `config.json` (gitignored — see `config.example.json`)

```json
{
  "self_name": "you",
  "self_handles": ["you", "Me", "YourPrimaryHandle"],
  "aliases": {
    "alice": ["AliceSmith", "alicia_s", "Alice (work)"]
  }
}
```

### Environment variables

| Var | Default | Purpose |
|---|---|---|
| `OLLAMA_MODEL` | `llama3.2` | Used for every LLM call. `gpt-oss:20b` recommended for synthesis depth; `qwen2.5:7b` / `aya-expanse:8b` / `llama3.1:8b` if you want speed over depth and your corpus is multilingual. |
| `AUSPEX_PORT` | `8765` | HTTP server port. |
| `AUSPEX_NO_SERVE` | unset | Set to anything to skip the embedded server. |

### Lexicons (`lexicons/` — gitignored)

Word lists and intent phrases live on disk as plain text, not in source. One entry per line, `#` for comments, blank lines ignored. The loader merges all `.txt` files under each category — file stems are language labels for diagnostic logs but the loader is language-agnostic. Add a language by dropping in `lexicons/self_ref/<whatever>.txt` and re-running. `lexicons.example/` ships as a thin English-only template; your actual lexicons are yours.

## What this refuses to do

These are not omissions. They are positions.

- **No IQ score.** Chat-lexicon → IQ has no defensible psychometric grounding. Anyone selling you one is selling you a story. The cognitive marker section reports z-scored signals you can interpret; the rest is your judgment.
- **No MBTI type.** Same logic. Big Five gets used as a *framing* because it's less unscientific, but even Big Five is presented as *citations*, not numbers — chat-derived Big Five correlates only ~0.3–0.5 with self-report inventories.
- **No refusing to comment.** The chat panel's system prompt tells the model in no uncertain terms: the user owns this data, is asking analytical questions about their own life, refusing to compare or analyze is the wrong answer. If you ask "who in my network is most likely to flake on me," it answers from the actual data and names names.
- **No silent self-claim folding.** Saying "I'm an analytical thinker" in chat does not produce a profile that says "this person is an analytical thinker." Self-statements are a separate reconciliation stream. The tool measures what you do, not what you claim about yourself.
- **No telemetry. No remote calls. No cloud sync. No update server. No analytics. No third-party API.** Everything stays local, every artifact is on your disk and you can read every byte. The HTTP server only listens on `127.0.0.1`. If you airgap the machine, nothing breaks.

## Honest limits

I'm being explicit about these because most tools in this space hide them.

- **Construct validity is unverified.** The tool measures *textual signatures* of personality and *behavioral patterns over time.* Whether these map to what someone "really is" offline has not been benchmarked against any standardized inventory. Treat outputs as one disciplined view of a chat-message stream, not as ground truth about a person.
- **Pair tone-toward-specific-person uses an adjacency heuristic.** When A's message is classified `addressee: specific`, the tone is attributed to whoever A just replied to in chronological order. Wrong for interleaved group chat where the prior message wasn't the intended target. Real reply-threading is on the roadmap.
- **First-run insights are sparse.** Most insight types are diff-based — they need a prior snapshot to compare against. On the very first run you only get `high_stakes_claim` insights (those don't need diffs). The tool is most valuable on update #2 onward.
- **Chat does not scale infinitely.** The system prompt dumps every profile into context. At ~50 people you're near the upper limit of a 128k-context model. Retrieval-augmented chat is on the roadmap.
- **LLM-dependent.** Phase 0 quality determines everything downstream. A weak model on your language(s) cascades. Test on a sample before committing to a multi-hour run.

These are real. None of them are fixed by hedging — they're fixed by either future work or by you reading the output skeptically.

## Performance / cost shape

Per person with N substantive messages:

| Phase | Calls | Cost |
|---|---|---|
| 0 — classify | ~N / 10 | ~10s per batch of 10 on a 20B local model |
| 1 — observations | ~N / 6 | similar |
| 3 — deepen | ~10 | ~10s each |
| 4 — falsify | ~240 | small per-candidate yes/no |
| 5a/b — markers | 0 LLM | derived from Phase 0 |
| 5c — self-claims | ≤30 | per non-ironic claim |
| 5 / 6 — synthesis & predict | 2 | |

For 5,000 substantive messages: ~1,500 calls, ~2 hours on local 20B. **Re-runs are vastly faster** — Phase 0 only classifies new messages, Phase 1 only scans new substantives, downstream phases reuse cache when upstream didn't change, fingerprint short-circuits skip the whole pipeline for unchanged people.

`--serve` is instant. No pipeline, just the UI.

## Who this is for

- People who keep their own chat archives and want to actually understand the network they're embedded in.
- Engineers who care about provenance, falsification, computed confidence, and incremental computation.
- Anyone tired of LLM tools that produce smooth nonsense and call it analysis.
- People who'd rather have an honest measurement of "this person is +1.8σ on multi-perspective relative to your corpus" than a fake IQ number.

## Who this is not for

- People who want a polished marketing-style personality assessment to share at a dinner party.
- People who think any kind of structured analysis of communication is unethical on its face. (The author thinks the ethical question is real and worth taking seriously — but the answer "refuse to look at your own data" isn't engaging with it.)
- People who want black-box "AI" to make decisions for them. This tool surfaces signal; it does not tell you what to do with it.

## Stack

- Rust — single binary, ~7,300 LoC across 14 modules
- fastembed (5.x) — local AllMiniLML6V2 embeddings, no GPU required
- regex, serde, serde_json, ureq — all the deps
- `std::net` — embedded HTTP server, no tokio, no hyper, no axum
- D3.js + marked.js — UI delivered inline by the binary, no build step
- Ollama — local LLM
- **Zero remote anything. Zero telemetry. Zero update mechanism.**

## Roadmap

What's planned:
- Resolve `addressee` to actual named recipient so pair-tone detection stops relying on adjacent-message order.
- Retrieval-augmented chat for >100-person networks.
- Triangulation insights (A talks about C to B is already counted; just needs surfacing).
- Multi-model agreement check on synthesis paragraphs.

What's deliberately NOT planned:
- Cloud sync. Telemetry. Update server. Crash reporter. Analytics. "AI safety" filters that refuse questions. Sign-in. Feature flags. A/B tests.

## License

TBD.
