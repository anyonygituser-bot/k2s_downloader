glm-5.1

1.  Analyze the Request:
    
    *   Goal: Create a GitHub `README.md` for the project.
    *   Audience: Casual users (how to use), programmers (how to dismantle/understand), future developers (how to improve).
    *   Tone/Style: Open source, permissive (use, share, dismantle, add/subtract), collaborative (share improvements).
    *   Specific Details to Include:
        *   Date: June 2026.
        *   Disclaimer: It works as of upload date. If K2S changes API, proxy structures, or requirements break, the program may break.
        *   Maintenance stance: Absolutely no intention of updating or maintaining it.
        *   Content focus: User manual (how to use), programmer guide (how it works/how to take it apart), base for improvement.
    *   Not a deep dive into K2S internals (that was the previous doc), but rather a practical "how-to" and "what-now".
2.  Structure the README:
    
    *   Title & Badges/Info: Project name, brief description, license/style (Open Source, Unlicense/MIT vibe).
    *   Status & Disclaimer: June 2026, "works today, might break tomorrow", no maintenance promised.
    *   Philosophy: Take it, break it, improve it, share it.
    *   For the Casual User (How to Use):
        *   Prerequisites (Python, run command).
        *   Step-by-step usage (Get file ID, run main, solve captcha, watch it download).
        *   Resuming downloads.
    *   For the Programmer (How It Works & How to Dismantle):
        *   Architecture overview (brief).
        *   Module breakdown (what each file does).
        *   How to take it apart (what to rip out, what to replace).
    *   For the Improver (Making It Better):
        *   Known limitations / low-hanging fruit for improvement.
        *   Ideas to build on top of it.
    *   Final Note: The license/manifesto.
3.  Drafting Content - Section by Section:
    
    *   _Header:_ `# K2S Downloader` - A parallel, proxy-swarming downloader for K2S free tier.
    *   _Disclaimer:_ "Built in June 2026. It works exactly as described today. If K2S changes their API, captcha flow, or proxy ecosystems die, it will break. There is absolutely no intention of updating or maintaining this. It is a snapshot in time."
    *   _License/Philosophy:_ "Do whatever you want with it. Use it, share it, dismantle it, add to it, subtract from it. The only ask: if you make it better, share it with others too."
    *   _Casual User Guide:_
        *   Need: Python 3.8+, a terminal.
        *   Step 1: Get proxies (`python tests/fresh_proxies.py`).
        *   Step 2: Run it (`python main.py`).
        *   Step 3: Paste the File ID (from the URL, e.g., `https://k2s.cc/file/4d0c302353422` -> `4d0c302353422`).
        *   Step 4: Solve the captcha that pops up.
        *   Step 5: Watch the dashboard. Go get coffee.
        *   _Note on Resuming:_ Just run it again with the same File ID. It picks up where it left off.
    *   _Programmer Guide:_
        *   It's built in layers. You can rip any layer out.
        *   `proxy_engine.py`: The blood supply. Scrapes, validates, pools. Swap this out if you have a paid proxy list API.
        *   `minter.py`: The key maker. Solves captchas and spams the K2S API for tokens. If K2S changes their captcha, fix it here.
        *   `state_manager.py`: The memory. Tracks chunks and tokens. Swap for a database if you want a web UI.
        *   `download_engine.py`: The muscle. Threads, speed checks, file I/O. Tweak thread counts or speed thresholds here.
        *   `main.py`: The glue. Put it all together.
    *   _Improver Guide:_
        *   "Here is what I didn't do, but you could:"
        *   OCR for captchas (automate the mint completely).
        *   Auto-scaling threads based on proxy pool size.
        *   Web UI instead of terminal dashboard.
        *   Multi-file queue.
        *   Smarter proxy scraping (API-based instead of HTML parsing).
4.  Review and Refine:
    
    *   Check against constraints: June 2026? Yes. No maintenance disclaimer? Yes. Open source manifesto? Yes. User/Programmer/Improver sections? Yes.
    *   Tone check: Direct, no-nonsense, practical.
5.  Final Polish: Add Markdown formatting for readability. (Output the response).
    

