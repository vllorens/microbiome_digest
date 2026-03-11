# Microbiome digest

A fully automated daily podcast for microbiome researchers. Every morning at **03:00 UTC**, a GitHub Actions workflow wakes up on GitHub's servers, runs the entire pipeline without any computer needing to be on, and publishes a fresh episode to GitHub Pages.

Please consider the first podcasts as *experimental*, as I polish and fine-tune the sources and paper rakings. 


## Credit

All credit goes to: 

* https://github.com/WenyueDai/protein_design_podcast, which is the original podcast from where I took all the code and info. This is just my adaptation to the microbiome field, tailored to the interests of our lab. 

* Claude code, for getting this up and running fast!



**Live site:** [vllorens.github.io/microbiome_digest](https://vllorens.github.io/microbiome_digest)

**Paper collection (Notion):** [click here](https://www.notion.so/31ff516be8ec807fb949ecadf0aab40c?v=31ff516be8ec8053aa74000cee39b8e9&source=copy_link)

**Deep dive notes (Notion):** [click here](https://www.notion.so/31ff516be8ec806aaf20fe60adf931b0?v=31ff516be8ec80569172000c737f8643&source=copy_link)


---

## Full End-to-End Workflow

### Phase 1 — Paper Collection (03:00 UTC)

GitHub Actions checks out the latest `main` branch and runs `python run_daily.py`.

An **idempotency guard** at the start of `run_daily.py` checks `state/release_index.json`: if today's date already has an entry, the run exits immediately without repeating work.

**1a. RSS feeds** (`src/collectors/rss.py`)
Fetches RSS/Atom feeds simultaneously, grouped into:

- **Core microbiome** — Cell Host & Microbe, Nature Microbiology, Microbiome (BioMed Central), Gut (BMJ), ISME Journal, Gut Microbes, mBio, mSystems, and related journals
- **Top journals** — Nature Communications, Nature Medicine, Nature Methods, PNAS, Genome Biology, Gastroenterology
- **Computational / omics** — bioRxiv Microbiology, bioRxiv Genomics, bioRxiv Bioinformatics, medRxiv Gastroenterology, medRxiv Oncology
- **Key researchers (absolute priority, tier 0)** — bioRxiv author feeds for tracked researchers including Nicola Segata, Curtis Huttenhower, Peer Bork, Ruth Ley, Jeffrey Gordon, Jennifer Wargo, Georg Zeller, and others
- **Blogs (absolute priority, tier 1)** — sources tagged `author` without a preprint-server source name

Each item gets a `bucket` tag: `microbiome`, `omics`, `computational`, `engineering`, or `clinical`.

**1b. PubMed search** (`src/collectors/pubmed.py`)
Runs keyword queries against the PubMed E-utilities API, organized by category:
- **General microbiome** — "human gut microbiota", "human gut microbiome", "host-microbiome interaction"
- **Functionality & multi-omics** — "gut microbiome functionality", "gut metatranscriptomics", "multiomics microbiome"
- **Computational** — "microbiome analysis tool", "microbiome bioinformatics", "benchmark microbiome"
- **Cancer** — "microbiome colorectal cancer", "gut microbiome CRC", "fusobacterium nucleatum cancer"
- **Other disorders** — "gut microbiome IBS", "gut microbiome IBD"
- **Engineering** — "CRISPR gut microbiome", "genetic engineering microbiome", "gut microbiome modulation"
- **Specific organisms** — "lacticaseibacillus rhamnosus GG", "fusobacterium nucleatum"

Returns articles published in the last 2 days.

**1c. bioRxiv author tracking** (`src/collectors/biorxiv_authors.py`)
Fetches recent bioRxiv and medRxiv preprints and filters them against a list of tracked researchers. Items matched this way get the `author` tag and are treated as **tier-0** by the ranker.

**1d. Deduplication**
Every item URL is checked against `state/seen_ids.json`, which persists across days. Items seen in previous runs are dropped. New items are added to seen_ids at the end of the run, so the podcast never repeats content. Runner-up articles that don't make the episode cap are intentionally kept unseen so they remain available for quieter days.

**1e. Content filtering**
Items whose title, source, or URL contain any term from `excluded_terms` in `config.yaml` are dropped (e.g. "mouse", "single-cell", "neurogenesis").

---

### Phase 2 — Analysis & Ranking

**2a. Parallel article fetch + LLM analysis** (`src/processing/article_analysis.py`)
Up to 8 articles are fetched and analyzed in parallel using a `ThreadPoolExecutor`. For each article, `newspaper4k` + `BeautifulSoup` extract the full text from the paper's webpage. A fast LLM (OpenRouter `stepfun/step-3.5-flash:free`) then reads the text and returns a structured analysis: core claim, novelty, and relevance score.

**2b. Ranking** (`src/processing/rank.py`)
Items are sorted by a 10-level priority key (lower = better):

| Tier | Factor | Rationale |
|------|--------|-----------|
| 0 | **Researcher feeds** — tracked authors on bioRxiv/medRxiv/arXiv (`author` tag + preprint server source) | Curated, highest trust — new papers from tracked researchers always appear first |
| 1 | **Blog/substack sources** (`author` tag, non-preprint source) | High-quality curated writing, just below new research |
| 2 | **Absolute title keywords** — "gut microbiota", "gut microbiome", "metatranscriptomics", "multiomics", "host-microbiome" | Landmark papers regardless of source |
| 3 | **Missed paper keywords** — topics extracted from owner-submitted missed papers (`boosted_topics.json`) | Ground truth: papers actively sought out that the pipeline failed to collect |
| 4 | **Graded feedback score** (time-decayed) — liked sources/keywords compound over time (range −10…0) | Personalized boost; frequency-weighted with 14-day half-life so interests drift naturally |
| 5 | **Config topic keywords** — "metatranscriptomics", "multi-omics microbiome", "CRC microbiome", "colorectal cancer microbiome", "IBS microbiome", etc. | Broad topic steering from `config.yaml` |
| 6 | **Journal quality** — driven by `source_priority_rules` in `config.yaml`; Cell Host & Microbe / Nature Microbiology / Nature Biotech > Nature Communications / PNAS / Genome Biology > field-specific journals | Source credibility |
| 7 | **Research bucket** — microbiome/omics/computational/engineering before clinical | Domain relevance |
| 8 | **Fulltext available** — papers where full text was successfully extracted | Content quality |
| 9 | **Extracted text length** | Final tie-breaker |

Tier-0 (researcher feeds) and tier-1 (blogs) items are **hoisted to the front** before any bucket quota is applied. Remaining items then have bucket quotas applied: up to 30 `microbiome`, 5 `clinical`. Total episode cap: 40 items.

---

### Phase 3 — Script Generation

**3a. LLM script writing** (`src/processing/script_llm.py`)
The ranked items are sent to the main LLM (`arcee-ai/trinity-large-preview:free` via OpenRouter). For each paper, the LLM writes:
- A **deep-dive segment** (~220–340 words): background, methodology, findings, significance
- A **roundup blurb** (~80–130 words): quick summary for papers without full text

Segments are joined with `[[TRANSITION]]` markers. The final script is saved as `output/DATE/podcast_script_DATE_llm.txt`.

---

### Phase 4 — Text-to-Speech

**4a. One MP3 per segment** (`src/outputs/tts_edge.py`)
The script is split on `[[TRANSITION]]` markers into individual segments. Each segment is converted to a separate MP3 using Microsoft Edge TTS (voice: `en-GB-RyanNeural`, rate: `+0%`). Edge TTS is free and runs over a WebSocket to Microsoft's servers.

- If Edge TTS fails or produces a corrupt file, it retries up to 3 times with fallback voices
- If all retries fail, it falls back to gTTS (Google TTS, lower quality but reliable)
- Existing valid MP3s are reused on re-runs (skip if file exists and passes ffprobe validation)

**4b. Concatenation with transitions** (`src/outputs/audio.py`)
All segment MP3s are concatenated by ffmpeg with a short transition sound between each paper:
```
[1.0s silence] → [ding C6, 0.12s] → [gap 0.06s] → [ding E6, 0.12s] → [1.0s silence]
```
The entire output is sped up by `atempo=1.2` (20% faster playback). Final file: `output/DATE/podcast_DATE.mp3`.

**4c. Timestamp calculation**
For each segment, the raw cumulative position is measured using `mutagen` (frame-accurate for VBR MP3). Timestamps are stored in `output/DATE/episode_items.json` so clicking a paper on the website seeks the audio player to exactly 0.5 seconds before the transition tones for that paper.

---

### Phase 5 — Publishing

**5a. GitHub Release** (`src/outputs/github_publish.py`)
The pipeline calls the **GitHub REST API** to:
1. Create a new GitHub Release tagged `episode-DATE` on `vllorens/microbiome_digest`
2. Upload `podcast_DATE.mp3` as a release asset

The MP3 is served directly from GitHub's CDN via the release asset URL. If a release already exists for today, the MP3 upload is skipped (set `FORCE_REPUBLISH=true` to override).

**5b. GitHub Pages site rebuild** (`tools/build_site.py`)
`build_site.py` is called to regenerate the `docs/` folder:
- Reads `state/release_index.json` to know which audio URLs are available
- Reads `output/DATE/episode_items.json` for paper titles, timestamps, and summaries
- Reads `state/paper_notes.json` to bake any owner notes into the HTML
- Reads `state/missed_papers.json` to bake submitted missed papers into the HTML
- Writes `docs/index.html` (the main site), `docs/feed.xml` (RSS podcast feed), `docs/cover.svg`

Audio uses `<source type="audio/mpeg">` for iOS Safari compatibility.

**5c. Notion digest** (`src/outputs/notion_publish.py`)
Creates a new page in the Paper Collection Notion database summarizing today's episode, with sections: **Microbiome & Research** and **Clinical**.

**5d. Git commit and push**
The GitHub Actions workflow commits all changed files back to `main`:
```
state/seen_ids.json        ← updated with today's paper URLs
state/release_index.json   ← updated with today's audio URL
output/DATE/               ← episode items, status, script
docs/                      ← rebuilt GitHub Pages site
```
GitHub Pages detects the change to `docs/` and automatically redeploys the website within ~30 seconds.

---

### Phase 6 — Interactive Features (browser-side)

These happen **in your browser**, not on GitHub's servers.

**6a. Clicking [N] to seek audio**
Each paper number `[N]` on the site is a `<span>` with `onclick="seekTo(this, event)"`. Clicking it sets `audio.currentTime = timestamp` where the timestamp was pre-calculated in Phase 4c. The audio player jumps to 0.5s before the transition tones for that paper.

**6b. Submitting a missed paper** (owner only)
The "Submit a missed paper" section at the bottom of the page is for the owner's use only. The JS:
1. Reads your GitHub token from `localStorage` (set once in ⚙ Settings)
2. Calls `GET /contents/state/missed_papers.json` to check for duplicate titles
3. Appends the entry and calls `PUT` to commit it directly to GitHub

This commit **immediately triggers** the `process_missed.yml` workflow (see Phase 7), so diagnosis and a Notion stub appear within ~2 minutes.

**6c. Saving feedback** (owner only)
Checking paper checkboxes and clicking "Save feedback" triggers JavaScript that:
1. Reads your GitHub token from `localStorage`
2. Calls `GET /contents/state/feedback.json` to fetch the current file + its SHA
3. Merges your new selections into the existing data
4. Calls `PUT` to commit the change

The next day's pipeline reads `feedback.json` and uses it to apply a **time-decayed ranking boost** (tier 4): source clicks and title keywords each contribute a frequency-weighted score with a 14-day half-life.

**6d. Writing "My Take" notes** (owner only)
Clicking ✏️ next to a paper opens an inline textarea. Saving writes to `state/paper_notes.json`. This commit **automatically triggers** the `sync_notes.yml` workflow (see Phase 8).

---

### Phase 7 — Missed Paper Processing (`process_missed.yml`)

Whenever the owner submits a missed paper, the **`process_missed.yml`** workflow fires immediately (triggered by a push to `state/missed_papers.json`). It also runs as part of the daily pipeline.

1. **Diagnose** each unprocessed entry:
   | Diagnosis | Meaning |
   |-----------|---------|
   | `already_collected` | The URL's SHA1 was already in `seen_ids.json` — paper ran in a previous episode |
   | `excluded_term` | An `excluded_terms` keyword matched the title |
   | `source_not_in_rss` | The URL's domain is not in any configured RSS feed |
   | `low_ranking` | The source domain is in RSS feeds but the paper was cut below the episode cap or wasn't in the recent 24h window |

2. **Extract keywords** (for `low_ranking` and `source_not_in_rss`): calls OpenRouter LLM to extract 3–5 topic phrases from the title. These are merged into `state/boosted_topics.json`. The next daily run's ranker loads these as **tier-3 priority**.

3. **Discover RSS feed** (for `source_not_in_rss`): probes common feed paths on the paper's domain. If a valid feed is found, it is saved to `state/extra_rss_sources.json`.

4. **Create Notion stub**: creates a page in the Deep Dive Notes database with the diagnosis, keywords boosted, and a bookmark to the paper.

5. Rebuilds the site and commits `missed_papers.json`, `boosted_topics.json`, `extra_rss_sources.json`, and `docs/` back to `main` with `[skip ci]`.

---

### Phase 8 — Notion Deep-Dive Sync (`sync_notes.yml`)

Whenever `paper_notes.json` is updated, the **`sync_notes.yml`** workflow fires automatically:

1. Runs `tools/sync_notion_notes.py`
2. For each note not yet synced (checked against `state/notion_created.json`):
   - Creates a stub page in your Deep Dive database with your note, paper metadata, and a bookmark
3. Commits `notion_created.json` back to the repo (`[skip ci]` prevents an infinite loop)

---

## How GitHub Actions Works

GitHub Actions is a CI/CD platform built into every GitHub repository. For this project:

```
.github/workflows/
├── daily_podcast.yml    ← runs at 03:00 UTC daily
│                           includes: main pipeline + process_missed_papers.py
├── sync_notes.yml       ← runs whenever paper_notes.json is pushed (owner notes → Notion)
└── process_missed.yml   ← runs whenever missed_papers.json is pushed (immediate diagnosis)
```

Each workflow run gets a **brand-new virtual machine** with access to repository secrets. Runs for free on GitHub's servers (~10–15 min/day, well within the 2000 min/month free tier).

---

## Repository Layout

```
openclaw-knowledge-radio/         ← Python pipeline package
├── run_daily.py                  ← main entry point
├── config.yaml                   ← all settings (sources, limits, LLM, TTS)
├── requirements.txt
├── src/
│   ├── collectors/
│   │   ├── rss.py                ← RSS/Atom feed fetcher
│   │   ├── pubmed.py             ← PubMed E-utilities search
│   │   ├── biorxiv_authors.py    ← tracked researcher preprint fetcher
│   │   └── biorxiv_keywords.py   ← bioRxiv keyword filter
│   ├── processing/
│   │   ├── article_analysis.py   ← parallel LLM article analysis
│   │   ├── rank.py               ← 10-tier ranking (see Phase 2b)
│   │   └── script_llm.py         ← podcast script generation
│   └── outputs/
│       ├── tts_edge.py           ← Edge TTS → MP3 per segment
│       ├── audio.py              ← ffmpeg concat + atempo + transitions
│       ├── github_publish.py     ← GitHub Releases upload
│       └── notion_publish.py     ← Notion paper collection digest
├── tools/
│   ├── build_site.py             ← generates docs/ (HTML + RSS feed)
│   ├── sync_notion_notes.py      ← syncs owner notes → Notion deep-dive stubs
│   └── process_missed_papers.py  ← diagnoses missed papers + extracts boost keywords
├── state/
│   ├── seen_ids.json             ← URLs seen in previous runs (dedup)
│   ├── release_index.json        ← date → GitHub Release audio URL
│   ├── feedback.json             ← owner's paper selections (time-decayed ranking signal)
│   ├── paper_notes.json          ← owner's expert notes per paper
│   ├── notion_created.json       ← tracks which notes have been synced to Notion
│   ├── missed_papers.json        ← owner-submitted missed papers (with diagnoses)
│   ├── boosted_topics.json       ← keywords from missed papers (tier-3 ranking priority)
│   └── extra_rss_sources.json    ← RSS feeds discovered from missed paper URLs
└── output/YYYY-MM-DD/            ← per-episode data (kept 30 days)
    ├── podcast_YYYY-MM-DD.mp3    ← final audio
    ├── podcast_script_*_llm.txt  ← LLM-generated script
    ├── episode_items.json        ← paper list + timestamps
    └── status.json               ← run metadata

docs/                             ← GitHub Pages site (auto-generated, never edit manually)
.github/workflows/
├── daily_podcast.yml
├── sync_notes.yml
└── process_missed.yml
```

---

## Setup (for a new installation)

### 1. Fork / clone and install

```bash
git clone https://github.com/vllorens/microbiome_digest.git
cd microbiome_digest/openclaw-knowledge-radio
pip install -r requirements.txt
sudo apt install ffmpeg        # Linux
# brew install ffmpeg          # macOS
```

### 2. Environment variables

Create `openclaw-knowledge-radio/.env`:

```env
OPENROUTER_API_KEY=sk-or-v1-...
GITHUB_TOKEN=ghp_...
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...   # optional
NOTION_TOKEN=ntn_...
NOTION_DATABASE_ID=<your-daily-digest-db-id>
```

### 3. Run manually

```bash
cd openclaw-knowledge-radio
set -a && source .env && set +a
python run_daily.py
```

Optional flags:
```bash
REGEN_FROM_CACHE=true python run_daily.py    # reuse today's cached items (skip re-fetch)
DEBUG=true python run_daily.py               # skip seen-URL dedup
RUN_DATE=2026-02-20 python run_daily.py      # generate for a specific past date
FORCE_REPUBLISH=true python run_daily.py     # re-upload MP3 even if release already exists
```

### 4. GitHub repository setup

1. Create the repo `vllorens/microbiome_digest` on GitHub
2. Enable **GitHub Pages** from `Settings → Pages`, source: `docs/` folder, branch `main`
3. Add the following **Actions secrets** at `Settings → Secrets and variables → Actions`:

| Secret | Description |
|--------|-------------|
| `GH_PAT` | GitHub PAT with `repo` + `workflow` scopes |
| `OPENROUTER_API_KEY` | OpenRouter API key (LLM script + analysis + missed-paper keyword extraction) |
| `SLACK_WEBHOOK_URL` | Slack incoming webhook (optional) |
| `NOTION_TOKEN` | Notion integration token for the Paper Collection database |
| `NOTION_DATABASE_ID` | Paper Collection database ID |
| `NOTION_API_KEY` | Notion integration token for the Deep Dive Notes database |

### 5. Browser setup (owner interactive features)

On the GitHub Pages site, click **⚙ Settings** and enter:
- Your GitHub personal access token (`repo` scope)
- Your repo (`vllorens/microbiome_digest`)

This is stored only in your browser's `localStorage`. It enables the feedback checkboxes, ✏️ note buttons, and the missed paper submission form.

---

## Configuration Reference (`config.yaml`)

| Section | Key settings |
|---------|-------------|
| `limits` | `max_items_total` (40), `max_items_microbiome` (30), `max_items_clinical` (5) |
| `excluded_terms` | Keywords that filter out off-topic items (cell biology, neurogenesis, animal models, etc.) |
| `rss_sources` | Feeds with `name`, `url`, `bucket`, `tags`; sources tagged `author` get tier-0 or tier-1 absolute priority |
| `pubmed` | `search_terms`, `lookback_days`, `max_results_per_term`, `bucket` |
| `biorxiv_authors` | `authors` list with `name`, `match`, optional `institution`; `bucket`, `tags` |
| `podcast` | `voice`, `voice_rate`, `target_minutes` |
| `llm` | `model` (script: `trinity-large-preview`), `analysis_model` (per-article: `step-3.5-flash`) |
| `ranking` | `absolute_title_keywords`, `absolute_source_substrings`, `source_priority_rules`, `topic_boost_keywords` |

---

## Active features

- 10-tier ranking: researcher feeds → blogs → landmark titles → missed paper keywords → time-decayed feedback → config topics → journal quality → bucket → fulltext → length
- Tier-0/1 hoisting: researcher feeds (bioRxiv/medRxiv/arXiv author feeds) and blogs are always shown before any bucket quota is applied
- Configurable journal priority via `source_priority_rules` in `config.yaml`
- Missed paper keyword boost: topics from owner-submitted missed papers go into `boosted_topics.json` at tier-3
- Time-decayed feedback at tier-4: 14-day half-life; liked sources/keywords compound over repeated days
- Idempotency guard: pipeline exits immediately if today's episode is already published
- iOS Safari audio compatibility: `<source type="audio/mpeg">` ensures playback on iPhone/Safari
- Timestamp seeking: clicking `[N]` lands 0.5s before transition tones
- "My Take" notes: ✏️ button on each paper, saves to GitHub + triggers Notion stub creation
- Missed paper form: immediate diagnosis + Notion stub via `process_missed.yml`
- RSS discovery: `source_not_in_rss` papers trigger a feed probe; discovered feeds saved to `extra_rss_sources.json`
- Bucket quotas: up to 30 `microbiome` items + 5 `clinical` items, total cap 40
