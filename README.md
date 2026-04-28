

# 研究計畫書

## 一、計畫名稱

### 中文題目

**結合 Chandra OCR 2 與輕量化多智能體架構之學術論文影音簡報自動生成系統**

### 英文題目

**A Chandra OCR 2-Augmented Lightweight Multi-Agent Framework for Scientific Paper-to-Video Presentation Generation**

---

## 二、研究摘要

本研究旨在設計並實作一套端到端學術論文影音簡報自動生成系統。系統以學術 PDF 論文作為輸入，透過 Chandra OCR 2 進行高品質文件解析，將 PDF 轉換為結構化 Markdown、HTML、JSON 與圖表資訊，再由 Document Normalizer 將文件內容正規化為統一的論文資料格式。後續系統透過多智能體協作完成論文理解、影片規劃、投影片生成、版面修正、講稿與字幕生成、圖表選擇、內容審查、語音合成與影片輸出。

本研究受 PaperTalker / Paper2Video 架構啟發。PaperTalker 將學術簡報影片生成拆分為 slide generation、layout refinement、subtitling、cursor grounding、speech synthesis 與 talking-head rendering 等模組，並提出 Meta Similarity、PresentArena、PresentQuiz 與 IP Memory 等評估方式。([arXiv][1]) 然而，PaperTalker 偏向完整的 academic presentation video generation，包含 cursor grounding 與 talking-head rendering；本研究第一階段則採取輕量化策略，聚焦於 PDF-first、Chandra OCR 2 文件解析、投影片生成、講稿 / 字幕生成、TTS 語音與 MP4 影片合成，不在第一階段實作 talking-head、cursor grounding 與 GraphRAG。

系統主要部署於具備 NVIDIA RTX 4090 的本地工作站。RTX 4090 具備 24 GB G6X 記憶體，適合做本地 OCR、LLM 推論與影音生成原型。([NVIDIA][2]) 主要生成任務使用本地開源模型處理，Planner Agent 與 Critic Agent 則使用 GPT-5.4 mini 作為雲端輔助模型。GPT-5.4 mini 官方文件指出其適合 high-volume workloads、subagents，支援文字與影像輸入，並具備 400,000 context window。([OpenAI 開發者][3])

---

## 三、研究背景與動機

學術論文通常篇幅長、專業術語多，並且包含複雜的圖表、公式、實驗設計與結果分析。對研究生、專題生或跨領域學習者而言，理解一篇論文往往需要花費大量時間。此外，若要將論文轉換成課堂報告或口頭簡報，使用者還需要額外整理重點、製作投影片、撰寫講稿與準備口頭說明。

近年已有研究開始探討從學術論文自動生成簡報或影片。例如 Paper2Video / PaperTalker 明確提出從 research paper 生成 academic presentation video 的任務，並整合投影片、字幕、游標、語音與 talking-head video rendering。([GitHub][4]) 這代表「論文轉簡報影片」並非完全沒有人做過。因此，本研究不主張該方向為全新，而是針對現有方法仍較少處理的幾個限制提出改良：

```text
1. 現有完整 paper-to-video 系統多依賴 LaTeX source、講者照片或語音樣本。
2. 一般使用者最常取得的是 PDF 論文，而不是完整 LaTeX project。
3. 學術 PDF 具有雙欄排版、圖表、公式、表格與跨頁內容等解析困難。
4. 完整 talking-head 與 cursor grounding 對硬體與模型要求較高。
5. 一般研究生需要的是可快速產出 PPTX、講稿、字幕、語音與 MP4 的輕量化工具。
```

因此，本研究提出一套 **PDF-first、layout-aware、local-deployable、lightweight Paper-to-Video system**。系統使用 Chandra OCR 2 作為 PDF 解析核心。Chandra OCR 2 官方說明其可將圖片與 PDF 轉換為 structured HTML / Markdown / JSON，並保留 layout information；也支援表格、數學式、複雜 layout、圖片與圖表擷取。([GitHub][5])

---

## 四、Design Thinking 導入

本研究不只是技術系統實作，也希望回應使用者在閱讀論文與準備報告時的真實需求。因此，本研究導入 Design Thinking 作為需求分析與系統迭代方法。

Design Thinking 五階段與本研究對應如下：

| Design Thinking 階段 | 本研究對應內容           | 實作方式                                          |
| ------------------ | ----------------- | --------------------------------------------- |
| Empathize          | 理解使用者閱讀論文與準備報告的困難 | 訪談研究生、專題生、跨領域學習者                              |
| Define             | 明確定義使用者痛點         | 建立問題陳述與系統需求                                   |
| Ideate             | 發想技術解決方案          | 設計 Chandra OCR 2 + Lightweight Multi-Agent 架構 |
| Prototype          | 建立系統原型            | 實作 PDF-to-Video MVP                           |
| Test               | 使用者測試與迭代          | 問卷、訪談、系統輸出品質評估                                |

---

## 五、問題定義

根據使用者需求，本研究將核心問題定義為：

> 如何設計一套能自動解析學術 PDF，並將其轉換為易理解、可展示、具備投影片、講稿、字幕、語音旁白與影片輸出的輕量化影音簡報系統？

使用者需求與系統需求對應如下：

