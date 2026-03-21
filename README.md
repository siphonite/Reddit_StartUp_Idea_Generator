# reddit-research-engine

## Overview

A Rust CLI engine that crawls Reddit posts and subreddits, extracts discussions (post body + top comments), and generates structured micro-SaaS startup ideas using Google Gemini. Results are printed to stdout in multiple formats and optionally appended to a Google Sheet for persistent tracking.

**Key capabilities:**

- Analyze individual posts, entire subreddits, or multiple subreddits in a single run
- Structured output: each idea includes product name, target user, core problem, MVP features, monetization model, and feasibility assessment
- Optional Google Sheets integration for building a timestamped research database
- Defensive JSON parsing with automatic model fallback across multiple Gemini endpoints

---

## Features

- **Single post analysis** — deep-dive into one Reddit discussion
- **Subreddit crawling** — fetch and analyze hot posts from any subreddit
- **Multi-subreddit scanning** — process multiple subreddits sequentially in one run
- **Batch processing** — analyze a list of URLs from a file
- **Structured idea extraction** — Gemini returns JSON-structured ideas, not freeform text
- **Google Sheets export** — append rows automatically, never overwrite existing data
- **Multiple output formats** — text, JSON, or markdown
- **Global idea cap** — `--max-ideas` to limit total ideas across a multi-subreddit scan
- **Graceful degradation** — Sheets export is optional and non-fatal; missing config is silently skipped

---

## Installation

### Prerequisites

