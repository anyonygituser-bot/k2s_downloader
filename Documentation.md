glm-5.1
1.	Understand the Goal: The user wants a comprehensive, detailed project documentation file. It needs to cover everything: what K2S is, its limitations, the project structure, what each file/module/method does, how components interact, the bypass strategies, test files, requirements, test results/inferences, and data management. It must be structured, educational, and leave nothing out.
2.	Target Audience & Tone: Educational, detailed, structured. Someone reading this should understand the entire ecosystem, the reasoning behind every design choice, and exactly how the code maps to the problem domain.
3.	Outline the Documentation:
o	Title & Overview: Project name, purpose, high-level summary.
o	1. The Problem Domain: Keep2Share (K2S) 
	What is K2S?
	How does it limit free users? (Speed, Captcha, IP binding/Token restrictions, Concurrent connections).
o	2. The Solution: Architecture & Bypass Strategies 
	High-level architecture diagram (text-based).
	Bypass Strategy 1: Distributed Proxy Network (IP diversity).
	Bypass Strategy 2: Token Minting (Automated captcha solving + API exploitation).
	Bypass Strategy 3: Chunked Parallel Downloading (Swarm downloading).
	Bypass Strategy 4: Resilience & Resumption (Micro-resumption, state management).
o	3. Project Structure & File Manifest 
	Directory tree.
	Description of top-level files (main.py, requirements.txt).
	src/ directory breakdown.
	tests/ directory breakdown.
	data/ directory breakdown.
o	4. Deep Dive: Core Modules (src/) 
	proxy_engine.py (Purpose, Classes, Methods, Data flow, Endpoints used).
	minter.py (Purpose, API flow, Captcha handling, Token caching).
	state_manager.py (Purpose, State machine, Chunk logic, Token pool, Persistence).
	download_engine.py (Purpose, Worker lifecycle, Wave startup, 1-to-1 binding, Speed/Grace logic, Memory management).
o	5. The Dashboard & User Interface (main.py) 
	Setup & Requirement checking.
	Execution flow.
	Dashboard metrics.
o	6. Testing & Inference (tests/) 
	fresh_proxies.py (Purpose, inference from testing).
	test_minter.py (Purpose, inference from testing).
	test_download.py (Purpose, inference from testing - e.g., chunk racing, 112% bug, proxy affinity).
o	7. Data Management & Persistence 
	How data is stored (data/ folder).
	File formats (JSON, TXT).
	Atomic writes & Crash safety.
o	8. Requirements & Dependencies 
	requests, rich, Pillow, beautifulsoup4.
	How .requirements.json lock file works.
4.	Drafting Content - Section by Section (Iterative refinement for detail):
o	K2S Section: Explain the 30KB/s per connection limit. Explain the captcha required to get a free_download_key. Explain the getUrl API that yields a direct CDN link, but that CDN link is tied to the IP that requested it (or max 2 IPs). Explain concurrent connection limits.
o	Architecture Section: Connect the K2S limits directly to our solutions. E.g., Limit: 30KB/s per IP -> Solution: 30 proxies = 900KB/s. Limit: Captcha -> Solution: Minter opens image, prompts user, mints 500 tokens in one go.
o	Module Details: 
	proxy_engine.py: Mention the 3 pools (connected, cold, prison), the validation logic, the semaphore, proxy affinity (workers hold proxies).
	minter.py: Mention K2S API endpoints (/getFreeDownloadCaptcha, /getFreeDownloadKey, /getUrl). Mention the cooldown file.
	state_manager.py: Detail chunk_taken_count (lowest count, highest progress). Detail micro-resumption (512KB updates). Detail atomic saves.
	download_engine.py: Detail the wave-based startup (5 workers, 60% health threshold). Detail 1-to-1 IP-Token binding (permanent marriage, no TTL, quarantine dead IPs). Detail first-byte grace period. Detail the already_done race condition fix. Detail the finally: r.close() memory leak fix.