| 使用者需求    | 系統需求                  |
| -------- | --------------------- |
| 快速理解論文   | 自動抽取研究問題、方法、貢獻與結果     |
| 減少閱讀負擔   | 產生結構化論文理解摘要           |
| 快速準備報告   | 自動產生投影片與講稿            |
| 希望內容易懂   | 產生口語化、教學式說明           |
| 希望圖表被利用  | 自動擷取並選擇圖表             |
| 擔心 AI 亂講 | 使用 Critic Agent 檢查忠實性 |
| 希望直接展示   | 輸出 PPTX、字幕、音訊與 MP4    |
| 硬體資源有限   | 以 RTX 4090 單卡為主要部署目標  |

---

## 六、研究目的

本研究主要目標如下：

```text
1. 建立 Chandra OCR 2-based 學術 PDF 解析流程。
2. 設計統一的 normalized paper schema。
3. 借鑑 PaperTalker 的 multi-builder 架構，提出輕量化多智能體 Paper-to-Video 流程。
4. 在 RTX 4090 本地環境部署主要生成與影音模組。
5. 使用 GPT-5.4 mini 輔助影片規劃與內容審查。
6. 自動生成投影片、講稿、字幕、語音與 MP4 影片。
7. 引入 Layout Refinement Agent 改善投影片 overflow 與可讀性。
8. 引入 PresentQuiz-lite 與 PresentArena-lite 評估影片是否有效傳達論文知識。
9. 第一階段不做 GraphRAG、cursor grounding 與 talking-head rendering，但保留未來擴充接口。
```

---

## 七、相關研究與差異化

### 7.1 Paper2Video / PaperTalker

Paper2Video / PaperTalker 是目前最接近本研究的工作之一。PaperTalker 將 academic presentation video generation 拆解為多個模組，包含 slide generation、layout refinement、subtitling、cursor grounding、speech synthesis 與 talking-head rendering，並透過 Tree Search Visual Choice 改善投影片排版。([arXiv][1])

Paper2Video / PaperTalker 對本研究最有啟發的部分包括：

```text
1. 將完整影片生成任務拆解為多個 builder / agent。
2. 不使用單一 end-to-end video model，而是模組化生成。
3. 導入 layout refinement 處理投影片 overflow 與可讀性。
4. 使用 PresentQuiz、PresentArena 等方式評估影片是否有效傳達論文內容。
```

然而，本研究與 PaperTalker 的差異在於：

| 面向       | PaperTalker                                                  | 本研究                                                             |
| -------- | ------------------------------------------------------------ | --------------------------------------------------------------- |
| 輸入       | Research paper + speaker identity，偏完整 paper-to-video setting | PDF 論文為主                                                        |
| 文件解析     | 非主打 OCR-based PDF parsing                                    | 使用 Chandra OCR 2 作為核心                                           |
| Slide 生成 | LaTeX Beamer                                                 | 第一階段以 PPTX / slide image 為主，可選 Beamer                           |
| 影音輸出     | Subtitles + cursor + speech + talking-head                   | 第一階段輸出 PPTX + subtitles + TTS + MP4                             |
| 硬體定位     | 完整 pipeline 較重                                               | RTX 4090 單卡可落地                                                  |
| 研究重點     | 完整 academic video generation benchmark                       | PDF-first + layout-aware parsing + lightweight local deployment |

---

### 7.2 Chandra OCR 2

Chandra OCR 2 是本研究文件解析層的核心。官方說明指出 Chandra 可輸出 Markdown、HTML 與 JSON，保留 detailed layout information，並支援表格、數學式、複雜 layout、圖片與圖說擷取。([Hugging Face][6])

因此，本研究將 Chandra OCR 2 放在 Paper-to-Video pipeline 的第一層，用來解決傳統 PDF parser 在學術論文中常見的閱讀順序錯亂、圖說遺失、表格破碎與公式解析不佳等問題。

---

### 7.3 LangGraph 與多智能體流程

本研究使用 LangGraph 作為多智能體流程控制工具。LangGraph 官方文件指出其提供 long-running、stateful workflow 或 agent 的底層基礎架構，支援 durable execution、human-in-the-loop、memory 與 debugging 等能力。([LangChain 文檔][7]) 這適合本研究中 Parser、Planner、Slide Builder、Layout Refiner、Critic 與 Revision 等多個節點之間的狀態管理與修正迴圈。

---

## 八、研究問題

### RQ1：Chandra OCR 2 是否能提升學術 PDF 解析品質？

比較 Chandra OCR 2 與 PyMuPDF / pdfplumber 在章節擷取、圖表擷取、表格保存、閱讀順序與頁首頁尾移除上的差異。

### RQ2：PDF-first 輕量化架構是否能有效替代 LaTeX-first 完整影片生成流程？

比較以 PDF 為輸入的系統是否能在不依賴 LaTeX source、speaker image 與 voice sample 的情況下，產生可用的學術簡報影片。

### RQ3：多智能體架構是否比單一 LLM 直接生成更穩定？

比較 single-agent 與 multi-agent 方法在知識覆蓋率、忠實性、投影片品質與使用者理解度上的差異。

### RQ4：Layout Refinement Agent 是否能改善投影片品質？

測量投影片 overflow rate、字體可讀性、圖文比例、visual clarity 與 human preference。

### RQ5：PresentQuiz-lite 是否能有效評估影片的知識傳達能力？

讓使用者或模型根據影片 / 字幕回答由原始論文產生的問題，衡量影片是否真正幫助理解論文。

---

## 九、研究範圍

### 9.1 第一階段包含

