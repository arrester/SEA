# SEA

> **Cyber Weapon Framework — S**ubSurfer · **E**ndAbyss · **A**ttack

📚 **Languages:** [English](README.md) · [한국어](README_ko.md)

SEA is an offensive automation pipeline framework that takes a single domain as input and chains **subdomain asset discovery → endpoint/parameter collection → vulnerability exploitation** into a single end-to-end flow. Every tool in the stack is connected through standard input/output pipes (`|`), so the entire chain works as a natural Unix one-liner.

---

## 🧩 Toolchain

| Stage | Tool | Role | Repository |
|-------|------|------|------------|
| 1. Recon | **SubSurfer** | Subdomain enumeration · Takeover check · Port scan | [arrester/SubSurfer](https://github.com/arrester/SubSurfer) |
| 2. Crawl | **EndAbyss** | Endpoint / parameter / directory discovery | [arrester/EndAbyss](https://github.com/arrester/EndAbyss) |
| 3-a. Attack | **FUZZmap** | Web vulnerability fuzzing (SQLi / XSS / SSTI) | [offensive-tooling/FUZZmap](https://github.com/offensive-tooling/FUZZmap) |
| 3-b. Attack | **sqlmap** | Dedicated SQL Injection scanner | [sqlmapproject/sqlmap](https://github.com/sqlmapproject/sqlmap) |
| 3-c. Attack | **dalfox** | Dedicated XSS scanner | [hahwul/dalfox](https://github.com/hahwul/dalfox) |

---

## ⚙️ Installation

```bash
pip install subsurfer endabyss fuzzmap sqlmap
brew install dalfox        # macOS
# go install github.com/hahwul/dalfox/v2@latest  # other platforms
```

> If you plan to use EndAbyss in dynamic mode, install Playwright browsers:
> `playwright install chromium`

---

## 🎯 Common Variables & Working Directory

```bash
export TARGET="testasp.vulnweb.com"
mkdir -p sea-results && cd sea-results
```

> **📌 Noise removal:** SubSurfer's underlying `webtech` library prints `Database file is older than 30 days.` to stdout. Every pipeline below filters it with `grep -v "Database file"`.
>
> **📌 URL filter:** EndAbyss may emit a few non-URL lines on stdout, so we keep only valid URLs with `grep -E '^https?://'`.
>
> **📌 Fallback:** If SubSurfer fails to find any live web server (e.g. all passive-result subdomains are dead), we always merge the original target into the next stage so the pipeline never breaks.

---

# 🟢 Normal Test Mode

> Run **one** attack tool at a time. Best when you want clean, focused results.

## ▶ Full Process — `Subdomain → Endpoint → Attack`

The full pipeline starting from SubSurfer reconnaissance: **`-to` (Takeover check) + `-fp` (Fast Port) + EndAbyss + attack tool**.

### Option 1) Full → FUZZmap (general-purpose fuzzing)

```bash
{ subsurfer -t "$TARGET" -to -fp -pipewsub 2>/dev/null | grep -v "Database file"; echo "$TARGET"; } \
  | sort -u \
  | xargs -I{} endabyss -t "http://{}" -pipeurl --silent 2>/dev/null \
  | grep -E '^https?://' | sort -u > urls.txt

xargs -a urls.txt -I{} -P3 fuzzmap -t "{}" -rp -a | tee fuzzmap.log
```

### Option 2) Full → sqlmap (SQL Injection)

```bash
{ subsurfer -t "$TARGET" -to -fp -pipewsub 2>/dev/null | grep -v "Database file"; echo "$TARGET"; } \
  | sort -u \
  | xargs -I{} endabyss -t "http://{}" -pipeurl --silent 2>/dev/null \
  | grep -E '^https?://' | sort -u > urls.txt

sqlmap -m urls.txt --batch --random-agent --smart \
       --threads=10 --level=2 --risk=1 --timeout=15 \
       --output-dir=./sqlmap-out | tee sqlmap.log
```

### Option 3) Full → dalfox (XSS)

```bash
{ subsurfer -t "$TARGET" -to -fp -pipewsub 2>/dev/null | grep -v "Database file"; echo "$TARGET"; } \
  | sort -u \
  | xargs -I{} endabyss -t "http://{}" -pipeurl --silent 2>/dev/null \
  | grep -E '^https?://' | sort -u > urls.txt

dalfox file urls.txt --skip-bav --skip-headless --skip-mining-all \
       --worker 50 --format plain -o dalfox.log
```

---

## ▶ Single Process — `Endpoint → Attack`

Skip SubSurfer when you already know the target host and start directly from EndAbyss.

### Option 1) EndAbyss → FUZZmap

```bash
endabyss -t "http://$TARGET" -pipeurl --silent 2>/dev/null \
  | grep -E '^https?://' | sort -u > urls.txt

xargs -a urls.txt -I{} -P3 fuzzmap -t "{}" -rp -a | tee fuzzmap.log
```

### Option 2) EndAbyss → sqlmap

```bash
endabyss -t "http://$TARGET" -pipeurl --silent 2>/dev/null \
  | grep -E '^https?://' | sort -u > urls.txt

sqlmap -m urls.txt --batch --random-agent --smart \
       --threads=10 --level=2 --risk=1 --timeout=15 \
       --output-dir=./sqlmap-out | tee sqlmap.log
```

### Option 3) EndAbyss → dalfox

```bash
endabyss -t "http://$TARGET" -pipeurl --silent 2>/dev/null \
  | grep -E '^https?://' | sort -u > urls.txt

dalfox file urls.txt --skip-bav --skip-headless --skip-mining-all \
       --worker 50 --format plain -o dalfox.log
```

---

# 🔴 Cyber Weapon Mode

> **Fire all three attack tools in parallel** for maximum surface coverage in minimum time. EndAbyss runs with `-ds` (Directory Scanning) so even hidden endpoints are unearthed.

## ▶ Full Process — `Subdomain → Endpoint(-ds) → Parallel Attack`

```bash
# 1) Recon: SubSurfer → EndAbyss(-ds) → urls.txt
{ subsurfer -t "$TARGET" -to -fp -pipewsub 2>/dev/null | grep -v "Database file"; echo "$TARGET"; } \
  | sort -u \
  | xargs -I{} endabyss -t "http://{}" -ds -pipeurl --silent 2>/dev/null \
  | grep -E '^https?://' | sort -u > urls.txt

echo "[+] Collected $(wc -l < urls.txt) URLs"
[ ! -s urls.txt ] && { echo "[!] urls.txt is empty. Aborting."; exit 1; }

# 2) 3-Way parallel attack
( xargs -a urls.txt -I{} -P5 fuzzmap -t "{}" -rp -a > fuzzmap.log 2>&1 ) & PID_FUZZ=$!

( sqlmap -m urls.txt --batch --random-agent --smart \
         --threads=10 --level=2 --risk=1 --timeout=15 \
         --output-dir=./sqlmap-out > sqlmap.log 2>&1 ) & PID_SQL=$!

( dalfox file urls.txt --skip-bav --skip-headless --skip-mining-all \
         --worker 50 --format plain -o dalfox.log > dalfox.stdout.log 2>&1 ) & PID_DAL=$!

echo "[+] Parallel attack started: FUZZmap($PID_FUZZ) / sqlmap($PID_SQL) / dalfox($PID_DAL)"
wait
echo "[✓] All attacks finished. Check fuzzmap.log / sqlmap.log / dalfox.log"
```

---

## ▶ Single Process — `Endpoint(-ds) → Parallel Attack`

```bash
# 1) Recon: EndAbyss(-ds) → urls.txt
endabyss -t "http://$TARGET" -ds -pipeurl --silent 2>/dev/null \
  | grep -E '^https?://' | sort -u > urls.txt

echo "[+] Collected $(wc -l < urls.txt) URLs"
[ ! -s urls.txt ] && { echo "[!] urls.txt is empty. Aborting."; exit 1; }

# 2) 3-Way parallel attack
( xargs -a urls.txt -I{} -P5 fuzzmap -t "{}" -rp -a > fuzzmap.log 2>&1 ) & PID_FUZZ=$!

( sqlmap -m urls.txt --batch --random-agent --smart \
         --threads=10 --level=2 --risk=1 --timeout=15 \
         --output-dir=./sqlmap-out > sqlmap.log 2>&1 ) & PID_SQL=$!

( dalfox file urls.txt --skip-bav --skip-headless --skip-mining-all \
         --worker 50 --format plain -o dalfox.log > dalfox.stdout.log 2>&1 ) & PID_DAL=$!

echo "[+] Parallel attack started: FUZZmap($PID_FUZZ) / sqlmap($PID_SQL) / dalfox($PID_DAL)"
wait
echo "[✓] All attacks finished. Check fuzzmap.log / sqlmap.log / dalfox.log"
```

> To use a custom wordlist for directory scanning, replace with `endabyss ... -ds -w /path/to/wordlist.txt`.

---

## ⏱️ Expected Runtime & Monitoring Tips

| Stage | Approx. time per host |
|-------|------------------------|
| SubSurfer (`-to -fp`) | 1 ~ 3 min |
| EndAbyss (`-pipeurl`) | 5 ~ 30 sec |
| EndAbyss (`-ds`) | 1 ~ 10 min (depends on wordlist size) |
| FUZZmap (per URL, `-rp -a`) | 5 ~ 20 sec |
| sqlmap (per URL, `--smart`) | 10 sec ~ several min |
| **dalfox (per URL)** | **30 ~ 60 sec** ← slowest |

> **Don't kill it just because there's no instant output.** dalfox and sqlmap can spend tens of seconds to several minutes on a single URL. Run them in the background and tail the log files instead:
>
> ```bash
> tail -f fuzzmap.log sqlmap.log dalfox.log
> ```

---

## 🛠️ Per-Stage Option Cheat Sheet

### SubSurfer
| Flag | Description |
|------|-------------|
| `-t <domain>` | Root domain |
| `-to` | Subdomain **Takeover** check |
| `-fp` | **Fast Port** scan (common web ports only) |
| `-dp` / `-p 80,443,...` | Default / custom port scan |
| `-a` | Enable active scanning |
| `-pipewsub` | Live **web server hosts** to stdout |
| `-pipeweb` | `http(s)://host:port` to stdout |
| `-pipesub` | All collected subdomains to stdout |
| `-pipejson` | Full result as JSON to stdout |

### EndAbyss
| Flag | Description |
|------|-------------|
| `-t <url>` | Target URL |
| `-m {static,dynamic}` | Static or Playwright-based dynamic crawl |
| `-d <N>` | Crawl depth (default 5, unlimited 0) |
| `-c <N>` | Concurrent requests |
| `-ds` | Enable **Directory Scanning** |
| `-w <file>` | Wordlist for directory scan |
| `--silent` | Mute banner / progress (pipe-friendly) |
| `-pipeurl` | URLs (incl. parameters) to stdout |
| `-pipeparam` | Parameters only to stdout |
| `-pipeendpoint` | Endpoints only to stdout |
| `-pipejson` | Full result as JSON to stdout |

### FUZZmap
| Flag | Description |
|------|-------------|
| `-t <url>` | Target URL |
| `-m GET/POST` | HTTP method |
| `-p <name,name>` | Parameters to test |
| `-rp` | **Auto parameter reconnaissance** |
| `-a` | Advanced payloads (deep SQLi, etc.) |

### sqlmap
| Flag | Description |
|------|-------------|
| `-u <url>` | Single URL |
| `-m <file>` | URL list file (multi-target) |
| `--batch` | Auto-answer all prompts with defaults |
| `--smart` | Test only suspicious parameters |
| `--threads=N` | Concurrency |
| `--level / --risk` | Detection depth / risk |
| `--timeout=N` | HTTP response timeout (seconds) |
| `--output-dir=PATH` | Result directory |

### dalfox
| Flag | Description |
|------|-------------|
| `dalfox url <u>` | Single URL mode |
| `dalfox file <f>` | File list mode |
| `dalfox pipe` | stdin pipe mode |
| `--worker N` | Concurrent workers |
| `--skip-bav` | Skip Basic Another Vulnerability checks |
| `--skip-headless` | Skip headless browser verification |
| `--skip-mining-all` | Skip parameter mining (speed↑) |
| `--format plain/json` | Output format |
| `-o <file>` | Output file |

---

## 🧪 Quick Sanity Check

Verify that the SubSurfer/EndAbyss part works against `testasp.vulnweb.com`:

```bash
# (1) Recon only
{ subsurfer -t testasp.vulnweb.com -to -fp -pipewsub 2>/dev/null | grep -v "Database file"; echo "testasp.vulnweb.com"; } | sort -u

# (2) Recon → Crawl → URLs only
{ subsurfer -t testasp.vulnweb.com -to -fp -pipewsub 2>/dev/null | grep -v "Database file"; echo "testasp.vulnweb.com"; } \
  | sort -u \
  | xargs -I{} endabyss -t "http://{}" -pipeurl --silent 2>/dev/null \
  | grep -E '^https?://' | sort -u
```

Step (2) should print roughly 15 parameterized URLs such as `http://testasp.vulnweb.com/showforum.asp?id=1`. **This output is the common input for every attack mode.**

---

## 📁 Result Directory Layout

```
sea-results/
├── urls.txt              # Attack URLs harvested by EndAbyss
├── fuzzmap.log           # FUZZmap output
├── sqlmap.log            # sqlmap stdout
├── sqlmap-out/           # sqlmap session / payload artifacts
├── dalfox.log            # dalfox PoC results
└── dalfox.stdout.log     # dalfox progress log
```

---

## ⚠️ Disclaimer

This framework is intended for use **only on assets you are explicitly authorized to test**. Unauthorized scanning or exploitation is illegal. The user assumes all responsibility and liability for any misuse.

## 📝 License

MIT