o	Testing Section: Must include the specific bugs found and fixed during our conversation: 
	Test 1: 112% progress bug (summing absolute byte positions instead of chunk sizes).
	Test 2: All workers downloading chunk 0 (missing assigned_chunks set).
	Test 3: Last-minute racing (4 workers on chunk 203, introduced chunk_taken_count).
	Test 4: Proxy death dropping tokens (switched to permanent binding quarantine).
	Test 5: Memory leak (unclosed HTTP streaming responses).
5.	Review and Refine:
o	Check against user prompt: "absolutely everything in detail", "structured and educational", "how data and files are made and saved".
o	Ensure markdown formatting is clean (headers, bolding, code blocks, tables).
o	Add a table of contents for easy navigation.
6.	Final Polish: Make the tone authoritative but accessible. Ensure all variable names, API endpoints, and file names are accurate to the code we wrote. (Self-correction: Ensure I explicitly mention the 2-IP token limit K2S has, and how we deliberately only use 1-to-1 for simplicity, treating the 2nd slot as accidental resilience).
7.	Generate Output. (Proceeding to generate the massive markdown document).
K2S Downloader: Complete Project Documentation
Table of Contents
1.	Introduction & Overview
2.	The Problem Domain: Keep2Share (K2S)
3.	Bypass Strategies & Architecture
4.	Project Structure & File Manifest
5.	Deep Dive: Core Modules (src/)
6.	The Orchestrator: main.py
7.	Testing & Inference (tests/)
8.	Data Management & Persistence
9.	Requirements & Dependencies
________________________________________
1. Introduction & Overview
K2S Downloader is a high-performance, parallel file downloader designed to bypass the free-tier restrictions of the Keep2Share (K2S) cloud storage service. It transforms a heavily throttled, single-connection download process into a multi-threaded, proxy-distributed swarm that achieves speeds comparable to premium accounts.
What It Does
•	Scrapes and validates thousands of free proxy servers.
•	Automates token minting by solving K2S captchas and generating download URLs.
•	Parallelizes downloads by splitting files into chunks and downloading them simultaneously through different proxies.
•	Manages state atomically, allowing seamless resume after crashes, disconnects, or user cancellation.
•	Optimizes resources using wave-based thread scaling, permanent IP-Token bindings, and intelligent chunk assignment.
Design Philosophy
1.	Replace only what's broken. If a proxy dies, the chunk and token are preserved. If a token dies, the proxy and chunk are preserved.
2.	Quarantine, don't kill. Tokens bound to dead IPs are locked away, not returned to the pool where they cause 409 errors.
3.	Measure, don't guess. Speed checks only start after the first byte of data arrives, preventing false kills on slow-to-connect proxies.
4.	Unique first, race later. Workers always claim un-attempted chunks before joining races on partially downloaded ones.
________________________________________
2. The Problem Domain: Keep2Share (K2S)
What is K2S?
Keep2Share (k2s.cc) is a cloud storage and file hosting platform. Users upload files and share links. Free users can download these files, but K2S employs aggressive monetization strategies to push users toward premium subscriptions.
How K2S Limits Free Users
Restriction	Mechanism	Impact
Speed Throttle	K2S CDN limits single-connection downloads to ~30 KB/s	A 1GB file takes ~9 hours
Captcha Gate	Requires solving a visual captcha to generate a download key	Blocks automated downloads
Token Expiry	Free download keys expire after 24 hours	Must re-solve captchas daily
IP Binding	The generated download URL (token) is bound to the IP that requested it (up to 2 IPs max)	Cannot share tokens across many IPs
Concurrency Limit	A free account/IP has limited simultaneous connections	Multiple threads from one IP get blocked
Cooldown Timers	After generating a key, a wait timer (1-60 seconds) is enforced	Slows down token generation
The K2S API Flow (Free Download)
text
1. POST /api/v2/getFreeDownloadCaptcha
   → Returns: {"status": "success", "captcha_url": "https://...", "challenge": "..."}

2. [USER SOLVES CAPTCHA] → Returns text like "rkz4Xk"

3. POST /api/v2/getFreeDownloadKey
   → Payload: {"file_id": "...", "captcha_challenge": "...", "captcha_answer": "..."}
   → Returns: {"status": "success", "free_download_key": "..."}