```text
1. Chandra OCR 2 文件解析
2. PDF → Markdown / HTML / JSON / Images
3. Document Normalizer
4. 本地 LLM Agent
5. GPT-5.4 mini Planner / Critic
6. Slide Builder Agent
7. Layout Refinement Agent
8. Script / Subtitle Builder Agent
9. Visual Selection Agent
10. TTS 語音生成
11. MP4 影片合成
12. PresentQuiz-lite / PresentArena-lite 評估
13. 使用者測試與系統評估
```

### 9.2 第一階段不包含

```text
1. 不做 GraphRAG
2. 不建立知識圖譜
3. 不做 cursor grounding
4. 不做 talking-head / avatar
5. 不做 lip-sync
6. 不做完整動畫生成
7. 不做商業化部署
```

---

## 十、系統整體架構

```text
User Upload PDF
        ↓
Chandra OCR 2 Parser
        ↓
Markdown / HTML / JSON / Images
        ↓
Document Normalizer
        ↓
Normalized Paper Schema
        ↓
Paper Understanding Agent
        ↓
Presentation Planner Agent
        ↓
Slide Builder Agent
        ↓
Layout Refinement Agent
        ↓
Script / Subtitle Builder Agent
        ↓
Visual Selection Agent
        ↓
Critic Agent
        ↓
Revision Agent
        ↓
TTS Builder
        ↓
Video Composer
        ↓
Final MP4 Presentation
```

---

## 十一、與 PaperTalker 對應的輕量化 Multi-Builder 設計

| PaperTalker 模組            | 本研究對應模組                              | 第一階段處理方式                              |
| ------------------------- | ------------------------------------ | ------------------------------------- |
| Slide Builder             | Slide Builder Agent                  | 產生 PPTX / slide images                |
| Tree Search Visual Choice | Layout Refinement Agent              | 產生多版面候選並選最佳版本                         |
| Subtitle Builder          | Script / Subtitle Builder Agent      | 產生逐頁講稿與 SRT 字幕                        |
| Cursor Builder            | 暫不實作                                 | 第二階段擴充                                |
| Talker Builder            | TTS Builder + Video Composer         | 第一階段只做語音與 slide video，不做 talking-head |
| Paper2Video metrics       | PresentQuiz-lite / PresentArena-lite | 以輕量方式評估知識傳達效果                         |

---

## 十二、部署架構

```text
Local Workstation: RTX 4090
│
├── Chandra OCR 2
│     └── PDF → Markdown / HTML / JSON / Images
│
├── Document Normalizer
│     └── normalized_paper.json
│
├── Local LLM Server
│     └── Qwen3-14B-Instruct 4-bit / Qwen3-8B fallback
│
├── Local Agents
│     ├── Paper Understanding Agent
│     ├── Slide Builder Agent
│     ├── Layout Refinement Agent
│     ├── Script / Subtitle Builder Agent
│     ├── Visual Selection Agent
│     └── Revision Agent
│
├── Cloud-assisted Agents
│     ├── Presentation Planner Agent: GPT-5.4 mini
│     └── Critic Agent: GPT-5.4 mini
│
├── Media Generation
│     ├── python-pptx
│     ├── LibreOffice headless
│     ├── Edge TTS / Piper TTS
│     ├── MoviePy
│     └── FFmpeg
│
└── Outputs
      ├── normalized_paper.json
      ├── paper_understanding.json
      ├── video_outline.json
      ├── slides.json
      ├── presentation.pptx
      ├── subtitles.srt
      ├── audio files
      └── final_video.mp4
```

---

## 十三、硬體與軟體環境

### 13.1 硬體環境

| 項目      | 規格                               |
| ------- | -------------------------------- |
| GPU     | NVIDIA GeForce RTX 4090          |
| VRAM    | 24 GB G6X                        |
| RAM     | 建議 64 GB 以上                      |
| Storage | 建議 1 TB SSD 以上                   |
| OS      | Ubuntu 22.04 / Windows 11 + WSL2 |
| CUDA    | CUDA 12.x                        |

### 13.2 軟體工具

| 類別                | 工具                       | 用途                           |
| ----------------- | ------------------------ | ---------------------------- |
| OCR / 文件解析        | Chandra OCR 2            | PDF 轉 Markdown / HTML / JSON |
| PDF fallback      | PyMuPDF / pdfplumber     | 快速文字與表格擷取                    |
| Local LLM         | Qwen3-14B-Instruct 4-bit | 本地 Agent 模型                  |
| Cloud LLM         | GPT-5.4 mini             | Planner / Critic             |
| Agent workflow    | LangGraph                | 多 Agent 流程控制                 |
| Backend           | FastAPI                  | API 服務                       |
| Demo UI           | Streamlit                | 快速展示                         |
| Slide generation  | python-pptx              | 產生 PPTX                      |
| Slide export      | LibreOffice headless     | PPTX 轉圖片                     |
| TTS               | Edge TTS / Piper TTS     | 語音生成                         |
| Video             | MoviePy + FFmpeg         | 影片合成                         |
| Schema validation | Pydantic                 | JSON 結構驗證                    |
| Storage           | SQLite / local folder    | 儲存中間結果                       |

---

## 十四、Agent 模型配置

