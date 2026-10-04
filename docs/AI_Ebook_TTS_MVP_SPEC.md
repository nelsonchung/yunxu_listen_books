# AI 電子書語音朗讀 App — MVP 實作規格

> Version: 0.1  
> Status: MVP / Proof of Concept  
> Purpose: 本文件可直接提供給 Coding AI 作為第一版實作依據。

## 1. Product Goal

建立一個「用聽的方式閱讀電子書」的 App。

核心原則：

1. **忠實原文**：不得摘要、改寫、補充、解釋或加入 AI 評論。
2. **自然朗讀**：允許透過 TTS 控制語速、停頓、重音與語氣，使長篇聆聽不機械。
3. **可續聽**：自動保存使用者最後聽到的位置，下次可以繼續播放。
4. **TTS 可替換**：App 不綁定單一 TTS provider，可比較本機、雲端與 self-hosted 模型。
5. **先完成端到端 MVP**：先跑通 EPUB → 解析 → TTS → 播放 → 保存進度 → 恢復播放。

---

## 2. MVP Scope

### 必做

- 匯入 EPUB
- 解析書名、章節、段落
- 以段落為基本朗讀單位
- 呼叫 TTS Provider 產生音訊
- 播放 / 暫停
- 上一段 / 下一段
- 快退 15 秒 / 快進 15 秒
- 語速調整
- 顯示目前章節與段落
- 自動保存播放進度
- App 重開後恢復上次位置
- 已生成語音使用 Cache，避免重複 TTS

### 暫不做

- PDF OCR
- AI 摘要
- AI 問答
- 翻譯
- 多角色 voice casting
- 跨裝置同步
- 社群功能
- DRM 破解

---

## 3. Core User Flow

```text
Import EPUB
    ↓
Parse metadata / chapters / paragraphs
    ↓
Store normalized book structure
    ↓
Select paragraph
    ↓
Check audio cache
    ├── HIT  → play cached audio
    └── MISS → request TTS → cache → play
    ↓
Continuously save listening progress
    ↓
Close App
    ↓
Open App again
    ↓
Offer:
  - Continue from last position
  - Start current chapter from beginning
  - Start book from beginning
```

---

## 4. Content Integrity Rule

原文與朗讀表現必須分離。

```text
Content Layer     = immutable source text
Presentation Layer = TTS parameters / prosody metadata
```

例如：

```text
原文：
「你真的要走嗎？」她低聲問道。
```

允許：

- pause
- speaking rate
- emphasis
- pitch / volume（若 provider 支援）
- style / emotion hint（若 provider 支援）

禁止：

```text
她很難過地詢問對方是否真的要離開。
```

### Integrity Requirement

送入 TTS 的 spoken text 必須可以追溯到原始 EPUB 文字。

若未來加入 LLM 作為「朗讀導演」，LLM **只能輸出控制 metadata，不得輸出替代朗讀文字**。

---

## 5. Book Data Model

```ts
type Book = {
  id: string
  title: string
  author?: string
  sourceFileName: string
  importedAt: string
  chapters: Chapter[]
}

type Chapter = {
  id: string
  bookId: string
  index: number
  title?: string
  paragraphs: Paragraph[]
}

type Paragraph = {
  id: string
  chapterId: string
  index: number
  text: string
  textHash: string
}
```

ID 建議：

```text
book_<uuid>
chapter_<uuid>
paragraph_<uuid>
```

不要只依賴 chapter index / paragraph index 作永久識別。

---

## 6. Listening Progress Model

```ts
type ListeningProgress = {
  bookId: string
  chapterId: string
  paragraphId: string
  audioPositionMs: number
  playbackRate: number
  updatedAt: string
}
```

### Save Policy

至少在以下事件保存：

- 每 5～10 秒
- Pause
- Paragraph changed
- App background
- App terminated / lifecycle callback 可用時

### Resume

重新進入書籍：

```text
Continue listening
→ chapterId
→ paragraphId
→ regenerate/load audio
→ seek(audioPositionMs)
→ play
```