4. POST /api/v2/getUrl
   → Payload: {"file_id": "...", "free_download_key": "..."}
   → Returns: {"status": "success", "url": "https://cmb-speed.k2s.cc/..."}
   → This URL is the "token". It is bound to the requesting IP.
________________________________________
3. Bypass Strategies & Architecture
Strategy 1: Distributed Proxy Network
Problem: 30 KB/s per connection.
Solution: Use 30+ proxies. Each proxy gets its own 30 KB/s connection.
Result: 30 proxies × 30 KB/s = ~900 KB/s.
Strategy 2: Automated Token Minting
Problem: Captchas block automation.
Solution: The minter fetches the captcha image, opens it on the user's machine, and prompts for the answer. One captcha solve yields a free_download_key. That key is then used with multiple proxies to mint 100-500 download URLs (tokens) in a burst.
Result: 2-3 minutes of user interaction generates enough tokens for a full download.
Strategy 3: Chunked Parallel Downloading
Problem: Single connection is slow and fragile.
Solution: Split the file into 5MB chunks. Assign each chunk to a different worker. All workers write to a pre-allocated .part file at their specific byte offset using r+b mode.
Result: Parallel downloading with zero file corruption.
Strategy 4: 1-to-1 IP-Token Binding (Permanent Marriage)
Problem: Tokens used from wrong IPs trigger HTTP 409 Conflict.
Solution: Maintain a dictionary mapping IP → Token. When a worker gets a proxy, it checks the dict first. If the IP was seen before, it reuses the exact same token. If the proxy dies, the binding stays (permanent marriage). If the IP never returns, the token is quarantined—harmlessly locked away rather than returned to the pool to cause 409s.
Result: Near-zero 409 errors. K2S allows 2 IPs per token, but we use only 1 for simplicity. The unused 2nd slot acts as accidental resilience.
Strategy 5: Intelligent Chunk Assignment
Problem: Multiple workers waste effort downloading the same chunk while others are untouched.
Solution: Every chunk tracks a taken_count. Workers always pick the chunk with the lowest taken_count. Among ties, they pick the chunk with the highest progress (partial downloads get finished first).
Result: 100% unique assignment when possible. Graceful, fair racing when necessary.
Strategy 6: Wave-Based Startup & Proxy Affinity
Problem: 50 workers launching simultaneously drains the proxy pool instantly.
Solution: Start 5 workers. Check health every 15 seconds. If ≥60% are actively downloading, launch 5 more. Workers hold their proxies across multiple chunks (proxy affinity). Only discard a proxy on failure.
Result: Organic growth that matches proxy supply.
Architecture Diagram
text
┌─────────────────────────────────────────────────────────────────┐
│                          main.py                                │
│  (Orchestrator: Requirements → File Info → Mint → Download)     │
└──────────┬───────────────┬──────────────────┬───────────────────┘
           │               │                  │
           ▼               ▼                  ▼
┌─────────────────┐ ┌──────────────┐ ┌──────────────────────┐
│  proxy_engine   │ │   minter     │ │   state_manager      │
│────────---------│ │--------------│ │----------------------│
│ Scraper         │ │ Captcha API  │ │ Chunk tracking       │
│ Validator       │ │ Key API      │ │ Token pool           │
│ 3-Pool System   │ │ URL API      │ │ Progress persistence │
│ Semaphore       │ │ Cooldowns    │ │ Atomic saves         │
└────────┬────────┘ └──────┬───────┘ └──────────┬───────────┘
         │                 │                     │
         │                 ▼                     │
         │          ┌──────────────┐             │
         │          │ tokens.json  │             │
         │          └──────────────┘             │
         │                                       │
         ▼                                       ▼