| Agent / Module                  | 使用模型 / 工具                                 | 部署位置        | 任務                                  |
| ------------------------------- | ----------------------------------------- | ----------- | ----------------------------------- |
| Document Parser                 | Chandra OCR 2                             | 本地 RTX 4090 | PDF OCR、layout parsing、圖表擷取         |
| Parser Fallback                 | PyMuPDF / pdfplumber                      | 本地 CPU      | 快速文字與表格 fallback                    |
| Paper Understanding Agent       | Qwen3-14B-Instruct 4-bit                  | 本地 RTX 4090 | 抽取問題、方法、貢獻、實驗與結果                    |
| Presentation Planner Agent      | GPT-5.4 mini                              | OpenAI API  | 規劃影片結構、投影片順序與敘事邏輯                   |
| Slide Builder Agent             | Qwen3-14B-Instruct 4-bit                  | 本地 RTX 4090 | 產生投影片初稿                             |
| Layout Refinement Agent         | Qwen3-VL / GPT-5.4 mini 可選                | 本地 / API    | 檢查 overflow、字太小、圖表不清楚               |
| Script / Subtitle Builder Agent | Qwen3-14B-Instruct 4-bit                  | 本地 RTX 4090 | 產生逐頁講稿與字幕                           |
| Visual Selection Agent          | Qwen3-14B-Instruct 4-bit                  | 本地 RTX 4090 | 根據圖說與表格選擇視覺素材                       |
| Critic Agent                    | GPT-5.4 mini                              | OpenAI API  | 檢查 hallucination、coverage、alignment |
| Revision Agent                  | Qwen3-14B-Instruct 4-bit；必要時 GPT-5.4 mini | 本地 / 雲端混合   | 根據 Critic 意見修正內容                    |
| TTS Builder                     | Edge TTS / Piper TTS                      | 本地          | 產生語音                                |
| Video Composer                  | MoviePy + FFmpeg                          | 本地          | 合成影片                                |

---

## 十五、各模組設計

### 15.1 Chandra OCR 2 Parser

輸入：

```text
paper.pdf
```

輸出：

```text
paper.md
paper.html
paper.json
metadata.json
extracted_images/
```

任務：

```text
1. 擷取論文文字
2. 保留閱讀順序
3. 保留 layout information
4. 擷取表格
5. 擷取公式
6. 擷取圖表與圖片
7. 擷取 captions
```

---

### 15.2 Document Normalizer

功能是將 Chandra 輸出轉換為統一 schema。

```json
{
  "paper_id": "paper_001",
  "title": "...",
  "abstract": "...",
  "sections": [
    {
      "section_id": "sec_1",
      "heading": "Introduction",
      "content": "...",
      "page_start": 1,
      "page_end": 2,
      "figures": ["fig_1"],
      "tables": []
    }
  ],
  "figures": [
    {
      "figure_id": "fig_1",
      "caption": "...",
      "image_path": "images/fig_1.png",
      "related_section": "Method"
    }
  ],
  "tables": [
    {
      "table_id": "tab_1",
      "caption": "...",
      "content": "...",
      "related_section": "Experiment"
    }
  ],
  "references": []
}
```

---

### 15.3 Paper Understanding Agent

使用模型：

```text
Qwen3-14B-Instruct 4-bit
```

任務：

```text
1. 抽取研究問題
2. 抽取研究動機
3. 抽取提出方法
4. 抽取主要貢獻
5. 抽取實驗設定
6. 抽取主要結果
7. 抽取限制
8. 標記重要 figures / tables
```

---

### 15.4 Presentation Planner Agent

使用模型：

```text
GPT-5.4 mini
```

任務：

```text
1. 規劃影片敘事結構
2. 決定投影片數量與順序
3. 控制影片長度
4. 分配每頁投影片時間
5. 根據目標受眾調整內容深度
```

輸出：

```json
{
  "video_title": "...",
  "target_duration": 180,
  "slides": [
    {
      "slide_id": 1,
      "section": "Opening",
      "goal": "Introduce the research problem",
      "duration_seconds": 25
    },
    {
      "slide_id": 2,
      "section": "Method",
      "goal": "Explain the proposed method",
      "duration_seconds": 40
    }
  ]
}
```

---

### 15.5 Slide Builder Agent

使用模型：

```text
Qwen3-14B-Instruct 4-bit
```

任務：

```text
1. 根據 video_outline 產生 slide title
2. 產生 3–5 個 bullet points
3. 加入 visual hint
4. 整合 selected figures / tables
5. 輸出 slides.json 或 PPTX draft
```

---

### 15.6 Layout Refinement Agent

這是吸收 PaperTalker 後新增的關鍵模組。

任務：

```text
1. 對每張投影片產生多個 layout variants
2. 調整 bullet 數量、字體大小、圖片比例、圖文排列
3. 將候選版面轉成圖片
4. 使用 VLM / Critic 選擇最佳版本
5. 降低 overflow、擁擠與圖表不可讀問題
```

Layout variants 範例：

```text
Variant A：文字優先，圖表縮小
Variant B：圖表優先，文字減少
Variant C：左右分欄
Variant D：上下分區
Variant E：只保留核心 bullet
```

評分項目：

```text
1. 是否 overflow
2. 字體是否可讀
3. 圖表是否清楚
4. bullet 是否太多
5. slide goal 是否達成
```

---

### 15.7 Script / Subtitle Builder Agent

使用模型：

```text
Qwen3-14B-Instruct 4-bit
```

任務：

```text
1. 根據每張 slide goal 產生講稿
2. 產生適合 TTS 的短句
3. 產生 SRT subtitle
4. 控制每張 slide 的講稿長度
5. 避免 unsupported claims
```

輸出：

```json
{
  "script": [
    {
      "slide_id": 1,
      "spoken_text": "今天要介紹的是一篇關於...",
      "subtitle_segments": [
        {
          "text": "今天要介紹的是...",
          "start": 0.0,
          "end": 4.2
        }
      ]
    }
  ]
}
```

