# n8n Daily News Audio Digest

An n8n workflow that reads RSS feeds every morning, filters articles using a local LLM, converts the best ones into a natural audio script, and outputs an MP3 — ready to listen to during your commute or coffee.

## What it does

1. **Fetch** — pulls the latest articles from three RSS feeds (Quanta Magazine, Il Post Mondo, Il Post Italia)
2. **Scrape** — fetches and cleans the full article HTML for each item
3. **Filter & Score** — a local Ollama model (`llama3.1:8b`) evaluates each article for topic relevance and depth, returning a `keep` flag and a 0–10 score
4. **Select** — keeps only substantive articles, sorts by score, takes the top 4
5. **Script** — Groq (`llama-3.3-70b-versatile`, 128K context) converts each article into natural spoken language, preserving all detail
6. **Combine** — the four per-article scripts are joined into one continuous text
7. **TTS** — OpenAI TTS (`tts-1-hd`, voice `nova`) renders the script to an MP3

The workflow runs automatically every day at 7am, or can be triggered manually for testing.

## Pipeline diagram

```
[Schedule / Manual]
        │
        ▼
[Quanta + ilPost Mondo + ilPost Italia RSS]
        │
        ▼
[Combine All Feeds] → [Remove Duplicates]
        │
        ▼
[Fetch Article Content] → [Extract Article Text] → [Clean Article Text]
        │                                                    │
        │◄───────────────────────────────────────────────────┘
        ▼
[Filter & Score]  ←── Ollama llama3.1:8b (local)
        │
        ▼
[Merge Article + Scores] → [Flatten Output]
        │
        ▼
[Keep High Quality] → [Sort by Score] → [Top 4 Articles]
        │
        ▼
[Generate Audio Script]  ←── Groq llama-3.3-70b-versatile
        │  (runs once per article)
        ▼
[Combine Scripts] → [Text to Speech] → MP3
```

## Requirements

| Service | Purpose | Cost |
|---|---|---|
| [Ollama](https://ollama.com) | Article filtering (local) | Free |
| [Groq](https://console.groq.com) | Script generation | Free tier |
| [OpenAI](https://platform.openai.com) | Text-to-speech (`tts-1-hd`) | Paid |
| [n8n](https://n8n.io) | Workflow engine | Self-hosted or cloud |

## Setup

### 1. Install Ollama and pull the model

```bash
# Install Ollama: https://ollama.com/download
ollama pull llama3.1:8b
```

### 2. Create credentials in n8n

Go to **Settings → Credentials → New credential** and create:

- **Groq** — paste your API key from [console.groq.com](https://console.groq.com)
- **OpenAI** — paste your API key from [platform.openai.com](https://platform.openai.com)

### 3. Import the workflow

1. In n8n, go to **Workflows → Import from file**
2. Select `My workflow.json`
3. Open the **Groq (Script)** node and assign your Groq credential
4. Open the **Text to Speech** node and assign your OpenAI credential
5. Activate the workflow

### 4. (Optional) Customise the RSS feeds

The workflow currently reads:
- `https://api.quantamagazine.org/feed/` — science & mathematics
- `https://www.ilpost.it/mondo/feed/` — international news (Italian)
- `https://www.ilpost.it/italia/feed/` — Italian news

Edit the RSS Feed nodes to point to any feeds you prefer.

### 5. (Optional) Customise the filter

The **Filter & Score** node prompt defines what to keep and what to drop. Edit it directly in n8n to match your interests. The key principle baked in: the article must itself contain substance — not just name interesting topics.

## Design notes

**Why Ollama for filtering, Groq for scripting?**  
Filtering is a short classification task (title + article → JSON). A local 8B model handles it well with zero API cost and no rate limits. Script generation needs a large context window (full article text) and higher output quality — Groq's free tier with `llama-3.3-70b-versatile` (128K context) covers both without cost.

**Why one LLM call per article instead of batching?**  
Batching all four articles into one prompt caused context exhaustion and truncated output on smaller models. Processing each article separately guarantees full attention and output space for every piece.