# K2S Downloader

A parallel, proxy-swarming downloader that bypasses Keep2Share (K2S) free-tier speed limits. Turns a 9-hour single-connection crawl into a 20-minute parallel download.

**Status: June 2026** — This program works exactly as described today. If K2S changes their API, captcha flow, token structure, or if the free proxy ecosystem shifts, parts of it will break. There is absolutely no intention of updating or maintaining this. It is a snapshot in time. Use it, learn from it, or break it apart.

* * *

### The Rule

This is open source. No limits. Use it, share it, dismantle it, add to it, subtract from it. Use it for parts to build something completely different. The only ask: **if you make it better, share it with others too.**

* * *

## For the Casual User: How to Use

### What You Need

*   A computer with **Python 3.8+** installed.
*   A terminal / command prompt.
*   An internet connection.
*   A K2S file link (e.g., `https://k2s.cc/file/4d0c302353422`).

### Step 1: Get Proxies

The downloader needs proxy servers to multiply your speed. You need to gather them first.

Open your terminal, navigate to the project folder, and run:

text

```
python tests/fresh_proxies.py
```

Wait a few minutes. It will scrape the web for free proxies and test them. When it finishes, you'll have a validated list ready to go.

### Step 2: Start the Downloader

text

```
python main.py
```

### Step 3: Enter the File ID

It will ask for a File ID. Take your K2S link (`https://k2s.cc/file/4d0c302353422`) and paste just the ID part:

text

```
File ID: 4d0c302353422
```

It will fetch the file name and size automatically.

### Step 4: Solve the Captcha

If you don't have enough cached tokens, the minter will start. An image will pop up on your screen. Type the text you see in the terminal and press Enter. You might need to do this 2-3 times. This generates the download keys.

### Step 5: Watch It Download

The dashboard will appear:

text

```
Overall: 48.66% █████████░░░░░░░░░░░ 974848KB / 2003544KB [167/392]  |  Speed: 698.5KB/s  |  Elapsed: 00:21:37  |  ETA: 00:24:32
Tokens: 1253  |  Bindings: 23  |  Active: 24  |  Waiting: 1  |  Disconnected: 5
Proxies: Validated: 312  |  Cold: 8420  |  Prison: 87
```

Go get coffee. It will finish on its own. The final file appears in the `data/` folder.

### If It Gets Interrupted

Press `Ctrl+C`, close the terminal, lose power—doesn't matter. Just run `python main.py` again with the same File ID. It picks up exactly where it left off.

* * *

## For the Programmer: How It Works & How to Dismantle It

The system is built in layers. Every layer is independent. You can rip any piece out, replace it, or use it standalone.

### The Layers

text

```
main.py          → The glue. Orchestrates everything. Start here to trace the flow.
  │
  ├── proxy_engine.py  → The blood supply. Finds and manages proxies.
  ├── minter.py        → The key maker. Talks to K2S API, handles captchas, mints tokens.
  ├── state_manager.py → The memory. Tracks chunks, progress, tokens. Saves to disk.
  └── download_engine.py → The muscle. Worker threads, file I/O, speed control.
```

### How to Take It Apart

**Want to use your own proxy source?**  
Throw away `proxy_engine.py`. Replace it with anything that provides a `get_proxy()` method returning `{"id": "...", "url": "http://ip:port"}` and a `report_drop(id)` method. The rest won't notice.

**Want to automate the captcha?**  
Open `minter.py`. Replace the `_request_captcha()` and `_submit_captcha()` methods with an OCR solver or an API call. Everything downstream expects a list of token URLs. As long as you feed it those, it works.

**Want to add a web UI?**  
`main.py` and the dashboard loop are cleanly separated. Replace the `format_dashboard()` function and the `while dl_thread.is_alive()` loop with a Flask/FastAPI server that reads `dl.get_full_stats()`. The engine doesn't care how you display the data.

**Want to download from a different site?**  
Keep `proxy_engine.py`, `state_manager.py`, and `download_engine.py`. Replace `minter.py` with whatever generates download URLs for your target site. The chunking, proxy rotation, and resume logic are site-agnostic.