---

### 15.8 Visual Selection Agent

任務：

```text
1. 讀取 Chandra 擷取出的 figures、tables 與 captions
2. 判斷哪些圖表適合放進簡報
3. 將圖表對應到適合的 slide
4. 說明選擇理由
```

---

### 15.9 Critic Agent

使用模型：

```text
GPT-5.4 mini
```

任務：

```text
1. 檢查講稿是否忠實於論文
2. 檢查是否有 hallucination
3. 檢查是否漏掉主要貢獻
4. 檢查投影片與講稿是否一致
5. 檢查圖表是否使用正確
6. 檢查 layout 是否清楚
7. 給出修改建議
```

輸出：

```json
{
  "faithfulness_score": 4,
  "coverage_score": 4,
  "alignment_score": 5,
  "visual_usage_score": 4,
  "layout_score": 4,
  "issues": [
    {
      "type": "missing_result",
      "location": "slide_6",
      "comment": "Main quantitative result is missing."
    }
  ],
  "revision_suggestions": [
    "Add the key result from the Results section to slide 6."
  ],
  "pass_or_fail": "pass"
}
```

---

### 15.10 Media Agent / Video Composer

使用工具：

```text
python-pptx
LibreOffice headless
Edge TTS / Piper TTS
MoviePy
FFmpeg
```

任務：

```text
1. slides.json → presentation.pptx
2. presentation.pptx → slide images
3. script.json → audio files
4. script.json → subtitles.srt
5. slide images + audio + subtitles → final_video.mp4
```

---

## 十六、系統實作流程

```text
Step 1：使用者上傳 PDF
        ↓
Step 2：Chandra OCR 2 解析 PDF
        ↓
Step 3：Document Normalizer 產生 normalized_paper.json
        ↓
Step 4：Paper Understanding Agent 產生論文理解摘要
        ↓
Step 5：Presentation Planner Agent 規劃影片架構
        ↓
Step 6：Visual Selection Agent 選擇圖表
        ↓
Step 7：Slide Builder Agent 產生投影片初稿
        ↓
Step 8：Layout Refinement Agent 產生與選擇最佳版面
        ↓
Step 9：Script / Subtitle Builder Agent 產生講稿與字幕
        ↓
Step 10：Critic Agent 檢查內容品質
        ↓
Step 11：Revision Agent 修正內容
        ↓
Step 12：TTS Builder 產生語音
        ↓
Step 13：Video Composer 合成 MP4
```

---

## 十七、專案資料夾結構

```text
paper2video-chandra-lightweight/
│
├── backend/
│   ├── app.py
│   ├── config.yaml
│   │
│   ├── api/
│   │   ├── upload.py
│   │   ├── generate.py
│   │   └── status.py
│   │
│   ├── parsers/
│   │   ├── chandra_parser.py
│   │   ├── pymupdf_fallback.py
│   │   ├── pdfplumber_fallback.py
│   │   └── parser_router.py
│   │
│   ├── normalizer/
│   │   ├── markdown_normalizer.py
│   │   ├── html_normalizer.py
│   │   ├── section_extractor.py
│   │   ├── figure_table_mapper.py
│   │   └── paper_schema_builder.py
│   │
│   ├── agents/
│   │   ├── paper_understanding_agent.py
│   │   ├── planner_agent.py
│   │   ├── slide_builder_agent.py
│   │   ├── layout_refinement_agent.py
│   │   ├── script_subtitle_builder_agent.py
│   │   ├── visual_selection_agent.py
│   │   ├── critic_agent.py
│   │   ├── revision_agent.py
│   │   └── media_agent.py
│   │
│   ├── prompts/
│   │   ├── paper_understanding_prompt.txt
│   │   ├── planner_prompt.txt
│   │   ├── slide_builder_prompt.txt
│   │   ├── layout_refinement_prompt.txt
│   │   ├── script_subtitle_prompt.txt
│   │   ├── visual_selection_prompt.txt
│   │   └── critic_prompt.txt
│   │
│   ├── services/
│   │   ├── local_llm_service.py
│   │   ├── openai_service.py
│   │   ├── tts_service.py
│   │   ├── ppt_service.py
│   │   ├── subtitle_service.py
│   │   ├── video_service.py
│   │   └── storage_service.py
│   │
│   ├── schemas/
│   │   ├── paper_schema.py
│   │   ├── understanding_schema.py
│   │   ├── outline_schema.py
│   │   ├── script_schema.py
│   │   ├── slide_schema.py
│   │   ├── layout_schema.py
│   │   └── critic_schema.py
│   │
│   └── outputs/
│       ├── chandra/
│       ├── normalized/
│       ├── scripts/
│       ├── slides/
│       ├── subtitles/
│       ├── audio/
│       └── videos/
│
├── frontend/
│   ├── streamlit_app.py
│   └── web/
│
├── experiments/
│   ├── eval_parsing_quality.py
│   ├── eval_slide_quality.py
│   ├── eval_layout_overflow.py
│   ├── eval_coverage.py
│   ├── eval_faithfulness.py
│   ├── eval_slide_alignment.py
│   ├── eval_presentquiz_lite.py
│   ├── eval_presentarena_lite.py
│   └── human_eval_form.md
│
├── data/
│   ├── papers/
│   ├── processed/
│   └── benchmark/
│
├── requirements.txt
└── README.md
```

---

