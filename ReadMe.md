# 從「他」→「he」到端到端可信入庫：Hermes / Corpus 本機 AI 可靠性工程實錄  
## 主題：資料完整性、確定性審查、GPU Resident Admission、前台失敗收斂與端到端驗證  
### 附註：色色的力量

> **狀態：2026-09-15**  
> Core correctness E2E 已通過；successful-ingest 前台立即收尾與 scheduler latency 仍待後續優化。

---

## 摘要

這份文件記錄一條看似簡單、實際極度容易失控的本機 AI 入庫流程，如何從：

```text
使用者貼上一篇短篇小說
→ Hermes 前台 Agent
→ Corpus formal ingest
→ 本機 Qwen artifact review
→ qualification
→ canonical / accepted
→ 背景完成通知
```

一路踩出並修正多個彼此獨立、但會互相放大後果的可靠性問題。

最具代表性的事故，是使用者原文明明是：

```text
第二天早上，他沒有去公司。
```

前置 Agent 在重新抄寫 temporary artifact 時，卻把它改成：

```text
第二天早上， he沒有去公司。
```

只是一個「他 → ` he`」的小錯，卻揭露了真正嚴重的架構問題：

> **LLM 不應同時扮演「理解資料」與「搬運 authoritative data」的角色。**

如果今天被改掉的是「他」，後果只是短篇小說驗證變成 `UNCERTAIN`。  
如果明天被改掉的是：

```text
4.25%   → 4.75%
-$3.2B  → $3.2B
2027 H2 → 2027 H1
```

那就可能污染金融資料、研究資料、合約內容或正式決策。

最終工程結果不是「把這篇小說修好」，而是建立一條更可信的本機 AI ingestion pipeline：

```text
Source integrity
→ deterministic provenance
→ bounded admission
→ deterministic local review
→ exact evidence anchoring
→ fail-closed qualification
→ canonical accepted artifact
```

最終正式 E2E：

```text
DELIVERABLE: 1
DELIVERABLE title: 最後一班公車
UNCERTAIN: 0
REJECTED: 0
FAILED: 0
stop reason: local_batch_finished
accepted path: exists
```

accepted artifact 的 SHA-256 對應到乾淨的 1318 字版本，證明整條鏈最終真的吃進了正確內容。

---

## 目錄