**Want to use a database instead of JSON files?**  
Replace `state_manager.py`. Implement the same methods (`get_next_chunk`, `complete_chunk`, `checkout_token`, etc.) but back it with SQLite, Redis, or whatever you want. The download engine only calls these methods.

### Core Concepts to Understand

1.  **Chunk Assignment (`chunk_taken_count`):** Every chunk tracks how many times it was assigned. Workers always pick the least-attempted chunk. Ties broken by highest progress. This guarantees unique assignment first, fair racing later.
    
2.  **Permanent IP-Token Binding (`ip_to_token`):** When a proxy IP pairs with a token, that binding is saved forever. If the proxy dies, the binding stays. The token is "quarantined"—never returned to the pool to cause 409 errors. If the IP comes back later, the token is reused instantly.
    
3.  **Wave-Based Startup:** Doesn't launch 50 threads at once. Starts 5, checks if 60% are healthy, then launches 5 more. Grows to match proxy supply.
    
4.  **First-Byte Grace:** Speed checks don't start based on wall clock. They start after the first byte of data arrives. Prevents killing proxies that are slow to connect but fast to download.
    
5.  **Micro-Resumption:** Workers save progress every 512KB. On disconnect, the next worker resumes from the exact byte, not from the beginning of the chunk.
    
6.  **Replace Only What's Broken:** Chunk finishes → keep proxy + token. Proxy dies → keep chunk + binding. Token dies (409) → keep chunk + proxy. Never throw away something that still works.
    

* * *

## For the Improver: Making It Better

Here is what I didn't build. Low-hanging fruit you could pick:

### Automation

*   **OCR the captcha.** The minter already isolates the image. Plug in a vision model or a solving service and remove the human entirely.
*   **Auto-retry minting.** If tokens run out mid-download, pause workers, mint more, resume.

### Smarter Downloading

*   **Auto-scale threads.** Read the proxy pool size. If validated = 10, run 10 threads. If validated = 100, run 50 threads. No hardcoded thread count.
*   **2-IP token binding.** K2S allows 2 IPs per token. We use 1 for simplicity. Using 2 would halve token consumption.
*   **Priority chunks.** Download chunks sequentially so the file can be previewed/played while still downloading.

### Better Proxies

*   **Paid proxy API integration.** Replace the scraper with a reliable proxy provider. Speed and stability would jump dramatically.
*   **Proxy health scoring.** Track how many bytes each proxy delivered. Prefer proxies with a proven track record.

### Better UI

*   **Rich terminal UI.** The `rich` package is already in requirements but unused. Build a proper TUI with colored output, progress bars, and live charts.
*   **Web dashboard.** Flask + WebSocket pushing stats to a browser. Monitor downloads from your phone.

### Architecture

*   **Multi-file queue.** Feed it a list of file IDs. Download them one after another, or in parallel.
*   **Distributed workers.** Run the proxy engine on one machine, download workers on multiple machines, all writing to a shared NAS.

* * *

## Project Structure

text

```
K2S-final/
├── main.py                 # Run this to start everything
├── requirements.txt        # pip dependencies
├── README.md               # You are here
├── .gitignore
│
├── src/                    # Core modules
│   ├── proxy_engine.py     # Proxy scraping, validation, pool management
│   ├── minter.py           # Token minting via K2S API
│   ├── state_manager.py    # Chunk state, token pool, persistence
│   └── download_engine.py  # Worker threads, downloading, file I/O
│
├── tests/                  # Standalone tools
│   ├── fresh_proxies.py    # Scrape and validate proxies
│   ├── test_minter.py      # Test token minting
│   └── test_download.py    # Test full download with dashboard
│
└── data/                   # Auto-created at runtime
    ├── .requirements.json  # Package lock file
    ├── proxies.txt         # Raw scraped proxies
    ├── connected.txt       # Validated proxies
    ├── cooldowns.json      # Minter IP cooldowns
    ├── tokens.json         # Cached download tokens
    ├── *.state.json        # Download state (for resume)
    └── *.part              # Download in progress
```

* * *

_Built June 2026. Works today. Tomorrow is your problem._

## Message from glm-5.1