# 🎼 AI Music Score Simplifier

[English](#english) | [中文](#中文)

---

<a id="english"></a>

## Intelligent Graded Simplification for Music Education

**Try it online**: [ModelScope Demo](https://www.modelscope.cn/studios/JeffreyZhou2026/AI-Music-Score-Simplifier-13) https://www.modelscope.cn/studios/JeffreyZhou2026/AI-Music-Score-Simplifier-13/summary

[![Python 3.10](https://img.shields.io/badge/Python-3.10-green)](https://www.python.org/)
[![Gradio 6.2](https://img.shields.io/badge/Gradio-6.2.0-orange)](https://www.gradio.app/)
[![License](https://img.shields.io/badge/License-MIT-blue)](LICENSE)

This application provides **intelligent graded simplification** for music scores, helping beginners choose simplified versions appropriate for their skill level. It prevents frustration caused by overly complex scores and gradually transitions learners to the original compositions.

> **Core principle**: Simplification means *preserving note pitches while removing, merging, or extending notes* — not generating new ones.

---

## Key Features

- **5 Graded Levels** — From skeleton outline to near-original fidelity
- **Dual Mode** — Single-staff (monophonic: violin, flute…) + Grand-staff (polyphonic: piano, organ…)
- **Auto Voice Detection** — SB/SAB/STB/SATB with manual correction
- **LOCKED Protection** — 5 types of structurally important notes are frozen from modification
- **AI-Assisted Analysis** — Pre-trained model provides semantic understanding for melody detection, importance scoring, and turning-point identification
- **Zero-Shot Inference** — No training required; uses pre-trained models directly
- **Metadata Preservation** — Title, composer, tempo, key/time signatures retained
- **English UI** — All interface elements in English with level descriptions
- **Progress Tracking** — Real-time progress bar during simplification pipeline
- **MusicXML Export** — Compatible with MuseScore and other notation software

---

## Simplification Levels

### Single-Staff Mode (Monophonic Instruments)

| Level | Name | Rule | Example (4/4) |
|:-----:|------|------|----------------|
| **1** | Skeleton | Keep only 1st note per bar, extend to full measure | 1 whole note per bar |
| **2** | Strong Beats | Keep strong-beat notes only, extend to next strong beat | Beats 1, 3 → expanded |
| **3** | Beat Heads | Keep 1st note of each beat, extend to next kept note | Beats 1, 2, 3, 4 preserved |
| **4** | Rhythm Preserved | Remove grace notes; shorten ≤16th → 8th; LOCKED active | All ≥8th retained |
| **5** | Near Original | Remove grace notes/ornaments only | Near-original fidelity |

**Strong Beat Map**: 2/4, 3/4 → beat 1; 4/4 → beats 1, 3; 6/8 → beats 1, 4.

**LOCKED Note Types** (protected from simplification):
1. Syncopation (weak-beat start + tie spanning strong beat)
2. Dotted rhythm main note (≥ eighth duration)
3. Tie across beat/measure boundary
4. Melodic contour turning point
5. Phrase/cadence endpoint

### Grand-Staff Mode (Polyphonic Instruments)

The system auto-detects voice structure type and allows manual correction:

| Type | Voices | Description |
|------|--------|-------------|
| **SB** | Soprano + Bass | Melody + simple accompaniment |
| **SAB** | Soprano + Alto + Bass | 3-part harmony |
| **STB** | Soprano + Tenor + Bass | 3-part harmony |
| **SATB** | Soprano + Alto + Tenor + Bass | Full 4-part |

Each voice is independently simplified using the Single-Staff Level 1–5 rules, with per-voice sub-level controls:

| Level | Soprano | Alto | Tenor | Bass |
|:-----:|---------|------|-------|------|
| **1** | L1–5 (default L4) | MUTE | MUTE | L1–5 (default L1) |
| **2** | L2–5 (default L4) | Fixed L2 | MUTE | L2–5 (default L2) |
| **3** | L2–5 (default L5) | MUTE | Fixed L2 | L2–5 (default L2) |
| **4** | L2–5 (default L4) | Fixed L2 | Fixed L2 | L2–5 (default L2) |
| **5** | L4–5 (default L5) | Fixed L4 | Fixed L4 | L4–5 (default L5) |

---

## Quick Start

### Installation

```bash
pip install -r requirements.txt
```

### Run

```bash
python app.py
# or: python app.py --port 8080
```

On first launch, the model is automatically downloaded from ModelScope and cached at `models/midibert-piano/`.

Access the application at: **http://localhost:7860**

### Manual Model Setup (Optional)

If auto-download fails, place these files manually:
1. `pretrain_model.ckpt` (~1.27 GB) → `midibert-piano/`
2. `CP.pkl` → `midibert-piano/`
3. `melody_best.ckpt` (optional) → `midibert-piano/`

---

## How It Works

```
MusicXML Input
     ↓
┌─────────────────────────────────────────┐
│  Layer 1: PARSING (music21)             │
│  Clef, key, time sig, pitch, duration,  │
│  rest, beam, slur/tie, ornament, etc.   │
└──────────────────┬──────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│  Layer 2: UNDERSTANDING (AI Model)      │
│  Voice classification, melody contour,  │
│  importance scoring, rhythm patterns    │
└──────────────────┬──────────────────────┘
                   ↓
┌─────────────────────────────────────────┐
│  Layer 3: DECISION (Rule Engine)        │
│  Level-specific rules + LOCKED overlay  │
│  + AI turning-point priority freeze     │
└──────────────────┬──────────────────────┘
                   ↓
         MusicXML Output (.musicxml)
```

**Three-way collaboration for voice detection**:
1. **music21** — Pitch-range & staff position analysis
2. **AI Model** — Melody/accompaniment probability
3. **Rules** — Voice density & chord-density heuristics

---

## Supported Formats

| Input | Output |
|-------|--------|
| `.mxl` | `.musicxml` |
| `.musicxml` | |
| `.mid` / `.midi` | |

Output files are fully compatible with [MuseScore](https://musescore.org/).

---

## Project Structure

```
MuseSimplifier-ModelScope01/
├── app.py                          # Gradio main application
├── config.py                       # Configuration
├── requirements.txt                # Dependencies
│
├── backend/
│   ├── score_processor.py          # Pipeline orchestrator
│   ├── score_parser.py             # MusicXML/MIDI → music21 Score → tokens
│   ├── voice_separator.py          # SATB voice separation
│   │
│   ├── engines/
│   │   ├── single_staff_engine.py  # Level 1–5 single-staff rules
│   │   ├── grand_staff_engine.py   # Level 1–5 grand-staff rules
│   │   ├── locked_protector.py     # LOCKED note detection (5 types)
│   │   └── square_structure_analyzer.py  # Phrase boundary detection
│   │
│   ├── exporter/
│   │   ├── xml_exporter.py         # MusicXML export
│   │   ├── score_builder.py        # Score builder
│   │   └── voice_aligner.py        # Voice alignment
│   │
│   └── knowledge/                  # Music theory knowledge base
│       ├── clefs.py                # 6 clef definitions
│       ├── key_signatures.py       # Key signatures
│       ├── strong_beats.py         # Strong beat positions per time sig
│       ├── time_signatures.py      # Time signature classifications
│       └── voice_ranges.py         # SATB MIDI pitch ranges
│
└── midibert-piano/                 # Pre-trained models (auto-downloaded)
```

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Web UI | Gradio 6.2.0 |
| Music Parsing | music21 9.x |
| Deep Learning | PyTorch 2.1 |
| Parallelism | ThreadPoolExecutor |
| Deployment | ModelScope (free CPU) |

---

## Acknowledgments

- **music21** — [MIT music21](https://web.mit.edu/music21/) — Music notation toolkit
- **Gradio** — [gradio.app](https://www.gradio.app/) — Web UI framework
- **ModelScope** — [modelscope.cn](https://modelscope.cn/) — Cloud deployment platform

---

**Author**: Jeffrey Zhou

**Version**: v2.1 | **License**: MIT

---

---

<a id="中文"></a>

## 🎼 AI 乐谱智能分级简化器

**在线体验**: [ModelScope 试用](https://www.modelscope.cn/studios/JeffreyZhou2026/AI-Music-Score-Simplifier-13)

本应用为乐器初学者提供**智能分级简化**功能，让学习者可以选择适合自身水平的简化乐谱，避免因乐曲难度过高而产生挫败感，最终逐步过渡到原始乐谱。

> **核心理念**：简化是*保留音高、删减/合并/延长音符*，而非从零生成。

---

## 主要功能

- **5 级简化** — 从骨架轮廓到近乎原谱
- **双模式** — 单行谱（小提琴、长笛等）+ 大谱表（钢琴、管风琴等）
- **声部自动检测** — SB/SAB/STB/SATB 四种结构，支持手动修正
- **LOCKED 保护** — 5 类结构重要音符冻结不被修改
- **AI 辅助分析** — 预训练模型提供旋律检测、重要性评分、转折点识别
- **零样本推理** — 无需训练，直接使用预训练模型
- **元数据保留** — 标题、作曲家、速度、调号、拍号原样保留
- **英文界面** — 全英文 UI 及级别描述
- **进度追踪** — 实时进度条
- **MusicXML 导出** — 兼容 MuseScore 等制谱软件

---

## 简化级别

### 单行谱模式（单音乐器）

| 级别 | 名称 | 规则 | 示例（4/4拍） |
|:----:|------|------|----------------|
| **1** | 骨架 | 每小节只保留第 1 个音，时值延至全小节 | 每小节 1 个全音符 |
| **2** | 强拍 | 只保留强拍音，时值延至下一强拍 | 第 1、3 拍保留 |
| **3** | 拍头 | 保留每拍第一个音，时值延至下一保留音 | 第 1、2、3、4 拍保留 |
| **4** | 保留节奏 | 移除装饰音；≤16 分音合并为 8 分；LOCKED 生效 | 保留所有 ≥8 分音符 |
| **5** | 近乎原谱 | 仅移除装饰音/华彩 | 接近原谱 |

**强拍定义**：2/4、3/4 → 第 1 拍；4/4 → 第 1、3 拍；6/8 → 第 1、4 拍。

**LOCKED 保护音类型**（不被简化）：
1. 切分音（弱拍起 + 连线跨强拍）
2. 附点节奏主音（≥八分时值）
3. 跨拍/跨小节连线音
4. 旋律轮廓转折点
5. 乐句/终止落点音

### 大谱表模式（复音乐器）

系统自动检测声部结构类型，支持手动修正：

| 类型 | 声部 | 说明 |
|------|------|------|
| **SB** | 女高 + 男低 | 旋律 + 简单伴奏 |
| **SAB** | 女高 + 女中 + 男低 | 三声部 |
| **STB** | 女高 + 男高 + 男低 | 三声部 |
| **SATB** | 女高 + 女中 + 男高 + 男低 | 四声部 |

每个声部独立应用单行谱 Level 1–5 规则，并提供逐声部子级别控制。

---

## 快速开始

### 安装

```bash
pip install -r requirements.txt
```

### 运行

```bash
python app.py
```

首次运行会自动从 ModelScope 下载模型，缓存至 `models/midibert-piano/`。

访问地址：**http://localhost:7860**

---

## 工作原理

```
MusicXML 输入
     ↓
Layer 1: 解析（music21）— 谱号、调号、拍号、音高、时值等
     ↓
Layer 2: 理解（AI 模型）— 声部分类、旋律轮廓、重要性评分
     ↓
Layer 3: 决策（规则引擎）— 分级规则 + LOCKED 保护 + AI 转折点优先冻结
     ↓
MusicXML 输出
```

**声部检测三方协作**：music21 音高分析 + AI 旋律概率 + 规则密度启发式

---

## 支持格式

| 输入 | 输出 |
|------|------|
| `.mxl` / `.musicxml` / `.mid` | `.musicxml` |

输出文件完全兼容 [MuseScore](https://musescore.org/)。

---

## 技术栈

| 组件 | 技术 |
|------|------|
| Web UI | Gradio 6.2.0 |
| 乐谱解析 | music21 9.x |
| 深度学习 | PyTorch 2.1 |
| 并行处理 | ThreadPoolExecutor |
| 部署平台 | ModelScope（免费 CPU） |

---

## 致谢

- **music21** — MIT 音乐记谱工具包
- **Gradio** — Web UI 框架
- **ModelScope** — 云端部署平台

---

**作者**: Jeffrey Zhou

**版本**: v2.1 | **许可**: MIT