┌──────────────────────────────────────────────────────────────┐
│                     download_engine                          │
│──────────────────────────────────────────────────────────────│
│  Workers (Wave-based)    │  IP-Token Binding (Permanent)     │
│  Chunk Downloading       │  Speed/Stale Monitoring           │
│  Micro-Resumption (512KB)│  First-Byte Grace Period          │
│  .part File Writing      │  Response Cleanup (No Leaks)      │
└──────────────────────────────────────────────────────────────┘
________________________________________
4. Project Structure & File Manifest
text
K2S-final/
├── main.py                    # Entry point. Orchestrates all steps.
├── README.md                  # This documentation file.
├── requirements.txt           # pip dependencies (reference only)
│
├── src/                       # Core library modules
│   ├── proxy_engine.py        # Proxy scraping, validation, pool management
│   ├── minter.py              # Token minting via K2S API
│   ├── state_manager.py       # Chunk state, token pool, persistence
│   └── download_engine.py     # Worker threads, downloading logic
│
├── tests/                     # Test scripts for each module
│   ├── fresh_proxies.py       # Scrape and validate proxy list
│   ├── test_minter.py         # Test token minting flow
│   └── test_download.py       # Test full download with dashboard
│
└── data/                      # Runtime data (auto-created)
    ├── .requirements.json     # Lock file for installed packages
    ├── proxies.txt            # Raw scraped proxies
    ├── connected.txt          # Validated working proxies
    ├── cooldowns.json         # Minter IP cooldown timestamps
    ├── tokens.json            # Cached tokens per file_id
    ├── {filename}.state.json  # Download state for resume
    └── {filename}.part        # Pre-allocated download target
________________________________________
5. Deep Dive: Core Modules (src/)
5.1 proxy_engine.py
Purpose: Provides a continuous supply of working HTTP/HTTPS proxies to the download engine.
Key Concepts:
•	Three-Pool System: 
o	_connected: Proxies validated and ready for immediate use.
o	_cold: Proxies scraped but not yet tested.
o	_prison: Proxies that failed validation (cooling off before re-test).
•	Semaphore: Limits concurrent proxy validations to prevent network saturation (default: 225).
•	Proxy Affinity: Workers hold proxies across chunks. Only discarded on failure.
Key Methods:
Method	Purpose
start()	Loads pools from disk, starts background validator threads
stop()	Saves pools to disk, shuts down gracefully
get_proxy(timeout)	Returns {id, url} of a connected proxy. Blocks until available.
release_proxy(proxy_id)	Returns proxy to connected pool (worker kept it across chunks)
report_drop(proxy_id)	Moves proxy to prison (it failed during download)
get_stats()	Returns {validated, cold, prison} counts for dashboard
How Proxies Are Validated:
1.	Proxy tries connecting to http://k2s.cc within 5 seconds.
2.	Checks if response contains "Keep2Share" (rules out fake/captive portals).
3.	Valid → _connected. Invalid → _prison with timestamp.
4.	Prison proxies re-tested after cooldown.
________________________________________
5.2 minter.py
Purpose: Automates the generation of K2S download tokens by interacting with the K2S API and prompting the user for captcha solves.
API Endpoints Used:
Endpoint	Purpose
POST /api/v2/getFreeDownloadCaptcha	Gets captcha image URL and challenge token
POST /api/v2/getFreeDownloadKey	Submits captcha answer, gets download key
POST /api/v2/getUrl	Uses key + proxy to generate CDN download URL (the "token")
Key Methods:
Method	Purpose
mint(file_id, target_tokens)	Main flow: get captcha → solve → burst mint tokens
_request_captcha()	Calls K2S API, saves image, opens it for user
_submit_captcha(challenge, answer)	Submits answer, returns free_download_key
_try_get_url(proxy, file_id, key)	Uses proxy + key to mint one CDN URL
_generate_tokens(proxy, file_id, key)	Spawns 10 threads to burst-mint URLs from one proxy
_save_cache()	Saves tokens to data/tokens.json with expiry timestamps
Captcha Strategy:
1.	Fetch captcha from home IP (not proxy) for reliability.
2.	Open image automatically using OS default viewer (os.startfile / open / xdg-open).
3.	Prompt user in console for answer.
4.	If invalid, allow 1 retry with same challenge.
5.	If still invalid, fetch a new captcha (max 3 total).
Token Burst Minting:
1.	One successful captcha → one free_download_key.
2.	That key is used with a validated proxy to spawn 10 parallel request threads.
3.	Each thread calls getUrl → gets a CDN URL (token).
4.	Tokens are cached with 24-hour expiry.
Cooldown Management:
•	K2S enforces cooldowns per IP after key generation.
•	Tracked in data/cooldowns.json as {ip: timestamp}.
•	Minter skips proxies on cooldown.
________________________________________
5.3 state_manager.py
Purpose: Single source of truth for chunk progress, token pool, and download state. Thread-safe. Persisted to disk atomically.
Chunk Lifecycle:
text
PENDING → ACTIVE (assigned to worker) → COMPLETED
                ↓ (worker fails)
           ACTIVE (stays, progress saved)
                ↓ (new worker picks up)
           ACTIVE → COMPLETED
