# AI Music Score Simplifier

[English](#english) | [中文](#中文)

---

<a id="english"></a>

## Intelligent Graded Music Score Simplification

**Online demo**: [ModelScope Studio](https://www.modelscope.cn/studios/JeffreyZhou2026/AI-Music-Score-Simplifier-13) https://www.modelscope.cn/studios/JeffreyZhou2026/AI-Music-Score-Simplifier-13


[![Python 3.10](https://img.shields.io/badge/Python-3.10-green)](https://www.python.org/)
[![Gradio 6.2](https://img.shields.io/badge/Gradio-6.2.0-orange)](https://www.gradio.app/)
[![PyTorch 2.1](https://img.shields.io/badge/PyTorch-2.1-red)](https://pytorch.org/)
[![ModelScope](https://img.shields.io/badge/Deploy-ModelScope-blue)](https://www.modelscope.cn/)
[![License](https://img.shields.io/badge/License-MIT-blue)](LICENSE)

AI Music Score Simplifier converts complex scores into graded simplified versions for music learners. It supports both monophonic single-staff scores, such as violin or flute music, and polyphonic grand-staff scores, such as piano music.

The system does not generate a new composition. Its core rule is:

> Keep original pitches, simplify rhythm and texture by removing, merging, muting, or extending existing notes.
---

<a id="english"></a>

---

## Key Features

- **Five simplification levels** from skeleton outline to near-original fidelity
- **Single-Staff mode** for monophonic instruments
- **Grand-Staff mode** for piano-style polyphonic scores
- **SATB-aware voice handling** for Soprano, Alto, Tenor, and Bass lines
- **Dynamic voice-level controls** in Grand-Staff mode
- **LOCKED note protection** for structurally important notes
- **MidiBERT-Piano assisted semantic analysis**
- **Zero-shot inference** with pretrained models, no dataset labeling or training required
- **music21-based deterministic parsing** for notation-level elements
- **MusicXML export** compatible with MuseScore and other notation software
- **ModelScope deployment** designed for free CPU instances

---

## Supported Inputs and Outputs

| Input            | Output      |
| ---------------- | ----------- |
| `.mxl`           | `.musicxml` |
| `.musicxml`      | `.musicxml` |
| `.mid` / `.midi` | `.musicxml` |

The exported score preserves notation metadata such as title, composer, tempo markings, clefs, key signatures, time signatures, and text annotations whenever possible.

---

## Simplification Algorithm

The application uses a layered pipeline:

```text
MusicXML / MIDI input
        |
        v
music21 parsing
        |
        |-- deterministic notation elements
        |   clef, key signature, time signature, pitch, duration,
        |   rests, ties, slurs, ornaments, tuplets, measures
        |
        v
MidiBERT-Piano inference
        |
        |-- semantic signals
        |   melody/accompaniment probability, importance score,
        |   contour turning points, voice-separation cues
        |
        v
RuleEngine
        |
        |-- level-specific simplification rules
        |-- LOCKED note protection
        |-- anacrusis handling
        |-- SATB voice strategy
        |
        v
music21 reconstruction and MusicXML export
```

The architecture deliberately separates **symbolic parsing**, **AI-assisted understanding**, and **rule-based rewriting**.

### Architecture Pipeline

```text
┌─────────────────────────────────────────────────────────┐
│  Frontend: Gradio 6.2.0 Web UI                          │
│  - file upload/download (.mxl / .musicxml)              │
│  - score type: Single-Staff / Grand-Staff               │
│  - simplification level selector: Level 1-5             │
│  - Grand-Staff mode: dynamic SATB voice-level buttons   │
│  - progress bar and notification system                 │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│  Parsing: music21 symbolic score processing             │
│  - score_parser.py: MusicXML parsing                    │
│  - voice_separator.py: voice separation                 │
│    based on staff position, pitch range, and AI signals │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│  Understanding: MidiBERT-Piano semantic analysis        │
│  - midibert_engine.py: model inference engine           │
│  - melody/accompaniment separation                      │
│  - zero-shot importance scoring                         │
│    using L2 norm and local variance                     │
│  - melodic contour turning-point detection              │
│  - SATB voice probability cues                          │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│  Decision: RuleEngine                                   │
│  - single_staff_engine.py: Single-Staff L1-L5           │
│  - grand_staff_engine.py: Grand-Staff L1-L5             │
│  - locked_protector.py: LOCKED note protection          │
│  - anacrusis_handler.py: pickup-measure handling        │
│  - square_structure_analyzer.py: phrase structure       │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│  Export: MusicXML reconstruction                        │
│  - score_builder.py: rebuild score object               │
│  - xml_exporter.py: generate MusicXML                   │
│  - voice_aligner.py: align multiple voices              │
└─────────────────────────────────────────────────────────┘
```

### Data Flow

```text
Input MusicXML
  -> music21.parse()
  -> parse low-level musical elements and convert to score objects/tokens
  -> MidiBERT-Piano inference
  -> extract embeddings and semantic signals
  -> zero-shot importance scoring with L2 norm and local variance
  -> apply graded RuleEngine strategy
  -> reconstruct with music21
  -> export MusicXML
```

### What music21 Does

music21 is responsible for deterministic low-level score parsing:

- staff structure detection
- note and rest extraction
- pitch, duration, dot, tuplet, beam, and measure information
- clef, key signature, time signature, and tempo metadata
- ties, slurs, ornaments, and phrase-related notation
- strong-beat positions by time signature

### What MidiBERT-Piano Does

MidiBERT-Piano is used as a semantic sensor, not as a score generator.

It helps estimate:

- melody and accompaniment likelihood
- note importance by embedding magnitude and local variance
- melodic contour turning points
- voice-separation cues for Soprano, Alto, Tenor, and Bass mapping

It does not:

- parse clefs, key signatures, or time signatures
- produce final SATB labels by itself
- rewrite the score directly
- generate new notes or new musical material

### MidiBERT-Piano Role Boundary

MidiBERT-Piano is an **understanding model** in this project. Its boundary is intentionally narrow:

| Role                             | Included? | Description                                                  |
| -------------------------------- | --------- | ------------------------------------------------------------ |
| Melody/accompaniment probability | Yes       | Provides contextual clues for melody detection and accompaniment separation. |
| Importance scoring               | Yes       | Uses hidden representations, L2 norm, and local variance as salience signals. |
| Voice-separation signal          | Yes       | Assists SATB mapping together with pitch range and staff-position rules. |
| Low-level notation parsing       | No        | Clefs, time signatures, key signatures, rests, ties, and ornaments are parsed by music21. |
| Final SATB labeling              | No        | Final labels come from music21 analysis + MidiBERT cues + rule mapping. |
| Score rewriting                  | No        | All simplification operations are executed by RuleEngine.    |
| New note generation              | No        | The app only transforms existing score material.             |

### What RuleEngine Does

RuleEngine is the final decision layer. It applies the selected simplification level, protects important notes, mutes or keeps voices in Grand-Staff mode, extends note durations, and rebuilds a readable score.

---

## Pretrained Models

The project uses **MidiBERT-Piano** pretrained checkpoints for zero-shot inference.

### Model Files

On first launch, the application downloads model assets from ModelScope Hub and caches them locally:

```text
models/JeffreyZhou2026/midibert-piano/
├── pretrain_model.ckpt    # MidiBERT-Piano pretrained checkpoint, about 1.24 GB
├── melody_best.ckpt       # optional melody extraction fine-tuned checkpoint, about 1.24 GB
└── CP.pkl                 # CP token dictionary, about 100 KB
```

Depending on the deployment package, compatible local folders such as `midibert-piano/` may also be used.

### Inference Mode

- **Zero-shot**: the app uses pretrained checkpoints directly.
- **No training**: no backpropagation, no optimizer, and no labeled user dataset are required.
- **CPU friendly**: designed for ModelScope free CPU deployment.
- **Lazy loading**: the model is loaded only when inference is needed.
- **Optional INT8 quantization**: available as a CPU memory and speed optimization.
- **Segmented processing**: long scores can be split into smaller chunks for inference.

### Why a Pretrained Model Is Used

Rule-based simplification can reliably process rhythm and notation, but it cannot always identify musical salience. MidiBERT-Piano adds contextual signals that help the system avoid deleting notes that carry melodic direction, phrase closure, or important voice-leading information.

The model therefore improves musical judgement while the rules still guarantee predictable educational simplification.

---

## Single-Staff Simplification Levels

Single-Staff mode is designed for monophonic instruments.

| Level | Name             | Algorithm                                                    |
| :---: | ---------------- | ------------------------------------------------------------ |
| **1** | Skeleton         | Keep only the first note of each measure and extend it to fill the measure. Anacrusis measures are preserved. |
| **2** | Strong Beats     | Keep notes on strong beats and extend each kept note to the next strong beat. |
| **3** | Beat Heads       | Keep the first note of each beat and remove intra-beat subdivisions. |
| **4** | Rhythm Preserved | Remove ornaments, normalize 16th/32nd notes to eighth-note level, and apply LOCKED protection. |
| **5** | Near Original    | Remove ornaments only and preserve the rest of the score.    |

### Strong-Beat Map

| Time Signature | Strong Beats                                |
| -------------- | ------------------------------------------- |
| 2/4            | beat 1                                      |
| 3/4            | beat 1                                      |
| 4/4            | beats 1 and 3                               |
| 6/8            | beats 1 and 4, counted by eighth-note pulse |

### LOCKED Note Protection

LOCKED notes are preserved even when the selected level would normally simplify them.

| LOCKED Type                     | Detection Rule                               | Protection          |
| ------------------------------- | -------------------------------------------- | ------------------- |
| Syncopation                     | weak-beat start extending into a strong beat | keep rhythm and tie |
| Dotted rhythm main note         | dotted eighth or longer                      | keep dotted pattern |
| Cross-beat or cross-measure tie | tie crosses beat or measure boundary         | keep tie relation   |
| Melodic contour turning point   | local high/low point with a significant leap | keep turning point  |
| Phrase or cadence endpoint      | note before a long rest or phrase boundary   | keep endpoint       |

Anacrusis measures are fully marked as LOCKED and preserved.

---

## Grand-Staff Simplification Levels

Grand-Staff mode is designed for polyphonic instruments such as piano.

The system separates the score into SATB-style lines:

| Voice   | Typical Range | Function                          |
| ------- | ------------- | --------------------------------- |
| Soprano | C4-C6         | main melody or upper counterpoint |
| Alto    | G3-A5         | inner harmony                     |
| Tenor   | C3-G4         | inner harmony                     |
| Bass    | E1-E3         | bass line and harmonic root       |

Voice separation combines:

- music21 pitch range and staff-position analysis
- MidiBERT-Piano melody/accompaniment probability
- density and chord-distribution heuristics

### SATB Processing Matrix

| Level | Soprano                      | Alto                        | Tenor                       | Bass             | Texture            |
| :---: | ---------------------------- | --------------------------- | --------------------------- | ---------------- | ------------------ |
| **1** | selectable L1-L5, default L4 | mute                        | mute                        | selectable L1-L5 | S+B                |
| **2** | selectable L2-L5, default L4 | fixed strong-beat treatment | mute                        | selectable L2-L5 | S+A+B              |
| **3** | selectable L2-L5, default L5 | mute                        | fixed strong-beat treatment | selectable L2-L5 | S+T+B              |
| **4** | selectable L2-L5, default L4 | fixed strong-beat treatment | fixed strong-beat treatment | selectable L2-L5 | SATB               |
| **5** | selectable L4-L5             | fixed L4                    | fixed L4                    | selectable L4-L5 | near-original SATB |

Only voices explicitly marked as selectable are exposed as user controls. Fixed and muted voices are handled automatically.

---

## Architecture

```text
MuseSimplifier-ModelScope01/
├── app.py
├── requirements.txt
├── backend/
│   ├── score_processor.py
│   ├── score_parser.py
│   ├── voice_separator.py
│   ├── midibert_engine.py
│   ├── melody_model_loader.py
│   ├── engines/
│   │   ├── single_staff_engine.py
│   │   ├── grand_staff_engine.py
│   │   ├── locked_protector.py
│   │   ├── anacrusis_handler.py
│   │   └── square_structure_analyzer.py
│   └── exporter/
│       ├── score_builder.py
│       ├── xml_exporter.py
│       └── voice_aligner.py
├── locales/
│   ├── en.json
│   ├── zh.json
│   └── ja.json
└── DefaultVoice_Child/
```

---

## Installation

```bash
pip install -r requirements.txt
```

Core dependencies:

```text
gradio==6.2.0
music21
torch==2.1.0
transformers==4.36.0
numpy
miditoolkit==0.1.14
typing-extensions>=4.12.2
packaging>=23.1
```

---

## Run Locally

```bash
python app.py
```

Optional custom port:

```bash
python app.py --port 8080
```

Default local URL:

```text
http://localhost:7860
```

The pretrained model files are downloaded automatically on first use.

---

## Deployment Notes

The project is designed for ModelScope free CPU instances:

| Item     | Setting                                                |
| -------- | ------------------------------------------------------ |
| Platform | ModelScope                                             |
| Resource | free CPU                                               |
| Instance | 2 vCPU / 16 GB / 1 instance                            |
| Image    | ubuntu22.04-py310-torch2.1.0-tf2.14.0-modelscope1.10.0 |
| Queue    | `app.queue(default_concurrency_limit=1, max_size=20)`  |

CPU optimizations include:

- `ThreadPoolExecutor(max_workers=2)` for voice-level parallelism
- lazy model loading
- optional INT8 quantization
- segmented inference for long scores
- explicit memory cleanup with `gc.collect()`

---

## Testing Checklist

| Scenario         | Mode         | Level | Expected Result                                       |
| ---------------- | ------------ | ----- | ----------------------------------------------------- |
| Jasmine Flower   | Single-Staff | L3    | keep beat heads and remove subdivisions               |
| Jasmine Flower   | Single-Staff | L4    | protect syncopation, dotted notes, and turning points |
| Turkish March    | Grand-Staff  | L2    | output Soprano, Alto, and Bass texture                |
| Turkish March    | Grand-Staff  | L4    | output SATB texture with dynamic voice controls       |
| Anacrusis score  | Single-Staff | L4    | preserve pickup measure completely                    |
| Ornamented score | Single-Staff | L4    | remove or merge ornaments                             |
| Any score        | Single-Staff | any   | hide SATB voice controls                              |

---

## Acknowledgments

- [music21](https://web.mit.edu/music21/) for symbolic music parsing
- [MidiBERT-Piano](https://github.com/wazenmai/MIDI-BERT) for pretrained symbolic-music representation learning
- [Gradio](https://www.gradio.app/) for the web interface
- [ModelScope](https://www.modelscope.cn/) for cloud deployment and model hosting

---

**Author**: Jeffrey Zhou  
**Document version**: v2.0  
**Last updated**: 2026-04-06  
**License**: MIT

---

<a id="中文"></a>

# AI 音乐乐谱简化器

## 面向音乐教育的智能分级简化系统

**在线体验**：[ModelScope Studio](https://www.modelscope.cn/studios/JeffreyZhou2026/AI-Music-Score-Simplifier-13)

AI 音乐乐谱简化器用于将复杂乐谱转换为适合不同学习阶段的分级简化版本。系统同时支持单音乐器的单行谱，以及钢琴等复音乐器的大谱表。

本项目的核心原则是：

> 简化不是重新作曲，而是在保留原始音高的基础上，对已有音符进行删减、合并、静音或时值延长。

---

## 主要功能

- **五级简化**：从骨架轮廓到接近原谱
- **Single-Staff 单行谱模式**：适合小提琴、长笛等单音乐器
- **Grand-Staff 大谱表模式**：适合钢琴等复音乐器
- **SATB 声部感知**：支持 Soprano、Alto、Tenor、Bass 分层处理
- **动态声部级别按钮**：根据大谱表 Level 自动显示可选声部
- **LOCKED 结构重要音保护**
- **MidiBERT-Piano 预训练模型辅助语义分析**
- **零样本推理**：无需训练、无需标注数据
- **music21 确定性解析**：解析谱号、拍号、调号、音高、时值等底层元素
- **MusicXML 导出**：兼容 MuseScore 等制谱软件
- **ModelScope CPU 部署友好**

---

## 输入与输出

| 输入             | 输出        |
| ---------------- | ----------- |
| `.mxl`           | `.musicxml` |
| `.musicxml`      | `.musicxml` |
| `.mid` / `.midi` | `.musicxml` |

导出时尽量保留标题、作曲家、速度标记、谱号、调号、拍号和文本注释等元数据。

---

## 核心算法

系统采用分层式算法管线：

```text
MusicXML / MIDI 输入
        |
        v
music21 解析
        |
        |-- 确定性低级音乐元素
        |   谱号、调号、拍号、音高、时值、休止符、
        |   连线、圆滑线、装饰音、三连音、小节结构
        |
        v
MidiBERT-Piano 推理
        |
        |-- 高级音乐语义信号
        |   旋律/伴奏概率、重要性评分、
        |   旋律轮廓转折点、声部分离线索
        |
        v
RuleEngine 规则引擎
        |
        |-- 分级简化规则
        |-- LOCKED 结构重要音保护
        |-- 弱起小节处理
        |-- SATB 声部策略
        |
        v
music21 重构并导出 MusicXML
```

系统将 **符号解析**、**AI 语义理解**、**规则化改写** 三个环节明确拆开。

### 架构 Pipeline 结构图

```text
┌─────────────────────────────────────────────────────────┐
│  前端层: Gradio 6.2.0 Web 界面                           │
│  - 文件上传/下载（.mxl / .musicxml）                     │
│  - 乐谱类型选择（Single-Staff / Grand-Staff）            │
│  - 简化级别选择器（Level 1-5）                           │
│  - Grand-Staff 模式：SATB 声部级别按钮（动态）           │
│  - 进度条实时反馈与通知系统                              │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│  解析层: music21 符号音乐处理                            │
│  - score_parser.py: MusicXML 解析                       │
│  - voice_separator.py: 声部分离                          │
│    基于谱表位置、音域与 AI 信号                          │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│  理解层: MidiBERT-Piano 语义理解                         │
│  - midibert_engine.py: 模型推理引擎                      │
│  - 旋律/伴奏声部分离                                     │
│  - 零样本重要性打分（L2 范数 + 局部方差）                │
│  - 旋律线轮廓转折点识别                                  │
│  - SATB 声部概率线索                                     │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│  决策层: RuleEngine 规则引擎                              │
│  - single_staff_engine.py: 单行谱 L1-L5                 │
│  - grand_staff_engine.py: 大谱表 L1-L5                  │
│  - locked_protector.py: LOCKED 音符保护                  │
│  - anacrusis_handler.py: 弱起小节处理                    │
│  - square_structure_analyzer.py: 方整性结构分析          │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│  导出层: MusicXML 重构                                   │
│  - score_builder.py: 乐谱对象重建                        │
│  - xml_exporter.py: MusicXML 文件生成                    │
│  - voice_aligner.py: 多声部对齐                          │
└─────────────────────────────────────────────────────────┘
```

### 数据流

```text
输入 MusicXML
  -> music21.parse()
  -> 解析 A 类低级音乐元素并转换为 score 对象 / token
  -> MidiBERT-Piano 推理
  -> 提取 embedding 与高级音乐语义信号
  -> 零样本重要性打分（L2 范数 + 局部方差）
  -> RuleEngine 应用分级策略
  -> music21 重构
  -> 导出 MusicXML
```

### music21 负责什么

music21 负责 100% 确定性的低级音乐元素解析：

- 谱表结构
- 音符与休止符
- 音高、时值、附点、三连音、符尾、小节信息
- 谱号、调号、拍号、速度标记
- 延音连线、连奏线、装饰音
- 按拍号计算强拍位置

### MidiBERT-Piano 负责什么

MidiBERT-Piano 在本项目中是“理解型传感器”，不是生成器。

它辅助判断：

- 旋律与伴奏概率
- 基于 embedding L2 范数和局部方差的音符重要性
- 旋律线轮廓转折点
- Soprano / Alto / Tenor / Bass 声部分离线索

它不负责：

- 解析谱号、调号、拍号等底层符号
- 独立输出最终 SATB 标签
- 直接改写乐谱
- 生成新的音符或音乐材料

### MidiBERT-Piano 角色边界

MidiBERT-Piano 是本系统的**理解型模型**，也可以理解为**语义感知传感器**。它的职责边界如下：

| 角色           | 是否负责 | 说明                                                         |
| -------------- | -------- | ------------------------------------------------------------ |
| 旋律/伴奏概率  | 是       | 提供旋律检测与伴奏分离的上下文线索。                         |
| 重要性评分     | 是       | 基于隐藏表示、L2 范数和局部方差提供显著性信号。              |
| 声部分离信号   | 是       | 与音域、谱表位置规则共同辅助 SATB 映射。                     |
| 低级记谱解析   | 否       | 谱号、拍号、调号、休止符、连线、装饰音等由 music21 解析。    |
| 最终 SATB 标签 | 否       | 最终标签由 music21 分析 + MidiBERT 线索 + 规则映射共同决定。 |
| 乐谱改写       | 否       | 所有简化操作由 RuleEngine 执行。                             |
| 新音符生成     | 否       | 系统只转换已有乐谱材料，不生成新音乐。                       |

### RuleEngine 负责什么

RuleEngine 是最终决策层。它根据用户选择的简化级别执行保留、删除、合并、静音、延长时值等操作，并结合 LOCKED 保护和 SATB 声部策略重建可读乐谱。

---

## 预训练模型

本项目使用 **MidiBERT-Piano** 预训练模型进行零样本推理。

### 模型文件

首次运行时，程序会从 ModelScope Hub 自动下载模型资产，并缓存到本地：

```text
models/JeffreyZhou2026/midibert-piano/
├── pretrain_model.ckpt    # MidiBERT-Piano 预训练权重，约 1.24 GB
├── melody_best.ckpt       # 可选旋律提取微调权重，约 1.24 GB
└── CP.pkl                 # CP token 字典，约 100 KB
```

根据部署包结构，也可能兼容使用 `midibert-piano/` 等本地模型目录。

### 推理方式

- **零样本推理**：直接使用预训练模型。
- **无需训练**：不进行反向传播，不需要优化器，不依赖用户标注数据。
- **CPU 友好**：面向 ModelScope 免费 CPU 环境设计。
- **延迟加载**：只有在需要推理时才加载模型。
- **可选 INT8 量化**：用于降低 CPU 推理内存占用并提升速度。
- **长谱分段处理**：长乐谱可按片段进行推理。

### 为什么需要预训练模型

纯规则系统能够稳定处理时值、拍点和记谱结构，但难以判断某个音是否具有旋律方向、乐句收束或声部进行意义。

MidiBERT-Piano 提供上下文语义信号，帮助系统避免删除旋律转折点、终止落点和重要声部进行音。最终是否保留、合并或删除，仍由 RuleEngine 执行，从而兼顾音乐理解和规则可控性。

---

## 单行谱简化规则

Single-Staff 模式适用于单音乐器。

| Level | 名称                      | 算法规则                                                     |
| :---: | ------------------------- | ------------------------------------------------------------ |
| **1** | Skeleton 骨架             | 每小节只保留第一个音符，并将其时值延长至整小节；弱起小节保持原样。 |
| **2** | Strong Beats 强拍         | 只保留强拍位置音符，并延长到下一强拍。                       |
| **3** | Beat Heads 拍头           | 保留每拍的第一个音符，删除拍内细分音。                       |
| **4** | Rhythm Preserved 保留节奏 | 移除装饰音，将十六分/三十二分正规化到八分层级，并启用 LOCKED 保护。 |
| **5** | Near Original 近乎原谱    | 仅移除装饰音，其余尽量保留。                                 |

### 强拍定义

| 拍号 | 强拍位置                     |
| ---- | ---------------------------- |
| 2/4  | 第 1 拍                      |
| 3/4  | 第 1 拍                      |
| 4/4  | 第 1、3 拍                   |
| 6/8  | 第 1、4 拍，以八分音符为一拍 |

### LOCKED 结构重要音

被标记为 LOCKED 的音符，即使当前 Level 通常会简化它，也会被保留。

| 类型            | 识别标准                             | 保护内容         |
| --------------- | ------------------------------------ | ---------------- |
| 切分音          | 弱拍或后半拍起音，并延续到下一个强拍 | 保留节奏型和连线 |
| 附点节奏主音    | 附点八分或更长时值                   | 保留附点模式     |
| 跨拍/跨小节连线 | 延音连线跨越拍点或小节线             | 保留连线关系     |
| 旋律轮廓转折点  | 局部最高/最低点，且存在明显跳进      | 保留转折点       |
| 乐句/终止落点   | 后接较长休止或处于乐句边界           | 保留终止音       |

弱起小节会整体标记为 LOCKED，并完整保留。

---

## 大谱表简化规则

Grand-Staff 模式适用于钢琴等复音乐器。

系统将大谱表拆解为 SATB 风格声部：

| 声部    | 典型音域 | 功能             |
| ------- | -------- | ---------------- |
| Soprano | C4-C6    | 主旋律或高音对位 |
| Alto    | G3-A5    | 内声部和声填充   |
| Tenor   | C3-G4    | 内声部和声填充   |
| Bass    | E1-E3    | 低音线与和声根音 |

声部分离由三类信息共同完成：

- music21 的音高范围与谱表位置分析
- MidiBERT-Piano 的旋律/伴奏概率
- 声部密度与和弦密度启发式规则

### SATB 处理矩阵

| Level | Soprano             | Alto         | Tenor        | Bass       | 织体          |
| :---: | ------------------- | ------------ | ------------ | ---------- | ------------- |
| **1** | 可选 L1-L5，默认 L4 | 静音         | 静音         | 可选 L1-L5 | S+B           |
| **2** | 可选 L2-L5，默认 L4 | 固定强拍处理 | 静音         | 可选 L2-L5 | S+A+B         |
| **3** | 可选 L2-L5，默认 L5 | 静音         | 固定强拍处理 | 可选 L2-L5 | S+T+B         |
| **4** | 可选 L2-L5，默认 L4 | 固定强拍处理 | 固定强拍处理 | 可选 L2-L5 | SATB          |
| **5** | 可选 L4-L5          | 固定 L4      | 固定 L4      | 可选 L4-L5 | 接近原谱 SATB |

只有蓝图中明确标记为“可选”的声部才在界面中显示按钮。固定处理和静音声部由系统自动处理。

---

## 技术架构

```text
MuseSimplifier-ModelScope01/
├── app.py                          # Gradio 主程序
├── requirements.txt                # Python 依赖
├── backend/
│   ├── score_processor.py          # 总协调器
│   ├── score_parser.py             # MusicXML/MIDI 解析
│   ├── voice_separator.py          # 声部分离
│   ├── midibert_engine.py          # MidiBERT 推理引擎
│   ├── melody_model_loader.py      # 旋律模型加载
│   ├── engines/
│   │   ├── single_staff_engine.py  # 单行谱 L1-L5 规则
│   │   ├── grand_staff_engine.py   # 大谱表 L1-L5 规则
│   │   ├── locked_protector.py     # LOCKED 保护
│   │   ├── anacrusis_handler.py    # 弱起小节处理
│   │   └── square_structure_analyzer.py
│   └── exporter/
│       ├── score_builder.py        # 乐谱对象重建
│       ├── xml_exporter.py         # MusicXML 导出
│       └── voice_aligner.py        # 多声部对齐
├── locales/
│   ├── en.json
│   ├── zh.json
│   └── ja.json
└── DefaultVoice_Child/
```

---

## 安装

```bash
pip install -r requirements.txt
```

核心依赖：

```text
gradio==6.2.0
music21
torch==2.1.0
transformers==4.36.0
numpy
miditoolkit==0.1.14
typing-extensions>=4.12.2
packaging>=23.1
```

---

## 本地运行

```bash
python app.py
```

可指定端口：

```bash
python app.py --port 8080
```

默认访问地址：

```text
http://localhost:7860
```

首次使用时会自动下载预训练模型文件。

---

## ModelScope 部署说明

项目面向 ModelScope 免费 CPU 实例设计：

| 项目     | 配置                                                   |
| -------- | ------------------------------------------------------ |
| 部署平台 | ModelScope                                             |
| 云资源   | 免费 CPU                                               |
| 实例     | 2 vCPU / 16 GB / 1 实例                                |
| 镜像     | ubuntu22.04-py310-torch2.1.0-tf2.14.0-modelscope1.10.0 |
| 队列     | `app.queue(default_concurrency_limit=1, max_size=20)`  |

CPU 优化策略：

- 使用 `ThreadPoolExecutor(max_workers=2)` 进行声部级并行
- 模型延迟加载
- 可选 INT8 量化
- 长乐谱分段推理
- 使用 `gc.collect()` 及时清理内存

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



## ⚠️ Copyright Notice

© 2026 Jeffrey Zhou. All rights reserved.

This repository and its contents are protected by copyright law.  
No part of this project may be copied, reproduced, modified, or distributed without prior written permission from the author.

Commercial use is strictly prohibited.


*Built with ❤️ for music education*


---

<a id="中文"></a>

## 🎼 AI 乐谱智能分级简化器

**在线体验**: [ModelScope 试用](https://www.modelscope.cn/studios/JeffreyZhou2026/AI-Music-Score-Simplifier-13) https://www.modelscope.cn/studios/JeffreyZhou2026/AI-Music-Score-Simplifier-13
本应用为乐器初学者提供**智能分级简化**功能，让学习者可以选择适合自身水平的简化乐谱，避免因乐曲难度过高而产生挫败感，最终逐步过渡到原始乐谱。


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

## ⚠️ Copyright Notice

© 2026 Jeffrey Zhou. All rights reserved.

This repository and its contents are protected by copyright law.  
No part of this project may be copied, reproduced, modified, or distributed without prior written permission from the author.

Commercial use is strictly prohibited.


*Built with ❤️ for music education*