---

## 7. TTS Abstraction

不要讓 UI 直接依賴任何特定 TTS SDK。

```ts
interface TTSProvider {
  synthesize(request: TTSRequest): Promise<TTSResult>
}

type TTSRequest = {
  text: string
  language: string
  voiceId?: string
  speakingRate?: number
  style?: TTSStyle
}

type TTSStyle = {
  pauseProfile?: string
  emphasis?: string[]
  emotion?: string
}

type TTSResult = {
  audioUri: string
  durationMs: number
  provider: string
  model?: string
}
```

Provider Adapter：

```text
TTSProvider
 ├── AppleTTSProvider
 ├── KokoroProvider
 ├── CloudTTSProvider
 └── FutureProvider
```

第一版只需要實作 1～2 個 Provider，但架構必須可替換。

---

## 8. TTS Server API

若採 self-hosted：

### POST `/v1/tts`

Request:

```json
{
  "text": "「你真的要走嗎？」她低聲問道。",
  "language": "zh-TW",
  "voice_id": "default",
  "speaking_rate": 1.0
}
```

Response:

```json
{
  "audio_id": "audio_xxx",
  "audio_url": "/v1/audio/audio_xxx",
  "duration_ms": 4230,
  "cached": false
}
```

### Cache Key

建議：

```text
SHA256(
  text +
  provider +
  model +
  voiceId +
  language +
  speakingRate +
  styleParameters
)
```

相同輸入不得重複生成。

---

## 9. Paragraph Segmentation

第一版：

1. 優先使用 EPUB 原始 paragraph 結構。
2. 過長 paragraph 再依句號、問號、驚嘆號等切成 TTS chunks。
3. 播放器 UI 仍以 paragraph 為閱讀位置單位。

建議先設定：

```text
TTS chunk target: 100～300 中文字
```

此數值是初始實驗值，必須透過實際模型測試調整。

不要一開始把整章一次送進 TTS。

---

## 10. Audio Cache

```ts
type AudioCacheEntry = {
  cacheKey: string
  paragraphId: string
  provider: string
  model?: string
  voiceId?: string
  localPath?: string
  remoteUrl?: string
  durationMs: number
  createdAt: string
}
```

播放流程：

```text
paragraph
   ↓
calculate cache key
   ↓
cache exists?
 ┌───────┴───────┐
 YES              NO
 ↓                 ↓
play           call TTS
                   ↓
                 cache
                   ↓
                  play
```

---

## 11. Player State

```ts
type PlayerState = {
  bookId: string
  chapterId: string
  paragraphId: string
  positionMs: number
  durationMs: number
  playbackRate: number
  status: "idle" | "loading" | "playing" | "paused" | "error"
}
```

播放器：

- Play / Pause
- Previous paragraph
- Next paragraph
- -15 sec
- +15 sec
- Playback speed: 0.75x / 1.0x / 1.25x / 1.5x / 2.0x

---

## 12. Suggested Architecture

```text
┌───────────────────────────┐
│        Mobile App         │
│                           │
│ EPUB Import               │
│ Book Library              │
│ Player                    │
│ Progress Store            │
└─────────────┬─────────────┘
              │ HTTPS
              ▼
┌───────────────────────────┐
│        Backend API        │
│                           │
│ TTS orchestration         │
│ Cache management          │
│ Optional account sync     │
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────┐
│       TTS Provider        │
│                           │
│ Self-hosted / Cloud       │
└───────────────────────────┘
```

MVP 若要降低複雜度，可以先讓 EPUB parsing 與 progress 存在手機本地端，Server 只負責 TTS。

---

## 13. Suggested Repository Structure

```text
project/
├── app/
│   ├── book/
│   ├── epub/
│   ├── player/
│   ├── progress/
│   ├── tts/
│   └── storage/
│
├── server/
│   ├── api/
│   ├── tts/
│   │   ├── providers/
│   │   └── models/
│   ├── cache/
│   └── tests/
│
├── docs/
│   ├── SPEC.md
│   └── TTS_EVALUATION.md
│
└── README.md
```