## 十八、核心 Prompt 設計

### 18.1 Paper Understanding Prompt

```text
You are a scientific paper understanding agent.

Input:
- Structured paper sections
- Figures and captions
- Tables and captions

Task:
Extract the key scientific information needed for a short video presentation.

Return JSON with:
- research_problem
- motivation
- proposed_method
- main_contributions
- experiment_setup
- main_results
- limitations
- important_figures
- important_tables
- keywords

Rules:
- Use only the provided paper content.
- Do not hallucinate.
- Prefer information from abstract, method, experiments, results, and conclusion.
- Return JSON only.
```

### 18.2 Presentation Planner Prompt

```text
You are a video presentation planner.

Create a 5-7 slide academic video outline.

Input:
{paper_understanding}

Requirements:
- Target duration: 3 minutes
- Audience: beginner graduate students
- Structure: opening, problem, method, experiment, result, conclusion
- Each slide must have a clear goal and duration.
- Return JSON only.
```

### 18.3 Slide Builder Prompt

```text
You are a scientific slide builder.

Input:
{video_outline}
{paper_understanding}
{selected_visuals}

Task:
Generate slide content for each slide.

Rules:
- Each slide should have a clear title.
- Each slide should contain 3-5 bullet points.
- Do not copy the full script into the slide.
- Prefer figures, method diagrams, and result tables when useful.
- Return JSON only.
```

### 18.4 Layout Refinement Prompt

```text
You are a slide layout refinement agent.

Input:
- Slide draft
- Rendered slide image
- Slide goal

Evaluate:
1. Is there text overflow?
2. Is the font readable?
3. Is the visual clear?
4. Are there too many bullets?
5. Does the slide match its goal?

Return:
- layout_score: 1-5
- issues
- revision_plan
- selected_variant
```

### 18.5 Script / Subtitle Builder Prompt

```text
You are a scientific script and subtitle builder.

Input:
{video_outline}
{slides}
{paper_understanding}

Task:
Generate spoken script and subtitle segments for each slide.

Rules:
- Use natural spoken language.
- Avoid unsupported claims.
- Keep the content faithful to the paper.
- Make each sentence suitable for TTS.
- Return JSON only.
```

### 18.6 Critic Prompt

```text
You are a scientific content critic.

Check the generated slides, script, subtitles, and visual selections against the source paper.

Evaluate:
1. Faithfulness
2. Knowledge coverage
3. Unsupported claims
4. Missing contributions
5. Slide-script alignment
6. Visual usage correctness
7. Layout clarity
8. Audience suitability

Return:
- faithfulness_score: 1-5
- coverage_score: 1-5
- alignment_score: 1-5
- visual_usage_score: 1-5
- layout_score: 1-5
- issues
- revision_suggestions
- pass_or_fail
```

---

## 十九、實驗設計

### 19.1 Baseline 比較

| 方法         | 說明                                                                 |
| ---------- | ------------------------------------------------------------------ |
| Baseline A | PyMuPDF + Single LLM + PPTX + TTS                                  |
| Baseline B | Chandra OCR 2 + Single LLM + PPTX + TTS                            |
| Baseline C | PyMuPDF + Multi-Agent                                              |
| Baseline D | Chandra OCR 2 + Multi-Agent without Layout Refiner                 |
| Proposed   | Chandra OCR 2 + Multi-Agent + Layout Refinement + PresentQuiz-lite |

---

### 19.2 消融實驗

| 消融項目                       | 目的                                   |
| -------------------------- | ------------------------------------ |
| w/o Chandra OCR 2          | 檢查高品質 PDF parsing 是否有幫助              |
| w/o Planner Agent          | 檢查影片敘事規劃是否重要                         |
| w/o Visual Selection Agent | 檢查圖表選擇是否影響理解                         |
| w/o Layout Refinement      | 檢查版面修正是否降低 overflow                  |
| w/o Critic Agent           | 檢查審查是否降低 hallucination               |
| Local-only vs Hybrid       | 檢查 GPT-5.4 mini Planner / Critic 的增益 |

---

## 二十、評估指標

### 20.1 Parsing Quality

```text
Section Detection Accuracy
Reading Order Correctness
Figure Caption Matching Accuracy
Table Extraction Quality
Header/Footer Removal Quality
```

### 20.2 Slide Quality

```text
Content Score
Design Score
Coherence Score
Overflow Rate
Visual Clarity Score
```

### 20.3 Layout Quality

```text
Text Overflow Rate
Average Bullet Count
Minimum Font Size
Visual Occupancy Ratio
Human Layout Preference
```

### 20.4 Knowledge Coverage

```text
Abstract Coverage
Contribution Coverage
Method Coverage
Experiment Coverage
Result Coverage
```

可使用：

```text
ROUGE-L
BERTScore
Keyword Coverage
LLM-based Coverage Score
```

### 20.5 Faithfulness

```text
Unsupported Claim Count
Hallucination Rate
Faithfulness Score
```

### 20.6 Slide-Script Alignment

```text
Slide bullet vs spoken script semantic similarity
LLM alignment score
```

### 20.7 PresentQuiz-lite

受 Paper2Video 的 PresentQuiz 啟發，本研究設計 PresentQuiz-lite。Paper2Video 使用 PresentQuiz 等指標衡量影片是否能傳達論文資訊。([arXiv][1])

流程：

```text
Step 1：根據原始論文產生 10–15 題問題
Step 2：使用者觀看生成影片
Step 3：使用者回答問題
Step 4：計算答對率
```

