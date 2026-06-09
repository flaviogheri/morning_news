Project: Automated morning news briefing (n8n)

## How to launch n8n

Start n8n locally:
```
npx n8n
```
Then open the editor at http://localhost:5678 in your browser. Your workflows and credentials are stored in ~/.n8n/ and persist across restarts.

To import the workflow: in the n8n UI, go to Workflows → Import from File and select `My workflow.json` from this project directory.
Goal: A workflow that runs every morning, gathers news from chosen RSS feeds, filters it down to genuinely interesting articles matching a specific taste, and produces a ~10-minute spoken-style script discussing the best 3–4 stories. (Eventually text-to-speech + delivery, but that's a later phase.)
Intended to run locally in future, so the cheap classification step should be swappable to a small local model (Ollama).
Confirmed sources (full-text verified)

Quanta Magazine — https://www.quantamagazine.org/feed (science/long-form, full text)
IEEE Spectrum — https://spectrum.ieee.org/feeds/feed.rss (robotics/engineering)
Il Post — https://www.ilpost.it/italia/feed/ and https://www.ilpost.it/mondo/feed/ (Italian general news)
Politico Europe was a candidate but untested; France 24 was dropped (snippets only, not full text).

Pipeline stages and the reasoning behind each
1. Trigger — Schedule Trigger (~7am), plus a Manual Trigger for testing.
2. Ingest — One RSS Read node per feed. Each node reads exactly one feed, so multiple feeds = multiple nodes.
3. Merge — Merge node (Append mode) collapses the parallel feeds into one item stream so downstream nodes act on all articles at once.
4. Dedup — Remove Duplicates, operation "Remove Items Processed in Previous Executions," comparing on article link/guid. Why: prevents re-processing (and re-paying for) articles seen on previous days. This handles cross-run state without needing a database.
5. Fetch full text — articles need their full body. Important cleanup problem: the current fetch returns the entire web page (nav bars, share buttons, "also in physics," etc.). A cleanup/extraction step is needed to reduce articleText to just the article body — otherwise we waste tokens and risk confusing the AI with boilerplate.
6. Topic classify (cheap pass) — Text Classifier node + a chat-model sub-node. Classifies each article into top-level IPTC Media Topics categories (the news industry's standard 17-bucket taxonomy). Discards unwanted buckets (crime, war, disaster, sport, etc.). Why cheap-first: mechanical-ish topic routing is cheap; it cuts volume before the expensive reasoning step. Note: AI nodes in n8n need a Model sub-node attached or they error with "A Model sub-node must be connected and enabled."
7. AI filter + rate (smart pass) — Basic LLM Chain, one call per surviving article. Returns JSON { keep: bool, score: 0–10, reason }. Why this is separate from topic classify: some exclusions aren't topics — they're entities ("Trump") or angles ("AI taking jobs") or quality judgments (the "is this insightful, Bill-Bryson-style?" test), none of which a topic classifier can express. Then a Filter node drops keep == false.
8. Select — Sort by score (desc) → Limit to top 3–4.
9. Generate script — Aggregate the survivors into one item → Basic LLM Chain prompt: "Write a ~1,400-word spoken discussion of these articles" (~1,400 words ≈ 10 min).
10. (Future) — TTS + delivery (email/audio).
The filtering taste (for the prompts in steps 6–7)
Wants: science, tech, robotics, economics/business, insight-driven explainer journalism (mechanism-and-meaning, "why X" pieces).
Discards: crime, death/murder, war, disaster, gossip/scandal/celebrities, "AI taking jobs," corruption/court cases, climate-change framing, refugees, Trump, sport (exception: Sinner and the Italy national team).
Current blocking error
"A Model sub-node must be connected and enabled" — the Text Classifier has no chat-model sub-node attached. Fix: attach an OpenAI/Ollama Chat Model to the Model connector and ensure it's enabled.
Two known issues to flag to Claude Code

Scraped page is full of boilerplate — needs a body-extraction step before classification (see stage 5).
Field consistency — Quanta, IEEE, and Il Post don't return identical fields; tag data is crude or absent (Quanta multimedia items have no useful tags, Il Post tags are literal nouns like "roma, cavalli"). So filtering can't rely on feed-provided tags — it must derive topic via the classifier.

That's the whole picture. One genuinely useful thing to keep in mind as you work with Claude Code: it'll happily generate the entire workflow JSON in one shot, but you've gotten this far by building and testing one node at a time — so consider having it help you stage-by-stage rather than dumping a finished pipeline you can't debug. Good luck with it.