- **Rust** (stable) — [install via rustup](https://rustup.rs/)
- **OpenSSL Dev Libraries** (Linux only) — `sudo apt install libssl-dev pkg-config`
- **Gemini API key** — [get one from Google AI Studio](https://aistudio.google.com/apikey)
- **Google Cloud project** *(optional)* — required only for Sheets export

### Clone & Build

```bash
git clone https://github.com/<your-username>/reddit-research-engine.git
cd reddit-research-engine
cargo build --release
```

The binary will be at `target/release/reddit-research-engine`.

### Environment Configuration

Create a `.env` file in the project root:

```env
GEMINI_API_KEY=your_gemini_api_key

# Optional — required only for Google Sheets export
GOOGLE_SHEET_ID=your_google_sheet_id
GOOGLE_APPLICATION_CREDENTIALS=google-credentials.json
```

| Variable | Required | Description |
|----------|----------|-------------|
| `GEMINI_API_KEY` | **Yes** | API key for Google Gemini |
| `GOOGLE_SHEET_ID` | No | The ID from your Google Sheet URL (`/d/SHEET_ID/edit`) |
| `GOOGLE_APPLICATION_CREDENTIALS` | No | Path to service account JSON credentials file |

---

## CLI Usage

### 1. Analyze a Single Post

```bash
cargo run -- analyze "https://www.reddit.com/r/startups/comments/..." --comments 10
```

| Option | Default | Description |
|--------|---------|-------------|
| `--comments <N>` | `10` | Number of top comments to include |
| `--format <FMT>` | `text` | Output format: `text`, `json`, `markdown` |
| `--save <FILE>` | — | Save output to a file |

### 2. Batch Mode

Process multiple URLs from a file (one URL per line, `#` lines are skipped):

```bash
cargo run -- batch urls.txt --format json --save results.json
```

### 3. Subreddit Mode

Crawl hot posts from a single subreddit:

```bash
cargo run -- subreddit AppDevelopers --limit 5 --comments 10
```

| Option | Default | Description |
|--------|---------|-------------|
| `--limit <N>` | `5` | Number of hot posts to fetch |
| `--comments <N>` | `10` | Number of top comments per post |
| `--format <FMT>` | `text` | Output format |
| `--save <FILE>` | — | Save output to a file |

### 4. Multi-Subreddit Mode

Scan multiple subreddits sequentially in one run:

```bash
cargo run -- multi startups,AppDevelopers,SideProject --limit 5 --comments 10 --max-ideas 50
```

| Option | Default | Description |
|--------|---------|-------------|
| `--limit <N>` | `5` | Posts to fetch per subreddit |
| `--comments <N>` | `10` | Comments per post |
| `--max-ideas <N>` | — | Global cap; stops processing when reached |
| `--format <FMT>` | `text` | Output format |
| `--save <FILE>` | — | Save output to a file |

At completion, a summary is printed:

```
Scan complete.

Subreddits processed: 3
Posts analyzed: 15
Ideas generated: 45
Posts failed: 0
```

---

## Output Formats

| Format | Flag | Description |
|--------|------|-------------|
| **Text** | `--format text` | Human-readable with section dividers (default) |
| **JSON** | `--format json` | Structured JSON array of all results |
| **Markdown** | `--format markdown` | Formatted markdown with headers and lists |

---

## Google Sheets Export (Optional)

When configured, every generated idea is automatically appended as a row to your Google Sheet. Rows are **never overwritten** — each run appends below existing data.

### Setup

1. Go to the [Google Cloud Console](https://console.cloud.google.com/)
2. Create a project and enable the **Google Sheets API** and **Google Drive API**
3. Create a **Service Account** and download the JSON key file
4. Save it as `google-credentials.json` in the project root
5. Open your Google Sheet and **share it** (Editor access) with the service account email from the JSON file (`client_email` field)
6. Copy the Sheet ID from the URL and add it to `.env`

### Sheet Columns (A–J)

| A | B | C | D | E | F | G | H | I | J |
|---|---|---|---|---|---|---|---|---|---|
| Date (UTC) | Subreddit | Post URL | Post Title | Product Name | Target User | Core Problem | MVP Features | Monetization | Feasibility |

If Sheets is not configured, the CLI operates normally without it. If a Sheets write fails, a warning is printed and processing continues.

---

## Architecture Overview

```
src/
├── main.rs          # Entry point, command routing, orchestration
├── cli.rs           # clap-based CLI definitions
├── config.rs        # Environment configuration loader
├── errors.rs        # AppError enum
├── models.rs        # RedditPost, Idea, AnalysisResult, JSON parsing
├── output.rs        # Text / JSON / Markdown formatters
├── services/
│   ├── reddit.rs    # Reddit post + subreddit fetcher
│   └── gemini.rs    # Gemini API client with model fallback
├── export/
│   └── sheets.rs    # Google Sheets batch append
└── utils/
    └── validation.rs # URL validation
```

**Processing model:** All operations are sequential. No concurrency, no thread pools. Each post is fetched, analyzed, and exported before moving to the next.

**Gemini fallback:** The engine cycles through multiple Gemini models (`gemini-2.5-flash`, `gemini-flash-latest`, `gemini-2.5-flash-lite`, `gemini-2.0-flash`) on timeout or rate-limit errors.

---

## Error Handling

| Scenario | Behavior |
|----------|----------|
| Missing `GEMINI_API_KEY` | Fails immediately at startup |
| Missing Sheets config | Sheets export silently skipped |
| Sheets write failure | Warning printed, processing continues |
| Gemini timeout/rate-limit | Falls back to next model automatically |
| Unparseable Gemini JSON | Falls back to raw text display |
| Invalid Reddit URL | Returns clear validation error |

---

## Example Workflow

```bash
# Scan 3 subreddits, 5 posts each, cap at 30 ideas, save as markdown
cargo run -- multi startups,SideProject,micro_saas \
  --limit 5 --comments 8 --max-ideas 30 \
  --format markdown --save research.md
```

This will:

1. Fetch 5 hot posts from each subreddit (up to 15 posts total)
2. Extract post body + top 8 comments for each
3. Generate 3 structured micro-SaaS ideas per post via Gemini
4. Stop early if 30 total ideas are reached
5. Append each batch of ideas to Google Sheet (if configured)
6. Print all results as markdown to stdout
7. Save the full output to `research.md`
8. Print a summary with counts

---

## License

Licensed under the [Apache License, Version 2.0](LICENSE).