1. [背景與目標](#1-背景與目標)
2. [測試環境](#2-測試環境)
3. [核心 regression fixture](#3-核心-regression-fixture)
4. [MIME 與 qualification 基線](#4-mime-與-qualification-基線)
5. [CJK exact quote anchor](#5-cjk-exact-quote-anchor)
6. [Reviewer sampling 與 deterministic extraction](#6-reviewer-sampling-與-deterministic-extraction)
7. [Same-resident GPU admission](#7-same-resident-gpu-admission)
8. [contract.paths 與 durable reservation](#8-contractpaths-與-durable-reservation)
9. [事故核心：「他」變成「he」](#9-事故核心他變成he)
10. [Raw user-turn fidelity](#10-raw-user-turn-fidelity)
11. [`local://user-paste/` provenance boundary](#11-localuser-paste-provenance-boundary)
12. [Exact contiguous fidelity gate](#12-exact-contiguous-fidelity-gate)
13. [真實 Agent call chain 漏傳 `_native_turn_key`](#13-真實-agent-call-chain-漏傳-_native_turn_key)
14. [Deterministic error 與 40/40 runaway](#14-deterministic-error-與-4040-runaway)
15. [最終 E2E 結果](#15-最終-e2e-結果)
16. [效能觀察](#16-效能觀察)
17. [Production hashes](#17-production-hashes)
18. [Regression matrix](#18-regression-matrix)
19. [核心工程原則](#19-核心工程原則)
20. [目前剩餘問題](#20-目前剩餘問題)
21. [Roadmap](#21-roadmap)
22. [結論](#22-結論)

---

# 1. 背景與目標

目標是讓 Hermes 前台 Agent 能將使用者提供的本機文字正式交給 Corpus：

```text
User turn
  ↓
Hermes Agent
  ↓
Corpus formal ingest
  ↓
local artifact review
  ↓
qualification
  ↓
canonical accepted result
  ↓
completion notification
```

這條路徑必須同時滿足：

- 不偷偷搜尋 Web / Telegram / 其他來源；
- authoritative local text 不被 LLM 改寫；
- reviewer 能用 exact evidence 做判斷；
- 正常資源 contention 不誤判成硬失敗；
- 已 resident 的本機模型可以安全重用；
- deterministic front-door error 不建立 durable garbage state；
- 同一 turn 的重試不能因 scope 漂移製造 idempotency conflict；
- 前台提交完成後要能正常交棒，不應自行輪詢；
- 背景完成通知必須能回到原始來源；
- 最終 accepted artifact 必須可由 hash 驗證。

---

# 2. 測試環境

## 2.1 系統

```text
OS: Windows

Hermes plugin root:
%LOCALAPPDATA%\hermes\plugins\corpus

Corpus root:
D:\HermesCorpus

Corpus DB:
D:\HermesCorpus\_state\index.sqlite3
```

Hermes Agent Python：

```text
%LOCALAPPDATA%\hermes\hermes-agent\venv\Scripts\python.exe
```

## 2.2 模型

```text
Model:
qwen3.8-27b-uncensored-q6-128k:latest

Endpoint:
http://localhost:11434/v1

Context:
131072
```

## 2.3 硬體

```text
CPU: Ryzen 9950X
RAM: 192 GB
GPU: RTX 5090 32 GB
```

---

# 3. 核心 regression fixture

本案使用固定短篇《最後一班公車》作為 regression fixture。

## 3.1 乾淨版本

```text
Chars: 1318

SHA256:
258478925277588f7ce51ac0f9f279f5b0355c32a5daa75a57e0f253a33888d8
```

關鍵句：

```text
第二天早上，他沒有去公司。
```

## 3.2 污染版本

```text
Chars: 1320

SHA256:
b856c9203fb4a38cf70c7cfd696718e5f5b46abf3c891829ca97987e62a43c56
```

污染句：

```text
第二天早上， he沒有去公司。
```

## 3.3 差異驗屍

`SequenceMatcher`：

```text
DIFF_COUNT = 1
TAG = replace

CLEAN_RANGE = 1187 1188
HELD_RANGE  = 1187 1190

CLEAN = \u4ed6
HELD  =  he
```

也就是：

```diff
- 他
+  he
```

兩份檔案只有這一處不同。

這個 fixture 後來成為 user-paste fidelity regression 的核心案例。

---

# 4. MIME 與 qualification 基線

早期 `qualification-5` 已修正短 quote anchoring，但 Windows / Python `mimetypes` 在 `.md` 檔案上可能不穩定，導致 Markdown 被錯誤拒絕。

因此 `qualification-6` 增加 deterministic fallback：

```text
.md / .markdown
→ text/markdown
```

這建立了後續所有 exact-evidence 驗證的穩定 MIME 基線。

---

# 5. CJK exact quote anchor

## 5.1 原始問題

Qualification 要求 reviewer 回傳：

```text
opening_quote
development_quote
ending_quote
```

並在原文中找到 exact anchor。

原 layout fallback 可以處理部分 CJK 空白：

```text
中文字 中文字
```

但遇到：

```text
在最後一頁，他加了一句：
「媽，我回來了。」
```

模型若輸出：

```text
在最後一頁，他加了一句：「媽，我回來了。」
```

原本 anchor 可能失敗。

問題不是語義，而是：

```text
：
\n
「
```

屬於排版換行差異。

## 5.2 修正原則

Layout fallback 允許：

- CJK compact script 之間的 layout whitespace 被忽略；
- 真實 line break 位於 CJK script / punctuation boundary 時視為 layout；
- 普通 ASCII word boundary 必須保留；
- 語義標點必須保留；
- 不做 fuzzy matching；
- 不做同義詞修復。

Regression：

```text
CJK_NEWLINE_PUNCT_LAYOUT = PASS
ASCII_WORD_BOUNDARY = PASS
CJK_COMPACT_SPACE = PASS
SEMANTIC_PUNCTUATION = PASS
ANCHOR LAYOUT REGRESSION = PASS
```

## 5.3 安全邊界

以下差異不能被 layout normalization 吞掉：

```text
原文：但站牌旁邊
模型：車站旁邊
```

這是真正的文字變更，不是 layout。

> **Layout normalization 可以處理 layout，但不能修 wording。**

---

# 6. Reviewer sampling 與 deterministic extraction

## 6.1 症狀

同一篇文章，某次 reviewer 正確引用：

```text
「她走以前，還一直跟人家說你工作忙，不怪你沒回來。」
```

另一次卻把：

```text
但站牌旁邊已經沒有人了。
```

改成：

```text
車站旁邊已經沒有人了。
```

也就是：

```text
但   → 省略
站牌 → 車站
```

Qualification 正確拒絕。

## 6.2 Auxiliary task config

`corpus_artifact_review` 當時：

```text
provider = main
timeout = 35
max_tokens = 800
think = false
reasoning_effort = none
local_only = true
single_attempt = true
```

但沒有明確指定：

```text
temperature
top_p
```

因此使用 provider / server sampling default。

## 6.3 修正

Artifact reviewer 專用：

```python
temperature=0 if task=='corpus_artifact_review' else None
```

只鎖 evidence extractor，不影響主聊天、search planner 或其他 auxiliary task。

同時強化 reviewer contract：

```text
Each quote MUST be one single contiguous substring copied exactly
from the captured text.

Never:
- omit a word
- substitute a synonym
- paraphrase
- rewrite punctuation
- splice non-contiguous spans
```

如果無法提供三段 exact quote：

```text
return uncertain
```

而不是自行補字。

## 6.4 真實重播

同一份 artifact 在 `temperature=0 + strict exact quote` 下連跑三次：

```text
RUN 1
opening exact_pos = 0
development exact_pos = 422
ending exact_pos = 1278

RUN 2
opening exact_pos = 0
development exact_pos = 422
ending exact_pos = 1278

RUN 3
opening exact_pos = 0
development exact_pos = 422
ending exact_pos = 1278
```

三輪一致。

---

# 7. Same-resident GPU admission

## 7.1 症狀

Qwen 已 resident：

```text
Qwen VRAM: 約 26 GB
GPU: RTX 5090 32 GB
free VRAM: 約 4.7 GB
GPU compute utilization: low / idle
```

Generic gate 卻只看 free VRAM，因此誤判：

> free VRAM 不足，不能再載模型。

但模型已 resident，不需要 cold load。

## 7.2 Artifact review calibration

24K 文字實測：

```text
elapsed ≈ 12.1 s
input tokens ≈ 17269
output tokens ≈ 290
incremental VRAM ≈ 40 MB
```

較大輸出壓力：

```text
output tokens ≈ 627
elapsed ≈ 17.3 s
incremental VRAM ≈ 198 MB
GPU peak ≈ 98%
```

採用保守 calibration：

```text
required_headroom_mb = 2048
timeout_seconds = 35
max_output_tokens = 800
```

## 7.3 Task-aware resident admission

結果：

```text
same resident + ~4700 MB free → PASS
same resident + 1500 MB free  → DENY
generic cold load + 4700 MB   → DENY
generic cold load + ~30 GB    → PASS
```

避免「模型已在 GPU 上，卻因為它自己佔了 VRAM 而被自己拒絕」。

---

# 8. contract.paths 與 durable reservation

## 8.1 Root cause

Agent 曾把 Windows local path：

```text
C:\Users\...\artifact.txt
```

塞進：

```text
contract.paths
```

但 qualification 的 `contract.paths` 真正語義是：

```text
source URL / ref path-prefix allowlist
```

例如：

```text
/ebooks/123
```

不是 filesystem path。

第一次錯誤 formal ingest 建立 durable reservation 後，再用同一 turn / action 重試不同 scope，觸發：

```text
idempotency_key_conflict
```

接著 front-agent 進入長時間自我 debug。

## 8.2 修正

Schema 明確寫出：

```text
contract.paths:
Source URL/ref path-prefix allowlist.
Never local filesystem path.
```

並在 `life.create()` 前檢查：

```text
Windows drive path         → reject
Windows forward slash path → reject
UNC path                   → reject
valid URL path prefix      → accept
```

核心原則：

> **所有可 deterministic 判定的錯誤，應在 durable state 建立前 fail。**

---

# 9. 事故核心：「他」變成「he」

## 9.1 原始 user turn

```text
第二天早上，他沒有去公司。
```

## 9.2 Front-agent 行為

Telegram UI 顯示：

```text
Writing temp file
Editing temp file
Corpus ingest
grep "he"
Editing temp file
```

代表 authoritative user text 在正式 ingest 前被模型重新抄寫。

## 9.3 正式 held artifact

正式 Corpus snapshot：

```text
SHA256:
b856c9203fb4a38cf70c7cfd696718e5f5b46abf3c891829ca97987e62a43c56

Chars:
1320
```

檢查：

```text
BAD_POS = 1188
GOOD_POS = -1
ASCII_HE_POSITIONS = [1188]
```

Context：

```text
第二天早上， he沒有去公司。
```

## 9.4 Reviewer 並沒有錯

Reviewer：

```text
kind = complete_short_work
matches_request = true
```

三段 exact quote：

```text
ordered = true
verified = 3
candidate_counts = [1,1,1]
valid_triplets = 1
mode = raw
```

但 `unresolved` 包含：

```text
Minor typo in text: 'he沒有去公司' uses Latin 'he' instead of Chinese '他'.
```

因此 `UNCERTAIN` 是合理結果。

```text
Reviewer correctness = PASS
Qualification safety = PASS
Artifact ingress     = FAIL
```

---

# 10. Raw user-turn fidelity

真正根治不能是：

> 「叫模型抄仔細一點。」

LLM 不是 byte-preserving transport。

調查 Hermes runtime 發現：

```text
conversation_loop.py
```

本來就持有：

```text
original_user_message
session_id
turn_id
```

而 `pre_api_request` hook 已傳：

```text
user_message = original_user_message
session_id
turn_id
```

Corpus plugin 本身也支援：

```python
ctx.register_hook(...)
```

因此不需要修改 Hermes core。

新增：

```python
ctx.register_hook('pre_api_request', raw_turn_hook)
```

## 10.1 Write-once cache

Raw-turn cache：

```text
process-local
thread-safe
write-once
TTL = 6h
max turns = 128
max per turn = 8 MiB
```

第一次 `pre_api_request`：

```text
(session_id, turn_id) → original user message
```

後續同一 turn 不得覆寫。

這點重要，因為 Hermes 同 turn redirect / correction 可能會改變 `original_user_message`。

Regression：

```text
RAW_TURN_CAPTURE = PASS
WRITE_ONCE = PASS
```

---

# 11. `local://user-paste/` provenance boundary

不是所有 local ingest 都應受到 user-turn containment。

例如：

```text
Web 下載 PDF
Telegram 抓檔
既有本機文件
影音轉錄
```

都可能合法來自 user turn 之外。

本案已有自然 provenance：

```text
local://user-paste/最後一班公車
```

因此 fidelity gate 僅作用於：

```text
local://user-paste/*
```

其他來源不受影響。

---

# 12. Exact contiguous fidelity gate

正式 local ingest 前：

```text
local artifact
  ↓
UTF-8 decode
  ↓
transport-only CRLF/LF canonicalization
  ↓
artifact 必須是 raw user turn 的 exact contiguous substring
```

允許：

```text
raw user turn:
[Corpus 指令]

[故事全文]

artifact:
[故事全文]
```

因為故事仍是 raw turn 的完整連續 substring。

禁止：

```text
他       → he
4.25%    → 4.75%
-$3.2B   → $3.2B
2027 H2  → 2027 H1
```

## 12.1 明確不做的事

Fidelity gate 不使用：

```text
NFKC
fuzzy matching
semantic similarity
whitespace collapse
paraphrase repair
```

最多只容許：

```text
CRLF ↔ LF
writer-added final newline
```

## 12.2 Durable boundary

Gate 在：

```text
life.create()
```

之前。

污染 artifact：

```text
不建立 campaign
不建立 reservation
不污染 durable state
不製造下一輪 scope conflict
```

Regression：

```text
RAW_TURN_CAPTURE = PASS
WRITE_ONCE = PASS
TURN_IDENTITY_LINK = PASS
EXACT_USER_PASTE = PASS
CRLF_TRANSPORT = PASS
TAMPER_HE_REJECTED = PASS
NON_USER_PASTE_UNAFFECTED = PASS
GATE_BEFORE_LIFECYCLE_CREATE = PASS
```

---

# 13. 真實 Agent call chain 漏傳 `_native_turn_key`

第一次正式新 fidelity gate E2E：

```text
USER_PASTE_RAW_TURN_IDENTITY_MISSING
```

原因不是 raw user turn 沒保存。

Root cause：

```text
pre_tool_call request_hook
```

只傳：

```text
_native_request_id
```

沒有：

```text
_native_turn_key
```

早期單元測試直接呼叫 `attach_runtime_identity()`，因此沒有模擬真正的：

```text
pre_api_request
→ pre_tool_call
→ modified args
→ tool handler
→ dispatch
```

## 13.1 修正

`request_hook()` 增加：

```python
update={
    '_native_request_id': key(...),
    '_native_turn_key': key(
        json.dumps([session_id,turn_id],ensure_ascii=False)
    ),
}
```

## 13.2 真實鏈回歸

```text
PRE_API_RAW_CAPTURE = PASS
PRE_TOOL_IDENTITY_PROPAGATION = PASS
CLEAN_REACHES_LIFECYCLE_CREATE = PASS
TAMPER_BLOCKED_BEFORE_CREATE = PASS

REAL REQUEST_HOOK -> DISPATCH REGRESSION = PASS
```

這個案例證明：

> **Helper-level unit test PASS，不等於 Agent 真實 call chain PASS。**

---

# 14. Deterministic error 與 40/40 runaway

某次 E2E 出現：

```text
USER_PASTE_RAW_TURN_IDENTITY_MISSING
```

Corpus 已 fail。

但前台 Agent 沒停止，反而：

```text
search_files
grep
shell
讀 Hermes source
查 DB
查 hooks
查 turn_id
...
```

最後：

```text
Iteration budget exhausted (40/40)
```

整輪約：

```text
08:10 → 08:19
```

浪費約 9 分鐘。

## 14.1 問題本質

Deterministic error 被 Agent 當成：

> 「我要自己做軟體工程，把整個 runtime 修到好再回覆。」

這不是正常 failure recovery。

## 14.2 利用既有 handoff middleware

Corpus 已有 `handoff.py`。

成功 collection 後原本就會：

```text
tool_choice = none
tools = []
```

因此擴充同一 middleware，只針對：

```text
USER_PASTE_*
```

deterministic front-door errors。

行為：

```text
Error once
→ preserve exact error code
→ append terminal notice
→ tool_choice = none
→ tools = []
→ no retry
→ no source inspection
→ end turn
```

Regression：

```text
TERMINAL_ERROR_DETECTED = PASS
TOOLS_DISABLED = PASS
ERROR_CODE_PRESERVED = PASS
NO_REPEAT_TERMINAL_NOTICE = PASS
NON_USER_PASTE_ERROR_UNAFFECTED = PASS
```

一般 transient Corpus error 不會被吞掉。

---

# 15. 最終 E2E 結果

第三輪正式 E2E：

```text
10:24  使用者送出乾淨 fixture
~10:25 正式 ingest 流程開始
~10:27 前台回正式提交完成
10:28  Corpus completion notification
```

最終：

```text
DELIVERABLE: 1
DELIVERABLE titles: 最後一班公車

REFERENCE_ONLY: 0
UNCERTAIN: 0
REJECTED: 0
FAILED: 0

stop reason:
local_batch_finished
```

Accepted path 存在。

更重要的是 accepted artifact filename / hash 對應：

```text
258478925277588f...
```

也就是乾淨的 1318 字版本。

正式判定：

```text
Raw user-turn capture            PASS
Turn identity propagation        PASS
User-paste exact fidelity        PASS
Formal ingest                    PASS
Same-resident reviewer           PASS
Reviewer deterministic sampling  PASS
Exact quote anchoring            PASS
Qualification                    PASS
Canonical accepted artifact      PASS
Completion notification          PASS
```

這是第一次真正完整乾淨的 E2E PASS。

---

# 16. 效能觀察

## 16.1 Reviewer inference 其實不慢

曾精確量到：

```text
content review started → finished
≈ 5.39 秒
```

24K calibration：

```text
≈ 12.1 秒
```

較大輸出壓力：

```text
≈ 17.3 秒
```

因此 Qwen inference 不是主要總延遲來源。

## 16.2 Scheduler 曾造成約 155 秒等待

某輪：

```text
job created:
06:51:49

review started:
06:54:24
```

等待：

```text
≈ 155 秒
```

而 inference 本身只有約 5 秒。

這表示：

```text
scheduler / native tick
```

曾是核心後端延遲來源。

## 16.3 前台成功後仍不肯收手

最終成功 E2E 中，formal ingest 已成功後，front-agent 還自己做：

```text
sqlite
ls
staging
events
grep uncertain
batches
讀檔
```

約多浪費 2 分鐘。

這是目前下一個最明確的 latency bug。

---

# 17. Production hashes

目前已驗證 production：

```text
lifecycle.py
804591EC2E8C42F9ECA4980B8EFA6759552FD173D8583DB9B00FF35EE0D28BBE

handoff.py
9D1218580F806BD7B3DDA88BC2CF302CF864EB1459A9DAB2ADA04D08A1FDF6DC

api.py
CFC112F52BD628477A2E26D2ABEEA0529D78D753C6B56050DF8CB71D6AEF957E

qualification.py
ECA5779311D3A3AB0E49B8B1CF5E9F177A9255EFFC1D291E32CAFC9BF36C69F3

search_plan.py
6EFE5C928ABC0F9B56FB4FA4150BE1994846E1CD6165E020E3A17CDC7B643BAA
```

> 這些 hash 是本次實際 production 狀態的工程紀錄，不是 upstream Hermes / Corpus 官方版本識別。

---

# 18. Regression matrix

## 18.1 Quote / layout

```text
CJK_NEWLINE_PUNCT_LAYOUT = PASS
ASCII_WORD_BOUNDARY = PASS
CJK_COMPACT_SPACE = PASS
SEMANTIC_PUNCTUATION = PASS
ANCHOR LAYOUT REGRESSION = PASS
```

## 18.2 Reviewer determinism

```text
temperature=0 artifact review
3 consecutive runs
3/3 exact opening
3/3 exact development
3/3 exact ending
```

## 18.3 Same-resident admission

```text
SAME_RESIDENT_4700MB = PASS
BELOW_HEADROOM_1500MB = PASS
GENERIC_4700MB_DENIED = PASS
GENERIC_30400MB_RECOVERS = PASS
```

## 18.4 User-paste fidelity

```text
RAW_TURN_CAPTURE = PASS
WRITE_ONCE = PASS
TURN_IDENTITY_LINK = PASS
EXACT_USER_PASTE = PASS
CRLF_TRANSPORT = PASS
TAMPER_HE_REJECTED = PASS
NON_USER_PASTE_UNAFFECTED = PASS
GATE_BEFORE_LIFECYCLE_CREATE = PASS
```

## 18.5 Real Agent chain

```text
PRE_API_RAW_CAPTURE = PASS
PRE_TOOL_IDENTITY_PROPAGATION = PASS
CLEAN_REACHES_LIFECYCLE_CREATE = PASS
TAMPER_BLOCKED_BEFORE_CREATE = PASS
REAL REQUEST_HOOK -> DISPATCH REGRESSION = PASS
```

## 18.6 Terminal deterministic failure

```text
TERMINAL_ERROR_DETECTED = PASS
TOOLS_DISABLED = PASS
ERROR_CODE_PRESERVED = PASS
NO_REPEAT_TERMINAL_NOTICE = PASS
NON_USER_PASTE_ERROR_UNAFFECTED = PASS
```

## 18.7 Final E2E

```text
DELIVERABLE = 1
UNCERTAIN = 0
REJECTED = 0
FAILED = 0
accepted artifact hash = clean artifact hash
```

---

# 19. 核心工程原則

## 19.1 LLM 不得成為 authoritative transport

LLM 可以：

```text
理解
規劃
分類
推理
判斷
```

但不應負責：

```text
重新抄寫 authoritative artifact
```

資料搬運層必須 deterministic。

---

## 19.2 Validator 不應因模型不穩而放寬

錯誤策略：

```text
Qwen 改字
→ validator 改成 fuzzy match
```

正確策略：

```text
Qwen 改字
→ reviewer 更 deterministic
→ instructions 更嚴格
→ validator 保持 exact
```

證據層應該比模型更嚴，而不是更寬。

---

## 19.3 Deterministic front-door error 必須發生在 durable state 前

包括：

```text
contract path misuse
user-paste fidelity mismatch
invalid provenance
invalid local path
```

避免：

```text
invalid request
→ reservation
→ retry
→ scope drift
→ idempotency conflict
```

---

## 19.4 Deterministic error 必須具有 terminal semantics

否則 Agent 會把：

```text
一個明確錯誤
```

擴張成：

```text
40 iterations 的自主 debug 專案
```

失敗本身不可怕。

**不肯停止的失敗才可怕。**

---

## 19.5 Regression 必須模擬真實 call chain

本案曾經：

```text
attach_runtime_identity() unit test
→ PASS
```

但真實：

```text
pre_api_request
→ pre_tool_call
→ handler
```

卻漏掉 `_native_turn_key`。

因此 future regression 應優先測：

```text
Hook
→ middleware
→ tool args
→ dispatch
→ durable boundary
```

而不是只測 helper function。

---

## 19.6 正常業務結果不等於 system failure

以下結果不應自動 cloud escalation：

```text
ZERO_RESULTS
normal REJECTED
normal UNCERTAIN
duplicate
resource defer
bounded transient dependency
```

真正值得 escalation 的是：

```text
system-level repeated failure
progress invariant violation
local recovery exhausted
planner main path failure
same failure signature repeated across ticks
failure cannot be reliably classified
```

---

# 20. 目前剩餘問題

目前 correctness 已完整 E2E PASS。

剩下主要是 latency / orchestration。

## 20.1 Successful local ingest foreground handoff

目前：

```text
successful local ingest
+ pending_review=true
```

之後 front-agent 還會自行查：

```text
DB
staging
events
batches
uncertain state
```

這些應全部消失。

建議：

```text
formal ingest success
+ pending_review=true
+ background completion owner ready
→ tool_choice = none
→ tools = []
→ 回「已提交，等待 Corpus 完成通知」
→ end turn
```

這應可砍掉約 2 分鐘前台延遲。

## 20.2 Event-driven local review continuation

不要只粗暴縮短 Cron ticker。

較好的 Phase C：

```text
formal ingest
→ durable event
→ immediate local review continuation
```

而不是：

```text
formal ingest
→ 等下一班 scheduler tick
```

目標是保留：

```text
durability
retry semantics
observability
bounded work
```

同時減少 60～155 秒空等。

---

# 21. Roadmap

## Phase C1 — Successful-ingest foreground handoff

驗收：

```text
ingest success
→ <= 幾秒前台收尾
→ no shell
→ no sqlite
→ no grep
→ no active polling
```

## Phase C2 — Scheduler timestamp instrumentation

每一輪精確記錄：

```text
tool submission
campaign creation
job creation
review scheduled
review started
review finished
qualification committed
campaign stopped
notification emitted
```

不要再用 Telegram 分鐘顯示猜 latency。

## Phase C3 — Event-driven continuation

把：

```text
reviewable local artifact
```

轉成 durable immediate continuation event。

## Phase C4 — Broader provenance fidelity

將 user-paste 的思路延伸到：

```text
attachments
downloaded reports
financial CSV
contracts
transcripts
```

核心：

```text
authoritative source bytes
→ immutable hash
→ every derived artifact keeps provenance link
```

## Phase C5 — Failure taxonomy

正式定義：

```text
business terminal
business uncertain
dependency defer
resource defer
transport transient
system invariant failure
operator-action-required
```

並明確決定：

```text
retry?
notify?
cloud escalate?
end turn?
```

避免所有 error 都被 Agent 當成「我要自己修系統」。

---

# 22. 結論

本案一開始只是：

> 「為什麼同一篇短篇小說有時候過、有時候 UNCERTAIN？」

最後真正揭露的是一整套本機 AI reliability 問題：

```text
MIME
CJK layout
sampling randomness
GPU resident admission
contract semantics
idempotency
artifact provenance
LLM copy corruption
hook identity propagation
front-agent runaway recovery
scheduler latency
foreground handoff
```

最重要的成果不是某一個 patch。

而是建立了幾條應長期保留的 architectural invariants：

```text
Authoritative content cannot be rewritten by the LLM.
Evidence must remain exact.
Deterministic validation must happen before durable ownership.
Deterministic failures must terminate deterministically.
Resident resources must be evaluated task-aware.
Real regressions must cover the real call chain.
```

最終：

```text
使用者原文
→ raw-turn capture
→ exact fidelity gate
→ formal ingest
→ deterministic review
→ exact quote anchor
→ qualification
→ canonical accepted
→ completion notification
```

整條鏈已能乾淨通過。

而那個把整件事情掀開的事故，只是一個看似荒謬的小錯：

```diff
- 他
+  he
```

但也正因為這個小錯，整條 ingestion pipeline 被迫從「大概能用」提升到「可以開始談可信」。

---

## 最後一句

> **色色可以自由，artifact 不行。**

這就是本案最重要的工程結論。
