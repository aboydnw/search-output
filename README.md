# Search Output

A living dataset of implementations, use cases, and impact signals for 
open-source tools built and maintained by Development Seed.

## Why this exists

We build and maintain open-source geospatial libraries used by teams around 
the world — but we've never had a reliable way to understand the full scope 
of that impact. Who is using our tools? How are they deploying them? What 
problems are they solving?

This repo is an experiment to answer that question systematically. An 
automated agent monitors the web daily, surfacing implementations, 
integrations, blog posts, and other signals that show how our work is being 
used in the wild.

## What's here

| Folder | Tool | Description |
|---|---|---|
| `data/titiler/` | TiTiler | GitHub repos, PyPI dependents, and web results referencing TiTiler |

More tools will be added over time.

## How it works

An [OpenClaw](https://openclaw.ai) agent runs on a remote VM and searches 
three sources daily:

- **GitHub Search API** — repositories referencing the tool
- **PyPI dependents** — packages that list it as a dependency  
- **Brave Search API** — broader web results including blog posts and deployments

New results are appended to the relevant CSV and pushed to this repo each 
morning. Duplicates are automatically filtered.

## Technical details

The crawler runs as an isolated `claw` system user on a Hetzner CPX31 VM 
(Ubuntu 24), separate from the main development environment. It has no access 
to project repositories, SSH keys, or GitHub credentials.

### Agent setup

- [OpenClaw](https://openclaw.ai) runs as a persistent gateway in a tmux 
  session, controlled via Telegram
- Model: `qwen2.5:3b` via [Ollama](https://ollama.com) (local inference, 
  no external API calls for the agent itself)
- Shell tool is disabled — the agent cannot execute arbitrary commands, 
  which limits the blast radius of any prompt injection from scraped content

### Crawler

The crawler is a standalone Python script registered as a custom OpenClaw 
skill at `/home/claw/workspace/titiler-crawler/crawler.py`. It runs inside 
a Python venv and has three source modules:

**GitHub Search API** (`source: github`)
- Unauthenticated requests to `api.github.com/search/repositories`
- Query: `titiler`, sorted by last updated
- Paginates up to 3 pages (90 results max per run)
- Rate limit: 10 req/min unauthenticated

**PyPI dependents** (`source: pypi`)
- Fetches package metadata from `pypi.org/pypi/{package}/json`
- Scrapes `github.com/{org}/{repo}/network/dependents` for downstream packages

**Brave Search API** (`source: brave`)
- Three queries per run: `"titiler" site:github.com`, 
  `titiler implementation example`, `titiler deployment`
- 10 results per query, ~30 results per run
- Free tier: $5/month credit (~1,000 queries/month)

### Dedup and output

Before appending, the script loads all existing URLs from the CSV into a 
set and skips any result already present. Results are appended to 
`/srv/search-output/data/{tool}/implementations.csv` on the shared volume.

### Sync

A separate cron job runs as the `anthony` user at 6:05am UTC, 5 minutes 
after the crawler. It runs `git add data/ && git commit && git push` to 
sync new results to this repo. If there are no new results, it exits 
without committing.

### Schedule

| Time (UTC) | User | Action |
|---|---|---|
| 6:00am | `claw` | Run crawler, append to CSV |
| 6:05am | `anthony` | Commit and push to GitHub |

## CSV columns

| Column | Description |
|---|---|
| `url` | Link to the implementation or resource |
| `repo_name` | Repository or page name |
| `source` | Where it was found (`github`, `pypi`, `brave`) |
| `discovered_at` | UTC timestamp when first found |
| `description` | Short description |
| `stars` | GitHub stars (if applicable) |
| `last_updated` | Last activity date (if applicable) |

## Contributing

Have a tool you want to monitor? Open an issue or PR with:
- Tool name
- Search queries to use
- Target sources (GitHub / PyPI / web)