自動化版本：

```text
Step 1：LLM 從論文產生 QA
Step 2：LLM / VLM 根據 slides + subtitles 回答問題
Step 3：計算 answer accuracy
```

### 20.8 PresentArena-lite

受 Paper2Video 的 PresentArena 啟發，本研究設計 pairwise comparison：

```text
Baseline A vs Proposed
Baseline B vs Proposed
Baseline D vs Proposed
```

評估者：

```text
1. 人類受試者
2. GPT-5.4 mini judge
```

指標：

```text
Win Rate
Clarity Preference
Completeness Preference
Slide Design Preference
Usefulness Preference
```

---

## 二十一、使用者評估

### 21.1 測試對象

```text
1. 研究生
2. 大學生專題生
3. 跨領域學習者
4. 有論文報告經驗的學生
```

### 21.2 問卷項目

採用 Likert Scale 1–5 分。

| 評估面向  | 問題                |
| ----- | ----------------- |
| 易理解性  | 影片是否幫助你快速理解論文？    |
| 完整性   | 影片是否涵蓋主要問題、方法與結果？ |
| 忠實性   | 影片內容是否符合原始論文？     |
| 投影片品質 | 投影片是否清楚且不過度擁擠？    |
| 語音品質  | 語音是否自然且清楚？        |
| 字幕品質  | 字幕是否有助於理解？        |
| 圖表使用  | 圖表是否有助於理解論文？      |
| 實用性   | 你是否願意使用此系統輔助報告？   |

---

## 二十二、預期成果

本研究預期完成：

```text
1. 一套可上傳 PDF 並自動生成影音簡報的系統。
2. Chandra OCR 2-based scientific paper parsing pipeline。
3. 統一的 normalized_paper.json 論文資料格式。
4. 受 PaperTalker 啟發的 lightweight multi-builder workflow。
5. 自動生成 PPTX、講稿、字幕、語音與 MP4 影片。
6. Layout Refinement Agent 與投影片版面品質評估。
7. PresentQuiz-lite 與 PresentArena-lite 評估方法。
8. 一套技術評估與使用者評估方法。
9. 可擴充至 cursor grounding、talking-head 與 GraphRAG 的系統基礎。
```

---

## 二十三、研究創新點

### 創新點一：PDF-first Lightweight Paper-to-Video Pipeline

相較於完整 academic video generation 系統，本研究以一般使用者更常取得的 PDF 論文為主要輸入，透過 Chandra OCR 2 進行 layout-aware parsing，降低使用門檻。

### 創新點二：PaperTalker-inspired Lightweight Multi-Builder Architecture

本研究借鑑 PaperTalker 的 builder 拆分思想，但將第一階段聚焦於可在 RTX 4090 單卡完成的 slide-video generation，不做 talking-head 與 cursor grounding。

### 創新點三：Chandra OCR 2 強化的學術 PDF 解析

使用 Chandra OCR 2 處理學術論文中的複雜 layout、表格、數學式、圖表與圖片，使後續 Agent 能使用更完整的結構化資料。

### 創新點四：Layout Refinement Agent

吸收 PaperTalker 的 Tree Search Visual Choice 思想，設計輕量化版面修正機制，降低投影片 overflow、文字過多與圖表不清楚問題。

### 創新點五：PresentQuiz-lite 與 PresentArena-lite

將 Paper2Video 的影片評估概念簡化並導入本研究，用於衡量影片是否有效傳達論文知識與使用者偏好。

### 創新點六：Hybrid Local-Cloud Multi-Agent Deployment

多數生成與影音處理流程部署於 RTX 4090 本地工作站，僅將 Planner 與 Critic 交由 GPT-5.4 mini 輔助，以兼顧本地部署、成本控制與生成品質。

---

## 二十四、開發時程

| 階段     | 時間    | 工作內容                                  | 對應 Design Thinking |
| ------ | ----- | ------------------------------------- | ------------------ |
| 第 1 階段 | 1–2 週 | 使用者訪談、需求分析、問題定義                       | Empathize / Define |
| 第 2 階段 | 2 週   | Chandra OCR 2 整合與 PDF 解析測試            | Ideate / Prototype |
| 第 3 階段 | 2 週   | Document Normalizer 與 paper schema 設計 | Prototype          |
| 第 4 階段 | 3 週   | 本地 Qwen3 與 Multi-Agent workflow 實作    | Prototype          |
| 第 5 階段 | 2 週   | GPT-5.4 mini Planner / Critic 串接      | Prototype          |
| 第 6 階段 | 2 週   | Slide Builder 與 Layout Refinement 實作  | Prototype          |
| 第 7 階段 | 2 週   | Script / Subtitle、TTS、影片合成            | Prototype          |
| 第 8 階段 | 3 週   | Baseline、消融實驗與 PresentQuiz-lite       | Test               |
| 第 9 階段 | 2 週   | 使用者測試、分析與報告撰寫                         | Iterate            |

---

## 二十五、風險與解決方案

### 風險一：Chandra OCR 2 推論成本較高

解法：

```text
1. 使用 page range 減少處理頁數。
2. 優先處理 abstract、method、results、conclusion。
3. 若 PDF 有乾淨 text layer，使用 PyMuPDF fallback。
4. 調整 batch size 以符合 RTX 4090 VRAM。
```

### 風險二：本地 Qwen3-14B 生成品質不足

解法：