Key Data Structures:
Structure	Type	Purpose
completed_chunks	set	Chunks fully downloaded and verified
active_chunks	dict {id: bytes}	Chunks currently being downloaded with progress
assigned_chunks	set	Chunks currently checked out by a worker
chunk_taken_count	dict {id: int}	How many times each chunk has been assigned
available_tokens	list	Tokens ready for checkout
checked_out_tokens	list	Tokens currently in use by workers
Key Methods:
Method	Purpose
init_download(...)	Creates fresh state for a new file
load_state(filename)	Loads state from disk for resume
save_state()	Atomic write via .tmp + os.replace()
get_next_chunk()	Returns chunk with lowest taken_count, then highest progress
update_chunk_progress(id, bytes)	Micro-resumption: saves progress every 512KB
complete_chunk(id)	Moves chunk to completed set
release_chunk(id)	Un-assigns chunk so another worker can pick it up
checkout_token()	Pops token from available pool
return_token(url)	Returns token to available pool
mark_token_dead(url)	Removes token from all pools
get_progress_bytes()	Returns total bytes on disk (completed size + active progress)
Chunk Assignment Algorithm (get_next_chunk):
1.	Build list of all chunks not in completed_chunks.
2.	Find the minimum taken_count among candidates.
3.	Filter to only chunks with that minimum count.
4.	Among those, pick the one with the highest progress (prefer partial downloads).
5.	Increment taken_count, add to assigned_chunks, return chunk ID.
Micro-Resumption:
•	Every 512KB, the worker calls update_chunk_progress().
•	On crash/restart, get_resume_byte() returns the last saved progress.
•	The worker sends Range: bytes={resume}-{end} to download only the missing portion.
•	Result: Zero wasted bandwidth.
Persistence:
•	Saved every 5 seconds by the download engine's monitor thread.
•	Uses atomic write: write to .tmp, then os.replace() (crash-safe on all OS).
•	On load, checked_out_tokens are returned to available_tokens (workers are gone after crash).
•	assigned_chunks is cleared on load (chunks become available again).
________________________________________
5.4 download_engine.py
Purpose: Manages worker threads that download chunks, handles IP-Token bindings, monitors speed, and finalizes the file.
Constants:
Constant	Value	Purpose
MIN_SPEED_KBPS	10	Minimum acceptable download speed
SPEED_CHECK_INTERVAL	5	Seconds between speed checks
STALE_DATA_TIMEOUT	10	Kill if no data received for 10s
PROGRESS_UPDATE_BYTES	524288	Save progress every 512KB
RECOVERY_COOLDOWN	5	Seconds to wait after proxy failure
WAVE_SIZE	5	Workers launched per wave
WAVE_HEALTH_THRESHOLD	0.6	60% of workers must be healthy to expand
STATE_SAVE_INTERVAL	5	Seconds between state saves
Key Components:
Worker Lifecycle
text
1. Get proxy from engine
2. Check ip_to_token binding for proxy IP
   → Found? Use that token
   → Not found? Checkout new token, save binding
3. Get chunk from state_manager (only if chunk_id is None)
4. Download chunk:
   → State: "connecting"
   → First byte arrives → State: "downloading"
   → Stream to .part file at correct offset
   → Update progress every 512KB
5. On success:
   → complete_chunk(), state: "finished", chunk_id = None
   → Keep proxy + token
6. On proxy failure:
   → report_drop(), state: "disconnected"
   → Keep chunk_id (will resume)
   → Keep binding (permanent marriage)
   → Get new proxy on next loop
7. On 409:
   → mark_token_dead(), remove binding
   → chunk_id = None, token = None
   → Keep proxy