---

## 14. TTS Evaluation

建立固定測試文章，所有模型使用完全相同文字。

評估：

| Metric | Description |
|---|---|
| Mandarin naturalness | 中文是否自然 |
| Taiwan pronunciation | 台灣使用者聽感 |
| Prosody | 停頓、重音、節奏 |
| Dialogue | 人物對話自然度 |
| Long listening comfort | 20～30 分鐘是否疲勞 |
| Latency | 生成速度 |
| RTF | 生成時間 / 音訊長度 |
| Hardware | CPU / GPU / RAM / VRAM |
| Cost | 每 10 萬字成本 |
| License | 是否允許預定使用方式 |

不要只用「像不像真人」作為唯一指標。

---

## 15. MVP Acceptance Criteria

### AC-01 Import

Given 一個有效 EPUB  
When 使用者匯入  
Then App 顯示書名與章節。

### AC-02 Playback

Given 已解析 paragraph  
When 使用者按 Play  
Then 系統產生或取得 TTS audio 並播放。

### AC-03 Integrity

Given EPUB 原始 paragraph  
When 產生語音  
Then spoken text 不得被摘要、改寫或加入額外句子。

### AC-04 Progress

Given 使用者正在 Chapter 3 / Paragraph 12 / 37 秒  
When App 保存狀態並關閉  
Then 下次可以回到同一位置附近繼續播放。

### AC-05 Cache

Given 同一 paragraph 與相同 TTS parameters 已生成  
When 再次播放  
Then 不重新執行 TTS inference。

### AC-06 Provider Replaceability

Given TTSProvider interface  
When 更換 Provider implementation  
Then Player / EPUB parser 不需要修改核心邏輯。

---

## 16. Implementation Phases

### Phase 1 — Vertical Slice

只做：

```text
EPUB
→ parse
→ first chapter
→ first paragraph
→ TTS
→ audio playback
```

### Phase 2 — Continuous Reading

```text
paragraph N
→ paragraph N+1
→ pre-generate next audio
→ continuous playback
```

### Phase 3 — Progress

加入：

```text
save
resume
chapter navigation
```

### Phase 4 — TTS Comparison

同一篇文字測：

```text
Provider A
Provider B
Provider C
```

紀錄品質、latency、硬體需求與成本。

### Phase 5 — Natural Reading Research

研究：

```text
Raw text
   ↓
Prosody Analyzer / LLM Director
   ↓
TTS control metadata
   ↓
TTS
```

**LLM Director 不得修改原文。**

---

## 17. Coding AI Initial Task

Coding AI 應先完成以下任務，不要一次實作完整產品：

> 建立最小 vertical slice。使用者可以選擇一個 EPUB，程式解析第一章與第一個有效段落，透過 `TTSProvider` interface 送入一個可工作的 TTS implementation，產生語音並播放。建立基本 log，顯示 book、chapter、paragraph ID、TTS latency 與 cache hit/miss。完成後加入自動測試，確認送入 TTS 的文字與解析後原文完全相同。

完成此 milestone 後，再進行 continuous playback 與 progress persistence。

---

## 18. Definition of Done — First Milestone

第一個 milestone 完成條件：

```text
[ ] 可以匯入 EPUB
[ ] 可以解析 Chapter
[ ] 可以解析 Paragraph
[ ] 可以將原文送入 TTS
[ ] 可以產生 Audio
[ ] 可以播放 Audio
[ ] 可以 Pause
[ ] 有 TTS Provider abstraction
[ ] 有基本 Cache
[ ] 有原文 integrity test
[ ] 有 log 可以觀察 latency
```

---

## 19. Product Principle

整個專案後續所有功能都必須遵守：

> **AI 可以決定「怎麼念」，但不能決定「書裡寫了什麼」。**

第一階段的成功不是做出世界上最好聽的 AI 有聲書，而是建立一條可靠、可替換、可測量的完整技術鏈：

**EPUB → Text → TTS → Audio → Playback → Progress → Resume**