```text
1. 使用更嚴格的 JSON schema。
2. 將任務拆成更小步驟。
3. 使用 Critic Agent 檢查輸出。
4. 嚴重錯誤時改用 GPT-5.4 mini 修正。
```

### 風險三：Layout Refinement 成本過高

解法：

```text
1. 只對 overflow 或 low-score slides 啟動 refinement。
2. 每張 slide 限制最多 3–5 個 variants。
3. 第一版使用 rule-based layout scoring。
4. 第二版再加入 VLM judge。
```

### 風險四：GPT-5.4 mini API 成本或額度限制

解法：

```text
1. 僅在 Planner 與 Critic 使用 GPT-5.4 mini。
2. 其他生成任務使用本地模型。
3. 保留 Qwen3 local-only 備案。
4. 實驗時限制處理論文篇數與頁數。
```

### 風險五：圖表選擇錯誤

解法：

```text
1. Visual Selection Agent 必須根據 caption 選圖。
2. Critic Agent 檢查 visual usage correctness。
3. 若圖表對應不明確，改用自動生成的流程圖或摘要表。
```

---

## 二十六、未來擴充方向

### Phase 2：Cursor Grounding

借鑑 PaperTalker 的 Cursor Builder 思路，在字幕播放時加入游標或 highlight，引導觀眾注意投影片上的關鍵區域。

### Phase 3：Talking-head / Avatar

加入虛擬人播報、lip-sync 與表情控制，但此階段需要更高硬體資源與更多影音模型整合。

### Phase 4：GraphRAG

將 `normalized_paper.json` 轉換為知識圖譜節點：

```text
Paper node
Section node
Method node
Dataset node
Experiment node
Result node
Figure node
Table node
Claim node
Evidence node
```

接著加入 GraphRAG：

```text
Chandra OCR 2
↓
Normalized Paper Schema
↓
Knowledge Graph Construction
↓
GraphRAG Retrieval
↓
Grounded Multi-Agent Generation
↓
Video Presentation
```

### Phase 5：多語言與風格控制

```text
中文報告
英文報告
初學者版本
專家版本
3 分鐘版本
10 分鐘版本
課堂教學版本
研討會報告版本
```

---

## 二十七、結論

本研究提出一套結合 Chandra OCR 2 與輕量化多智能體架構的學術論文影音簡報自動生成系統。系統受 PaperTalker / Paper2Video 的 multi-builder 思想啟發，但針對一般使用者更常見的 PDF 輸入、RTX 4090 單卡部署限制與研究生實際簡報需求進行重新設計。

本研究第一階段不做 talking-head、cursor grounding 與 GraphRAG，而是優先建立穩定、可展示、可評估的 PDF-to-Video MVP。系統透過 Chandra OCR 2 強化學術 PDF 解析，透過多智能體流程完成論文理解、簡報規劃、投影片生成、版面修正、講稿 / 字幕生成、內容審查與影片合成，並使用 PresentQuiz-lite、PresentArena-lite 與使用者評估驗證影片是否真正幫助理解論文。

因此，本研究的主要貢獻不是宣稱「第一個 paper-to-video 系統」，而是提出一個**PDF-first、layout-aware、local-deployable、lightweight、user-centered** 的學術論文影音簡報生成框架，具有實作可行性、研究評估價值與未來擴充潛力。

---

# 可直接放 proposal 的精簡摘要

本研究旨在設計一套結合 Chandra OCR 2 與輕量化多智能體架構之學術論文影音簡報自動生成系統。系統受 PaperTalker / Paper2Video 的多模組生成架構啟發，但針對 PDF-first 使用情境與 RTX 4090 單卡部署條件進行輕量化改造。系統首先透過 Chandra OCR 2 將 PDF 論文轉換為 Markdown、HTML、JSON 與圖表資訊，再由 Document Normalizer 建立統一的論文結構化資料格式。後續由多個 Agent 分別負責論文理解、影片規劃、投影片生成、版面修正、圖表選擇、講稿與字幕生成、內容審查、語音生成與影片合成。第一階段不實作 GraphRAG、cursor grounding 與 talking-head rendering，而是聚焦於可落地的 PPTX、字幕、TTS 與 MP4 影片輸出。系統主要部署於 RTX 4090 本地工作站，使用本地 Qwen3 量化模型執行多數生成任務，並以 GPT-5.4 mini 輔助 Planner Agent 與 Critic Agent。本研究將透過解析品質、知識覆蓋率、忠實性、投影片版面品質、PresentQuiz-lite、PresentArena-lite 與使用者評估驗證系統可行性。

[1]: https://arxiv.org/html/2510.05096v2 "Paper2Video: Automatic Video Generation from Scientific Papers"
[2]: https://www.nvidia.com/en-us/geforce/graphics-cards/40-series/rtx-4090/ "GeForce RTX 4090 Graphics Cards for Gaming | NVIDIA"
[3]: https://developers.openai.com/api/docs/models/gpt-5.4-mini "GPT-5.4 mini Model | OpenAI API"
[4]: https://github.com/showlab/Paper2Video "GitHub - showlab/Paper2Video: Automatic Video Generation from Scientific Papers · GitHub"
[5]: https://github.com/datalab-to/chandra "GitHub - datalab-to/chandra: OCR model that handles complex tables, forms, handwriting with full layout. · GitHub"
[6]: https://huggingface.co/datalab-to/chandra "datalab-to/chandra · Hugging Face"
[7]: https://docs.langchain.com/oss/python/langgraph/overview "LangGraph overview - Docs by LangChain"