8. On already_done:
   → release_chunk(), state: "finished", chunk_id = None
   → Keep proxy + token
IP-Token Binding (1-to-1, Permanent)
Python
ip_to_token = {"1.2.3.4": "https://cmb-...", "5.6.7.8": "https://cmb-..."}
•	On proxy assignment: Check dict. If IP exists, reuse token. If not, checkout new token and save.
•	On proxy death: Do nothing. Binding stays. Token stays locked to that IP.
•	On 409: Remove binding. Token is dead.
•	On finalize: Clear entire dict.
•	Why no TTL: A token bound to a dead IP is useless from any other IP. Returning it to the pool just causes 409 errors. Better to quarantine it. K2S allows 2 IPs per token, but we use only 1 for simplicity.
Wave-Based Startup
1.	Launch 5 workers initially.
2.	Every 15 seconds, monitor checks: are ≥60% of launched workers actively downloading?
3.	If yes → launch 5 more. If no → wait.
4.	Repeat until thread_count reached.
5.	Why: Prevents 50 workers from draining the proxy pool simultaneously. Grows organically with supply.
First-Byte Grace Period
•	Speed checks do NOT start based on wall clock time.
•	They start 5 seconds after the first byte of data arrives.
•	Why: A proxy that takes 9 seconds to connect but then downloads at 40KB/s is perfectly good. A fixed 10-second grace period would kill it.
Pre-Allocated .part File
•	On startup, create {filename}.part with exact file size.
•	Uses f.seek(size - 1); f.write(b'\0').
•	Workers open with r+b mode and seek to their chunk's offset.
•	Multiple workers write simultaneously without corruption (each writes to different offset).
Memory Leak Prevention
•	requests.get(..., stream=True) borrows a connection from urllib3's pool.
•	If not closed, the connection leaks.
•	Fix: r = None before try block, finally: if r is not None: r.close() ensures cleanup on every path (success, error, 409, already_done, KeyboardInterrupt).
Finalization
1.	Rename {filename}.part → {filename} (atomic on same filesystem).
2.	Delete state file.
3.	Clear ip_to_token dict.
4.	Print completion message with elapsed time.
________________________________________
6. The Orchestrator: main.py
Purpose: Single entry point. Handles requirements, gathers user input, orchestrates all modules, displays dashboard.
Execution Flow:
text
[1/5] Requirements
  → Check .requirements.json lock file
  → If missing: check each package, install if needed, save lock
  → Packages: requests, rich, Pillow, beautifulsoup4

[2/5] File Info
  → Prompt for File ID
  → Fetch name + size from K2S API

[3/5] Proxy Check
  → Start ProxyEngine
  → Wait 2 seconds for initial validation
  → If 0 proxies → tell user to run fresh_proxies.py, exit

[4/5] Token Check
  → Check tokens.json for cached tokens
  → If < 50 tokens → launch Minter inline
  → User solves captchas, minter generates 500 tokens

[5/5] Download
  → Init StateManager (resume or fresh)
  → Add tokens to state
  → Setup DownloadEngine (pre-allocate .part file)
  → Start download in background thread
  → Main thread: dashboard loop (1 refresh/second)
  → On completion: show final stats + elapsed time
Dashboard Format:
text
Overall: 48.66% █████████░░░░░░░░░░░ 974848KB / 2003544KB [167/392]  |  Speed: 698.5KB/s  |  Elapsed: 00:21:37  |  ETA: 00:24:32
Tokens: 1253  |  Bindings: 23  |  Active: 24  |  Waiting: 1  |  Disconnected: 5
Proxies: Validated: 312  |  Cold: 8420  |  Prison: 87

W00 : Chunk 258 |      1536KB / 5120KB |    31.45KB/s | Downloading (32s) (resumed 512KB)
W01 : Chunk --- |                  --- |     0.00KB/s | Disconnected (2s)
Cancellation Handling:
•	Ctrl+C during minting → stops minter, stops engine, exits cleanly.
•	Ctrl+C during download → calls dl.stop() (saves state), joins threads with timeout, exits cleanly.
•	All wrapped in try/except at top level to catch unexpected errors.
Console Compatibility:
•	Uses os.system('cls' if os.name == 'nt' else 'clear') for cross-platform screen clearing.
•	Uses input("Press Enter to exit.") to prevent console window from closing on error.
•	No IDE required. Run with py main.py.
________________________________________
7. Testing & Inference (tests/)
7.1 fresh_proxies.py
Purpose: Standalone script to scrape proxy lists and validate them before running the main downloader.
What It Does:
1.	Scrapes multiple free proxy list websites.
2.	Deduplicates and cleans proxy URLs.
3.	Validates each proxy against K2S.
4.	Saves working proxies to data/connected.txt.
Inference from Testing:
•	~30,000 raw proxies scraped → ~700 validated on first pass.
•	Proxy health degrades rapidly. A proxy validated 10 minutes ago may be dead now.
•	Conclusion: The proxy engine must continuously re-validate in the background, not rely on a single upfront check.
7.2 test_minter.py
Purpose: Test the token minting pipeline in isolation.
What It Does:
1.	Prompts for file ID.
2.	Loads validated proxies.
3.	Requests captcha from home IP.
4.	Prompts user for answer.
5.	Uses proxy + key to burst-mint tokens.
6.	Saves tokens to cache.
Inference from Testing:
•	Invalid captcha handling: K2S returns {"status": "error", "message": "Invalid captcha"}. We allow 1 retry with same challenge, then fetch new captcha. Max 3 captchas total.
•	Key reuse: One free_download_key can generate 100-500 tokens through different proxies before the key expires or rate-limits.
•	Proxy exhaustion: Minting 500 tokens burns through ~5-10 proxies (each proxy generates ~50-100 tokens before dying). Need at least 20 validated proxies for a successful mint.
•	Cooldown tracking: K2S enforces cooldowns per IP. Without tracking, we waste time hitting rate-limited proxies. Cooldown file saves {ip: timestamp} and skips hot IPs.
7.3 test_download.py
Purpose: Test the full download engine with a live dashboard.
What It Does:
1.	Loads cached tokens.
2.	Initializes/resumes state manager.
3.	Starts proxy engine and download engine.
4.	Displays live dashboard with workers, progress, speed, ETA, proxy stats.
Inference from Testing — Bugs Found and Fixed:
Bug	Symptom	Root Cause	Fix
112% Progress	Progress exceeded 100%	get_progress_bytes() summed absolute byte positions instead of chunk sizes	Changed to end - start per chunk
All workers on same chunk	Every worker showed "Chunk 0"	No assigned_chunks tracking; all workers grabbed the same active chunk	Added assigned_chunks set to prevent duplicate assignment
Last-minute racing chaos	4 workers on chunk 203, 3 stuck connecting	No priority system; workers piled on recently-released chunks	Implemented chunk_taken_count to prefer un-attempted chunks
"Chunk ---" showing "Downloading"	Idle workers displayed as downloading	State wasn't cleared when get_next_chunk() returned None	_update_worker_state now clears chunk info for non-download states
Double parentheses ))	Dashboard showed Downloading (57s))	Old frame's (resumed 512KB) was longer; new frame didn't clear it	Pad all lines to 120 characters with .ljust(120)
Proxy death wasting tokens	Token returned to pool, caused 409s for next worker	Binding was deleted on proxy death; token tried with wrong IP	Permanent binding: keep marriage, quarantine token if IP dies
Connection timeout not working	Worker stuck "Connecting" for 239 seconds	requests timeout covers connect+read, but TLS handshake hangs beyond both	Added first-byte grace: speed/stale checks only start after data arrives
Memory leak	Memory grew slowly over 20-minute download	stream=True responses never closed on early returns	Added finally: if r is not None: r.close()
Workers don't stop on completion	Process hung after 100% download	Workers stuck inside _download_chunk never checked is_download_complete()	Workers check is_running flag (set to False by monitor on completion)
Performance Observations:
Metric	Value	Notes
Optimal thread count	30	More threads → more proxy churn, same average speed
Per-worker speed	~30 KB/s	K2S hard limit per connection
Total speed (30 workers)	~700-900 KB/s	Limited by proxy supply, not tokens
Tokens consumed per download	~30-50	1-to-1 binding with permanent quarantine
Tokens minted per captcha	~100-250	Depends on proxy quality
Time to download 1GB	~20 minutes	Versus ~9 hours single-threaded
________________________________________
8. Data Management & Persistence
Directory: data/
Auto-created on first run. All runtime data lives here.
Files
File	Created By	Format	Purpose
.requirements.json	main.py	JSON	Lock file: {package: version}. Skips pip checks if valid.
proxies.txt	fresh_proxies.py	Text, 1 per line	Raw scraped proxies (http://ip:port)
connected.txt	proxy_engine.py	Text, 1 per line	Validated working proxies
cooldowns.json	minter.py	JSON	{ip: timestamp} for K2S rate-limit cooldowns
tokens.json	minter.py	JSON	{file_id: {tokens: [{url, mint_time, expiry}]}}
{name}.state.json	state_manager.py	JSON	Chunk progress, active chunks, token pool for resume
{name}.part	download_engine.py	Binary	Pre-allocated file. Workers write at offsets. Renamed to final on completion.
State File Schema ({name}.state.json)
JSON
{
  "file_id": "4d0c30562bcb9",
  "file_name": "movie.mp4",
  "file_size": 1065353216,
  "chunk_size": 5242880,
  "total_chunks": 204,
  "completed_chunks": [0, 1, 4, 5, 12],
  "active_chunks": {
    "2": 3800000,
    "3": 0
  },
  "chunk_taken_count": {
    "0": 1, "1": 1, "2": 3, "3": 2
  },
  "token_pool": {
    "available": ["https://cmb-...", "https://cmb-..."],
    "checked_out": ["https://cmb-..."]
  }
}
•	completed_chunks: Never re-downloaded.
•	active_chunks: Resume from this byte count on restart.
•	chunk_taken_count: Preserved across restarts. Workers still prefer least-attempted chunks.
•	checked_out_tokens: Returned to available on load (workers are gone after crash).
Atomic Write Pattern
Used by both state_manager.py and minter.py:
Python
tmp_path = target_path + ".tmp"
with open(tmp_path, "w") as f:
    json.dump(data, f, indent=2)
os.replace(tmp_path, target_path)
os.replace() is atomic on POSIX and nearly atomic on Windows (within same drive). A crash mid-write leaves either the old file intact or the new file complete. Never a corrupted half-file.
Cleanup
Event	What Gets Cleaned
Download completes	.state.json deleted, .part renamed to final filename, ip_to_token dict cleared
Proxy engine stops	connected.txt saved (warm start next time)
Minter finishes	tokens.json updated, cooldowns.json saved
________________________________________
9. Requirements & Dependencies
Required Packages
Package	Version	Import Name	Purpose
requests	≥2.31.0	requests	HTTP client for K2S API, chunk downloading
rich	≥13.0.0	rich	Future: enhanced terminal formatting
Pillow	≥10.0.0	PIL	Captcha image processing/display
beautifulsoup4	latest	bs4	HTML parsing for proxy scraping
Requirement Checking Flow
text
main.py starts
  → Does data/.requirements.json exist?
    → YES: Try importing each package
      → All succeed? Skip install. Print "OK (cached)"
      → Any fail? Fall through to install flow
    → NO: Install flow
      → For each package:
        → Try import
        → Success? Record version
        → Fail? Run: pip install {package}>=version
        → Record version
      → All met? Save .requirements.json
      → Some failed? Print manual install instructions, exit
Why These Packages
•	requests: The only reliable HTTP library for streaming downloads with proxy support, timeout control, and status code handling.
•	Pillow: Used by minter.py to process and save captcha images locally before opening them with the OS viewer.
•	beautifulsoup4: Used by proxy_engine.py's scraper to parse HTML tables from free proxy list websites.
•	rich: Reserved for future dashboard enhancements (e.g., colored output, progress bars). Not actively used in current rendering.
Standard Library Usage
No external packages are used for:
•	Threading (threading)
•	JSON parsing (json)
•	File operations (os, shutil)
•	Time tracking (time)
•	Process management (subprocess)
This minimizes dependency count and installation failures.
________________________________________
End of Documentation
Message from glm-5.1
