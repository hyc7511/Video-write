# Codex 插件开发文档：视频/字幕/小说对话驱动的小说重构改写系统

版本号：v0.3.1  
日期：2026-06-13  
插件名称：`video-dialogue-novel-rewrite`  
中文名称：`视频对话驱动小说重构与章节式改写插件`

---

## 0. 本版核心要求

本插件用于把视频、字幕、小说文案、对话文本转成结构化素材，再由 Codex 当前接入的模型完成理解、世界观重建、章节式小说改写、审核和最终导出。

本版必须落实以下要求：

```text
1. 不使用本地 LLM。
   因为这是 Codex 插件，所有理解、规划、写作、审核都直接使用 Codex 当前接入的模型完成。
   禁止设计 Ollama、本地大模型、本地 RAG 推理、本地 LLM API。

2. Skill 文件和 Prompt 文件必须分开。
   Skill 负责说明“什么时候用、输入是什么、输出是什么、不允许做什么”。
   Prompt 负责具体给 Codex 的写作/理解/审核指令。

3. scripts 文件不需要一个节点一个文件。
   脚本能合并就合并，按流程分组，文件数量尽量少。
   不要创建几十个小脚本。

4. 每个 Skill 文件前部必须有中文功能说明。
   不强制一定是第一行，但必须在文件最前面几行，打开文件后一眼能看懂这个 Skill 是干什么的。
   示例：
   <!-- 功能：本技能只负责视频处理，包括音频提取、截图和切片，不负责写作、审核或导出。 -->

5. 开发完成后必须写完整使用文档。
   必须包含 README.md、CHANGELOG.md、VERSION、文件职责表、Skill/Prompt 对照表、GitHub 参考项目表。
   更新记录必须有版本号、日期、新增、变更、修复、已知问题。

6. 每次从第一步开始，必须根据“新小说初步定位名称”创建项目文件夹。
   文件夹不能直接用原小说名、原视频名、原主角名。
   示例：
   workspace/第一人称乡村逆袭爽文_田螺夜市供货/

7. 新改编小说必须以对话为主。
   对话内容必须占每章篇幅一半以上。
   只要一个段落包含人物说出口的话，整段都算对白内容。
```

---

## 1. 插件最终目标

完整工作流：

```text
视频 / 字幕 / 小说文案 / 对话文本
↓
根据新小说初步定位名称创建项目文件夹
↓
导入素材
↓
视频处理：ffmpeg 提取音频、截图、切片
↓
字幕构建：Whisper 语音转文字 / 用户字幕解析 / OCR 硬字幕补充
↓
场景切分：根据视频、时间轴、对白冲突切分场景
↓
素材包：合并字幕、画面、场景、时间轴
↓
对白理解：说话人、听话人、情绪、目的、冲突、剧情功能
↓
故事世界：人物表、地点表、道具表、世界观、利益规则
↓
章节大纲：章节/场景概要、冲突推进、每章钩子
↓
改写方案：新命名、新题材、新行业、新地点、新人物、新规则
↓
章节写作：一次只写一章，第一人称，对话占比超过 50%
↓
章节门控：写完一章总结，等待用户说“继续”
↓
审核：原文残留、相似度、逻辑、适宜性、对话占比
↓
修复：根据审核报告修复
↓
导出：最终小说、章节索引、审核摘要、使用索引
```

---

## 2. GitHub 参考项目取长补短

本插件不是照搬任何开源项目，而是参考多个项目的优点后重新设计。

| 参考项目 | 可借鉴点 | 本插件如何采用 | 不采用内容 |
|---|---|---|---|
| OpenAI Codex Plugins / Skills | `.codex-plugin/plugin.json`、`skills/<skill>/SKILL.md`、Skill 独立说明 | 采用 Codex 插件结构和 Skill 目录 | 不设计本地 LLM |
| WhatIf | 从小说提取角色、地点、事件、知识体系，形成世界包 | 设计 `material_package.json`、`story_world.json` | 不做互动小说游戏 |
| AI-Novel-Writing-Assistant | 世界观、写法控制、章节生成、Agent 工作流 | 拆成世界观、改写方案、章节写作、审核 | 不做大型 RAG 和数据库 |
| AI_NovelGenerator | 多章节生成、角色轨迹、逻辑冲突校对 | 用 `chapter_state.json` 跟踪章节状态，用 audit 审核一致性 | 不做复杂伏笔数据库 |
| Novel-OS / book-os | 文件夹式上下文、设定和正文分离 | 每本新小说一个独立 workspace | 不照搬整套写作系统 |
| Awesome Novel Studio | 多 Skill / 多阶段写作流程 | Skill 单职责、多 Skill 协作 | 不允许一个 Skill 包办多个节点 |
| autonovel | 写作 → 评估 → 修复闭环 | 章节写作后审核，再根据报告修复 | 不自动无限续写 |
| AI-Text-Rewriting-Toolbox | 轻量文本改写、Prompt 模板化、结果保存 | Prompt 独立、输出文件化 | 不做简单替换改写 |
| faster-whisper / Whisper | 音频转字幕，保留时间轴 | 只用于字幕构建 | 不用于剧情理解和写作 |
| PySceneDetect | 镜头切分、场景边界检测 | 辅助生成 `scenes.json` | 不把镜头切分等同剧情场景 |
| PaddleOCR / EasyOCR | OCR 识别硬字幕 | 作为字幕补充和校正 | 没有 OCR 不影响主流程 |

参考项目链接：

```text
https://developers.openai.com/codex/plugins/build
https://developers.openai.com/codex/skills
https://github.com/ypcypc/WhatIf
https://github.com/ExplosiveCoderflome/AI-Novel-Writing-Assistant
https://github.com/YILING0013/AI_NovelGenerator
https://github.com/forsonny/book-os
https://github.com/MJbae/awesome-novel-studio
https://github.com/NousResearch/autonovel
https://github.com/danielrosehill/AI-Text-Rewriting-Toolbox
https://github.com/SYSTRAN/faster-whisper
https://github.com/openai/whisper
https://github.com/Breakthrough/PySceneDetect
https://github.com/PaddlePaddle/PaddleOCR
https://github.com/JaidedAI/EasyOCR
```

---

## 3. 项目目录结构

请创建以下目录结构：

```text
video-dialogue-novel-rewrite/
├─ .codex-plugin/
│  └─ plugin.json
├─ skills/
│  ├─ 01-project-create/
│  │  └─ SKILL.md
│  ├─ 02-source-ingest/
│  │  └─ SKILL.md
│  ├─ 03-video-process/
│  │  └─ SKILL.md
│  ├─ 04-transcript-build/
│  │  └─ SKILL.md
│  ├─ 05-ocr-subtitle/
│  │  └─ SKILL.md
│  ├─ 06-scene-segment/
│  │  └─ SKILL.md
│  ├─ 07-material-package/
│  │  └─ SKILL.md
│  ├─ 08-dialogue-understand/
│  │  └─ SKILL.md
│  ├─ 09-story-world-build/
│  │  └─ SKILL.md
│  ├─ 10-outline-build/
│  │  └─ SKILL.md
│  ├─ 11-rewrite-plan/
│  │  └─ SKILL.md
│  ├─ 12-chapter-writing/
│  │  └─ SKILL.md
│  ├─ 13-chapter-gate/
│  │  └─ SKILL.md
│  ├─ 14-audit/
│  │  └─ SKILL.md
│  ├─ 15-final-fix/
│  │  └─ SKILL.md
│  ├─ 16-final-export/
│  │  └─ SKILL.md
│  └─ 17-reference-research/
│     └─ SKILL.md
├─ prompts/
│  ├─ dialogue_understand.md
│  ├─ story_world_build.md
│  ├─ outline_build.md
│  ├─ rewrite_plan.md
│  ├─ chapter_writing.md
│  ├─ chapter_gate.md
│  ├─ audit.md
│  └─ final_fix.md
├─ scripts/
│  ├─ __init__.py
│  ├─ cli.py
│  ├─ core_utils.py
│  ├─ media_pipeline.py
│  ├─ subtitle_pipeline.py
│  ├─ story_assets_pipeline.py
│  ├─ writing_pipeline.py
│  └─ audit_export_pipeline.py
├─ schemas/
│  ├─ project.schema.json
│  ├─ material_package.schema.json
│  ├─ dialogue_understanding.schema.json
│  ├─ story_world.schema.json
│  ├─ outline.schema.json
│  ├─ rewrite_plan.schema.json
│  ├─ chapter_state.schema.json
│  └─ audit_report.schema.json
├─ references/
│  ├─ github_reference_matrix.md
│  ├─ skill_prompt_mapping.md
│  ├─ file_responsibility_table.md
│  └─ development_notes.md
├─ examples/
│  ├─ sample_dialogue.txt
│  ├─ sample_subtitle.srt
│  └─ sample_rewrite_rules.json
├─ workspace/
│  └─ .gitkeep
├─ tests/
│  ├─ test_project_folder.py
│  ├─ test_subtitle_pipeline.py
│  ├─ test_material_package.py
│  ├─ test_chapter_state.py
│  ├─ test_dialogue_ratio.py
│  └─ test_audit.py
├─ AGENTS.md
├─ README.md
├─ CHANGELOG.md
├─ VERSION
├─ requirements.txt
├─ requirements-optional.txt
└─ pyproject.toml
```

---

## 4. Skill 文件职责

### 4.1 `skills/01-project-create/SKILL.md`

文件前部必须包含：

```markdown
<!-- 功能：本技能只负责根据新小说初步定位名称创建项目文件夹，不负责导入素材、处理视频、写作或审核。 -->
```

职责：

```text
根据新小说初步定位名称创建项目文件夹。
```

输入：

```text
新小说名 / 新小说初步定位名称 / 改写方向
```

输出：

```text
workspace/<新小说初步定位名称>/
workspace/<新小说初步定位名称>/project.json
```

文件夹命名规则：

```text
1. 文件夹名必须使用新小说初步定位名称。
2. 不使用原小说名作为最终文件夹名。
3. 不使用原视频名作为最终文件夹名。
4. 不使用原人物名作为文件夹名。
5. 去掉 Windows 不允许的字符：\ / : * ? " < > |
6. 如果用户没有给新小说名，则根据改写方向生成初步定位名。
7. 如果重名，自动追加 _002、_003。
```

---

### 4.2 `skills/02-source-ingest/SKILL.md`

文件前部必须包含：

```markdown
<!-- 功能：本技能只负责导入视频、字幕、小说文案或对话文本，不负责视频处理、理解剧情、写作或审核。 -->
```

输入：

```text
mp4 / mov / mkv / srt / vtt / txt / md / json
```

输出：

```text
00_input/
```

禁止：

```text
不处理视频
不识别字幕
不写文案
不审核
```

---

### 4.3 `skills/03-video-process/SKILL.md`

文件前部必须包含：

```markdown
<!-- 功能：本技能只负责视频处理，包括音频提取、截图和视频切片，不负责字幕识别、剧情理解、写作或审核。 -->
```

职责：

```text
ffmpeg 提取音频、截图、切片、视频信息。
```

输出：

```text
01_media/audio.wav
01_media/frames/
01_media/slices/
01_media/media_info.json
01_media/media_report.md
```

禁止：

```text
不做语音转文字
不做 OCR
不理解剧情
不写小说
```

---

### 4.4 `skills/04-transcript-build/SKILL.md`

文件前部必须包含：

```markdown
<!-- 功能：本技能只负责把音频或字幕整理成时间轴文本，不负责 OCR、剧情理解、文案写作或审核。 -->
```

职责：

```text
Whisper 语音转文字。
解析用户提供的 srt/vtt/txt。
生成统一 transcript。
```

输出：

```text
02_transcript/transcript.srt
02_transcript/transcript.json
02_transcript/transcript.md
```

---

### 4.5 `skills/05-ocr-subtitle/SKILL.md`

文件前部必须包含：

```markdown
<!-- 功能：本技能只负责 OCR 识别视频硬字幕，不负责语音转文字、剧情理解、文案写作或审核。 -->
```

输出：

```text
03_ocr/ocr_subtitles.json
03_ocr/ocr_report.md
```

---

### 4.6 `skills/06-scene-segment/SKILL.md`

文件前部必须包含：

```markdown
<!-- 功能：本技能只负责场景切分和时间段归并，不负责人物理解、世界观、文案写作或审核。 -->
```

输出：

```text
04_scenes/scenes.json
04_scenes/scenes.md
```

---

### 4.7 `skills/07-material-package/SKILL.md`

文件前部必须包含：

```markdown
<!-- 功能：本技能只负责把视频、字幕、OCR、场景整理成统一素材包，不负责剧情理解、文案写作或审核。 -->
```

输出：

```text
05_material/material_package.json
05_material/material_package.md
```

---

### 4.8 `skills/08-dialogue-understand/SKILL.md`

文件前部必须包含：

```markdown
<!-- 功能：本技能只负责对白理解，包括说话人、情绪、冲突和剧情功能，不负责世界观、章节正文或审核。 -->
```

职责：

```text
理解对白中的说话人、听话人、情绪、目的、冲突、人物关系信号、剧情功能。
```

输出：

```text
06_dialogue/dialogue_understanding_prompt.md
06_dialogue/dialogue_understanding.json
06_dialogue/dialogue_understanding.md
```

---

### 4.9 `skills/09-story-world-build/SKILL.md`

文件前部必须包含：

```markdown
<!-- 功能：本技能只负责建立人物表、地点表、道具表和世界观，不负责章节正文写作或审核。 -->
```

输出：

```text
07_world/characters.json
07_world/locations.json
07_world/items.json
07_world/story_world.json
07_world/story_world.md
```

---

### 4.10 `skills/10-outline-build/SKILL.md`

文件前部必须包含：

```markdown
<!-- 功能：本技能只负责建立章节大纲和场景概要，不负责写章节正文、审核或导出。 -->
```

输出：

```text
08_outline/outline_prompt.md
08_outline/outline.json
08_outline/outline.md
```

---

### 4.11 `skills/11-rewrite-plan/SKILL.md`

文件前部必须包含：

```markdown
<!-- 功能：本技能只负责生成改写方案和写作规则，不负责写章节正文、章节总结或审核。 -->
```

必须包含：

```text
新小说初步定位名称
新人物名
新地点名
新行业设定
第一人称规则
总字数 40000-50000
每章 1000-3000 字
第一章炸裂规则
每章钩子规则
对话占比超过 50%
弱化不适宜描写规则
禁止原文命名
禁止原文特征性台词
```

输出：

```text
09_plan/rewrite_plan_prompt.md
09_plan/rewrite_plan.json
09_plan/rewrite_plan.md
```

---

### 4.12 `skills/12-chapter-writing/SKILL.md`

文件前部必须包含：

```markdown
<!-- 功能：本技能只负责按改写方案写当前一章小说正文，不负责章节总结、审核、修复或导出。 -->
```

写作硬性要求：

```text
1. 只写当前章节。
2. 主角必须第一人称。
3. 所有新命名不得和原文一样。
4. 文字排版不能太紧凑，要像常规小说。
5. 每章 1000 到 3000 中文字。
6. 第一章开篇即高潮，直接抛核心设定和大反转。
7. 整体节奏快，爽点多。
8. 每章结尾必须有钩子。
9. 不写剧情摘要，必须写小说正文。
10. 不出现“旁白、画外音、内心OS”等影视术语。
11. 弱化血腥、暴力、恐怖、低俗、不适宜未成年人阅读的直接描写。
12. 保留情节推进、人物关系和结局走向的抽象功能，但不要照搬原文句子。
13. 对话和心理描写可以重新创作，但不能保留原文特征性台词。
14. 新改编小说必须以对话为主。
15. 对话内容必须占本章篇幅一半以上。
16. 只要一个段落包含人物说出口的话，整段都算对白内容。
```

输出：

```text
10_chapters/chapter_001.md
10_chapters/chapter_001_meta.json
```

禁止：

```text
不总结本章。
不询问继续。
不审核。
不修复最终稿。
```

---

### 4.13 `skills/13-chapter-gate/SKILL.md`

文件前部必须包含：

```markdown
<!-- 功能：本技能只负责章节总结、进度记录和等待用户继续，不负责写下一章、审核或导出。 -->
```

必须输出：

```text
本章摘要
本章爽点
本章钩子
本章对话占比
当前总字数
下一章建议方向
请回复“继续”后再写下一章
```

输出：

```text
10_chapters/chapter_001_summary.md
10_chapters/chapter_state.json
```

禁止：

```text
不自动写下一章。
```

---

### 4.14 `skills/14-audit/SKILL.md`

文件前部必须包含：

```markdown
<!-- 功能：本技能只负责审核原文残留、相似度、逻辑、内容适宜性和对话占比，不负责改写、修复或导出。 -->
```

审核项目：

```text
原文人名残留
原文地点残留
原文行业词残留
原文特征性台词残留
连续 8/10/12 字重复
是否只是换名字
是否第一人称
是否每章有钩子
是否对话占比超过 50%
是否存在过度血腥、暴力、恐怖、低俗或不适宜未成年人阅读的直接描写
是否有章节逻辑断裂
```

输出：

```text
11_audit/audit_report.json
11_audit/audit_report.md
```

---

### 4.15 `skills/15-final-fix/SKILL.md`

文件前部必须包含：

```markdown
<!-- 功能：本技能只负责根据审核报告修复章节或全书，不负责重新处理视频、重新建世界观或导出。 -->
```

输出：

```text
12_fixed/fixed_chapters/
12_fixed/fix_report.md
```

---

### 4.16 `skills/16-final-export/SKILL.md`

文件前部必须包含：

```markdown
<!-- 功能：本技能只负责最终导出小说、章节索引、使用文档索引和审核摘要，不负责写作或审核。 -->
```

输出：

```text
13_final/final_story.md
13_final/final_chapter_index.md
13_final/final_audit_summary.md
13_final/export_index.md
```

---

### 4.17 `skills/17-reference-research/SKILL.md`

文件前部必须包含：

```markdown
<!-- 功能：本技能只负责整理 GitHub 参考项目和取长补短说明，不负责处理视频、写作、审核或导出。 -->
```

输出：

```text
references/github_reference_matrix.md
```

---

## 5. Prompt 文件职责

Prompt 文件必须和 Skill 分开：

```text
prompts/dialogue_understand.md     只做对白理解
prompts/story_world_build.md       只做故事世界资产
prompts/outline_build.md           只做章节/场景概要
prompts/rewrite_plan.md            只做改写方案
prompts/chapter_writing.md         只写当前章节正文
prompts/chapter_gate.md            只做章节总结和继续门控
prompts/audit.md                   只做审核
prompts/final_fix.md               只做修复
```

---

## 6. 脚本文件职责：能合并就合并

### 6.1 `scripts/cli.py`

```text
命令行入口。
只解析命令，调用流程脚本。
```

### 6.2 `scripts/core_utils.py`

```text
路径管理
安全文件夹名
项目初始化
JSON/Markdown 读写
编码识别
依赖检查
日志
版本号读取
日期写入
```

### 6.3 `scripts/media_pipeline.py`

```text
视频处理流程：
ffprobe 读取视频信息
ffmpeg 提取音频
ffmpeg 抽帧
ffmpeg 切片
生成媒体报告
```

不负责：

```text
不做文案写作。
不做世界观。
不做审核。
```

### 6.4 `scripts/subtitle_pipeline.py`

```text
字幕流程：
解析 srt/vtt/txt
Whisper / faster-whisper 转写
OCR 字幕识别
字幕合并
生成 transcript.json
```

不负责：

```text
不写小说。
不审核小说。
```

### 6.5 `scripts/story_assets_pipeline.py`

```text
故事资产流程：
构建 material_package
生成对白理解 prompt
生成故事世界 prompt
生成大纲 prompt
生成改写方案 prompt
保存对应 JSON/MD 占位文件
```

不负责：

```text
不直接写小说正文。
不审核。
```

### 6.6 `scripts/writing_pipeline.py`

```text
写作流程：
生成当前章节写作 prompt
保存当前章节
统计章节字数
统计对话占比
更新 chapter_state.json
生成章节总结 prompt
```

不负责：

```text
不做视频处理。
不做审核修复。
```

### 6.7 `scripts/audit_export_pipeline.py`

```text
审核和导出流程：
原文残留检查
相似度粗检
对话占比检查
内容适宜性审核 prompt
最终修复 prompt
导出最终文件
生成使用文档索引
生成更新记录辅助文件
```

不负责：

```text
不做视频处理。
不写新章节正文。
```

---

## 7. 新小说项目文件夹创建规则

每次从第一步开始，必须创建新文件夹。

文件夹名来源优先级：

```text
1. 用户明确给出的新小说名。
2. rewrite_rules.json 中的 novel_positioning_name。
3. Codex 根据改写方向生成的初步定位名称。
```

禁止：

```text
不直接使用原小说名。
不直接使用原视频名。
不使用原主角名。
不使用原地点名。
```

示例：

```json
{
  "novel_positioning_name": "第一人称乡村逆袭爽文_田螺夜市供货"
}
```

对应文件夹：

```text
workspace/第一人称乡村逆袭爽文_田螺夜市供货/
```

如果重名：

```text
workspace/第一人称乡村逆袭爽文_田螺夜市供货_002/
```

---

## 8. 对话占比硬性规则

### 8.1 写作规则

```text
每一章的对话内容占比必须超过 50%。
建议目标为 55% 到 65%，避免刚好卡线。
```

### 8.2 对话内容计算规则

只要一个段落包含人物说出口的话，整段都算对白内容。

例如：

```text
我猛地抬起手，瞪着他，一字一句地说：“你今天必须把这事说清楚。”
```

这一整段全部算对白内容。

再比如：

```text
许川把账本摔在桌上，盯着老周冷声说：“三块一斤？你这是想让我白干。”
```

这一整段全部算对白内容。

没有任何人物说出口的话，才算非对白内容。

例如：

```text
院子里的人全都安静下来，只有水田边的风吹得塑料布哗哗作响。
```

这算非对白内容。

### 8.3 审核输出

审核必须输出：

```json
{
  "dialogue_ratio": 0.58,
  "dialogue_ratio_passed": true,
  "dialogue_ratio_rule": "含有引号对白或人物冒号对白的整段，全部按对白内容计算"
}
```

低于 50% 时，必须要求修复：

```text
本章对话占比不足，需要增加人物交锋、追问、反驳、威胁、辩解、揭底、反转类对白。
```

---

## 9. Workspace 目录结构

每本新小说一个独立文件夹：

```text
workspace/<新小说初步定位名称>/
├─ 00_input/
├─ 01_media/
├─ 02_transcript/
├─ 03_ocr/
├─ 04_scenes/
├─ 05_material/
├─ 06_dialogue/
├─ 07_world/
├─ 08_outline/
├─ 09_plan/
├─ 10_chapters/
├─ 11_audit/
├─ 12_fixed/
├─ 13_final/
└─ logs/
```

目录职责：

```text
00_input      原始视频、字幕、文本
01_media      视频处理结果、音频、截图、切片
02_transcript 字幕和语音转写
03_ocr        OCR 硬字幕
04_scenes     场景切分
05_material   统一剧情素材包
06_dialogue   对白理解
07_world      人物、地点、道具、世界观
08_outline    章节/场景概要
09_plan       改写方案
10_chapters   章节初稿、章节总结、进度状态
11_audit      审核报告
12_fixed      修复稿
13_final      最终导出
logs          日志
```

---

## 10. CLI 命令设计

统一入口：

```bash
python -m scripts.cli
```

### 10.1 创建新小说项目

```bash
python -m scripts.cli create-project --name "第一人称乡村逆袭爽文_田螺夜市供货"
```

### 10.2 导入素材

```bash
python -m scripts.cli ingest --project "第一人称乡村逆袭爽文_田螺夜市供货" --input input/demo.srt
```

### 10.3 视频处理

```bash
python -m scripts.cli process-video --project "第一人称乡村逆袭爽文_田螺夜市供货"
```

执行：

```text
提取音频
抽帧
切片
读取媒体信息
```

### 10.4 构建字幕

```bash
python -m scripts.cli build-transcript --project "第一人称乡村逆袭爽文_田螺夜市供货"
```

执行：

```text
解析字幕
Whisper 转写
合并字幕
```

### 10.5 OCR

```bash
python -m scripts.cli build-ocr --project "第一人称乡村逆袭爽文_田螺夜市供货"
```

### 10.6 场景切分

```bash
python -m scripts.cli segment-scenes --project "第一人称乡村逆袭爽文_田螺夜市供货"
```

### 10.7 构建素材包

```bash
python -m scripts.cli build-material --project "第一人称乡村逆袭爽文_田螺夜市供货"
```

### 10.8 构建故事资产

```bash
python -m scripts.cli build-story-assets --project "第一人称乡村逆袭爽文_田螺夜市供货"
```

生成：

```text
对白理解 prompt
世界观 prompt
大纲 prompt
改写方案 prompt
```

### 10.9 写当前章节

```bash
python -m scripts.cli write-chapter --project "第一人称乡村逆袭爽文_田螺夜市供货" --chapter 1
```

注意：

```text
只写第 1 章。
不能自动写第 2 章。
```

### 10.10 章节总结和等待继续

```bash
python -m scripts.cli gate-chapter --project "第一人称乡村逆袭爽文_田螺夜市供货" --chapter 1
```

输出：

```text
章节总结
章节钩子
对话占比
等待用户回复“继续”
```

### 10.11 审核

```bash
python -m scripts.cli audit --project "第一人称乡村逆袭爽文_田螺夜市供货"
```

### 10.12 修复

```bash
python -m scripts.cli fix --project "第一人称乡村逆袭爽文_田螺夜市供货"
```

### 10.13 导出

```bash
python -m scripts.cli export --project "第一人称乡村逆袭爽文_田螺夜市供货"
```

### 10.14 字幕到第一章一键流程

```bash
python -m scripts.cli run-subtitle-to-chapter --name "第一人称乡村逆袭爽文_田螺夜市供货" --input input/demo.srt --rules input/rewrite_rules.json --chapter 1
```

执行到：

```text
第一章生成
第一章总结
等待用户说继续
```

### 10.15 视频到第一章一键流程

```bash
python -m scripts.cli run-video-to-chapter --name "第一人称乡村逆袭爽文_田螺夜市供货" --input input/demo.mp4 --rules input/rewrite_rules.json --chapter 1
```

---

## 11. `examples/sample_rewrite_rules.json`

```json
{
  "novel_positioning_name": "第一人称乡村逆袭爽文_田螺夜市供货",
  "rewrite_goal": "根据视频/小说对话素材，改写成新的第一人称快节奏爽文小说",
  "target_style": "短平快爽文，节奏快，爽点多，第一章开篇即高潮",
  "pov": "第一人称",
  "target_total_words": {
    "min": 40000,
    "max": 50000
  },
  "chapter_word_range": {
    "min": 1000,
    "max": 3000
  },
  "dialogue_ratio_rule": {
    "min": 0.5,
    "target": 0.6,
    "counting_method": "只要段落包含人物说出口的话，整段都算对白内容"
  },
  "layout_rule": "常规小说排版，段落不要太紧凑",
  "chapter_output_rule": "一次只输出一章，写完总结并等待用户说继续",
  "chapter_hook_rule": "每章结尾必须有钩子",
  "naming_rule": "所有人物名、地点名、组织名、产业名不得与原文一样",
  "safety_rewrite_rule": "弱化血腥、暴力、恐怖、低俗和不适宜未成年人阅读的直接描写，不写刺激性细节，保留情节推进、人物关系与结局走向",
  "forbidden_terms": [],
  "required_terms": [],
  "character_mapping": {},
  "topic_mapping": {},
  "location_mapping": {},
  "industry_mapping": {}
}
```

---

## 12. 关键 Prompt 内容

### 12.1 `prompts/chapter_writing.md`

必须包含：

```markdown
# 当前章节写作任务

你只写当前一章，不要写下一章。

## 强制要求

1. 主角必须使用第一人称。
2. 所有人名、地名、组织名、产业名不得和原文一样。
3. 文字排版不能太紧凑，要像常规小说。
4. 每章 1000 到 3000 中文字。
5. 如果是第一章，开篇必须炸裂，直接抛核心设定和惊天反转。
6. 整体节奏快，爽点多。
7. 每章结尾必须有钩子。
8. 不要写剧情摘要，必须写小说正文。
9. 弱化血腥、暴力、恐怖、低俗、不适宜未成年人阅读的直接描写。
10. 不照搬原文句子、原文特征性台词、原文命名。
11. 对话内容必须占本章篇幅一半以上。
12. 只要段落包含人物说出口的话，整段都算对白内容。
13. 建议对话占比控制在 55% 到 65%。

## 输出

只输出当前章节正文。
```

### 12.2 `prompts/chapter_gate.md`

必须包含：

```markdown
# 章节总结与继续门控

请只总结当前章节，不要写下一章。

必须输出：

1. 本章摘要
2. 本章爽点
3. 本章结尾钩子
4. 本章字数
5. 本章对话占比
6. 当前总进度
7. 下一章建议方向
8. 最后一行必须写：请回复“继续”，我再写下一章。
```

### 12.3 `prompts/audit.md`

必须包含：

```markdown
# 审核任务

只做审核，不做修复，不写新正文。

审核项目：

1. 原文人名是否残留
2. 原文地名是否残留
3. 原文行业词是否残留
4. 原文特征性台词是否残留
5. 是否存在整句重复
6. 是否存在连续 8/10/12 字重复
7. 是否只是换名字
8. 是否第一人称
9. 是否每章 1000 到 3000 字
10. 是否每章有钩子
11. 是否大段摘要化
12. 是否对话占比超过 50%
13. 是否存在过度血腥、暴力、恐怖、低俗、不适宜未成年人阅读的直接描写
14. 是否有章节逻辑断裂
15. 是否需要修复

对话占比规则：
只要段落包含人物说出口的话，整段都算对白内容。
```

---

## 13. 必须生成的文档

开发完成后必须生成：

```text
README.md
CHANGELOG.md
VERSION
references/file_responsibility_table.md
references/skill_prompt_mapping.md
references/github_reference_matrix.md
```

### 13.1 README.md 必须包含

```text
1. 插件介绍
2. 适用场景
3. 不适用场景
4. 安装方法
5. 可选依赖说明
6. 每个 Skill 文件是干什么的
7. 每个 Prompt 文件是干什么的
8. 每个脚本文件是干什么的
9. workspace 每个目录是干什么的
10. 如何根据新小说初步定位名称新建文件夹
11. 如何处理视频
12. 如何处理字幕
13. 如何一章一章写
14. 用户说“继续”后如何写下一章
15. 对话占比超过 50% 的规则
16. 如何审核
17. 如何修复
18. 如何导出
19. 常见问题
20. 版权与原创性提醒
```

### 13.2 CHANGELOG.md 初始内容

```markdown
# 更新记录

## v0.3.1 - 2026-06-13

### 新增
- 新增 Codex 插件结构。
- 新增 Skill 与 Prompt 分离。
- 新增新小说初步定位名称创建项目文件夹。
- 新增对话占比超过 50% 的写作与审核规则。
- 新增章节门控：用户说“继续”后才写下一章。
- 新增视频、字幕、OCR、场景、素材包、世界观、章节写作、审核、导出流程。
- 新增每个 Skill 文件前部中文功能说明要求。

### 变更
- 不使用本地 LLM，所有文本理解、规划、写作、审核都交给 Codex 当前接入模型。
- scripts 从节点级拆分改为流程级拆分，减少文件数量。
- 中文功能说明不强制必须是第一行，但必须在文件前部醒目位置。

### 已知问题
- 视频 OCR 依赖环境较重，未安装时会跳过。
- Whisper 未安装时需要用户提供字幕文件。
- 场景切分只能辅助剧情拆分，最终剧情场景仍需结合对白理解。
```

### 13.3 VERSION

```text
0.3.1
```

### 13.4 `references/file_responsibility_table.md`

必须列出：

```text
每个文件
文件职责
输入
输出
不允许做什么
```

### 13.5 `references/skill_prompt_mapping.md`

必须列出：

```text
每个 Skill 对应哪个 Prompt
哪些 Skill 不需要 Prompt
每个 Prompt 只负责什么
```

### 13.6 `references/github_reference_matrix.md`

必须列出：

```text
参考项目
可借鉴点
本插件如何采用
不采用的部分
```

---

## 14. requirements

### 14.1 requirements.txt

```txt
pydantic>=2.0.0
python-dotenv>=1.0.0
pytest>=8.0.0
rich>=13.0.0
```

### 14.2 requirements-optional.txt

```txt
faster-whisper>=1.0.0
opencv-python>=4.8.0
pillow>=10.0.0
pysrt>=1.1.2
webvtt-py>=0.5.1
paddleocr>=2.7.0
easyocr>=1.7.0
scenedetect>=0.6.0
python-docx>=1.1.0
```

缺依赖处理：

```text
没有 ffmpeg：跳过视频处理，提示用户提供字幕。
没有 faster-whisper：跳过语音转文字，提示用户提供 srt。
没有 OCR：跳过硬字幕识别。
没有 PySceneDetect：用固定时间切分。
```

不能因为可选依赖缺失导致整个流程崩溃。

---

## 15. plugin.json

创建：

```text
.codex-plugin/plugin.json
```

内容：

```json
{
  "name": "video-dialogue-novel-rewrite",
  "version": "0.3.1",
  "description": "A Codex plugin for video/subtitle/dialogue-driven story reconstruction and dialogue-heavy chapter-by-chapter first-person novel rewriting. No local LLM is used.",
  "author": {
    "name": "Local User"
  },
  "license": "MIT",
  "keywords": [
    "video",
    "dialogue",
    "subtitle",
    "ffmpeg",
    "whisper",
    "ocr",
    "scene-segmentation",
    "novel",
    "rewrite",
    "chapter-writing",
    "dialogue-heavy",
    "audit",
    "codex-skill"
  ],
  "skills": "./skills/",
  "interface": {
    "displayName": "Video Dialogue Novel Rewrite",
    "shortDescription": "Extract dialogue, build story assets, write dialogue-heavy chapters, and audit residue.",
    "longDescription": "This plugin uses separate Skills and Prompts to process video/subtitle/dialogue material, build story assets, plan rewriting, write one chapter at a time, enforce dialogue ratio above 50%, audit residue and logic, and export final novels.",
    "developerName": "Local User",
    "category": "Productivity",
    "capabilities": [
      "Read",
      "Write",
      "Run"
    ],
    "defaultPrompt": [
      "Create a project folder using the new novel positioning name.",
      "Use Codex-connected models for all language understanding and writing; do not use local LLM.",
      "Write exactly one chapter, then summarize it and wait for the user to say 继续."
    ]
  }
}
```

---

## 16. AGENTS.md

创建：

```markdown
# AGENTS.md

## Project purpose

This repository is a Codex plugin for video/subtitle/dialogue-driven story reconstruction and chapter-by-chapter first-person novel rewriting.

## Most important rules

1. Do not use local LLM.
2. Codex handles all language understanding, planning, writing, and audit.
3. Skill files and Prompt files must be separate.
4. Scripts must be grouped by workflow and kept minimal.
5. Each Skill must serve exactly one workflow node.
6. Each Skill file must include a Chinese function comment near the top.
7. The project folder must be created from the new novel positioning name.
8. The rewritten novel must be dialogue-heavy.
9. Dialogue content must be more than 50% of each chapter.
10. A paragraph containing spoken dialogue counts entirely as dialogue content.

## Writing rules

- First-person protagonist.
- No original names.
- No original place names.
- No original unique lines.
- Chapter 1 must open with high impact.
- Each chapter must be 1000 to 3000 Chinese characters unless the user says otherwise.
- Total target is 40000 to 50000 Chinese characters unless the user says otherwise.
- Each chapter must end with a hook.
- After each chapter, summarize and ask the user to reply “继续”.
- Do not write the next chapter until the user explicitly says “继续”.
- Weaken direct depictions of blood, violence, horror, vulgarity, and content unsuitable for minors.

## File minimization rule

- Skills and Prompts must be separate.
- Scripts should be grouped by workflow:
  - core_utils.py
  - media_pipeline.py
  - subtitle_pipeline.py
  - story_assets_pipeline.py
  - writing_pipeline.py
  - audit_export_pipeline.py
- Do not create one script per small node unless necessary.

## Documentation rule

After completing the plugin, write:
- README.md
- CHANGELOG.md
- VERSION
- references/file_responsibility_table.md
- references/skill_prompt_mapping.md
- references/github_reference_matrix.md
```

---

## 17. 测试要求

必须有测试：

```text
test_project_folder.py
test_subtitle_pipeline.py
test_material_package.py
test_chapter_state.py
test_dialogue_ratio.py
test_audit.py
```

重点测试：

```text
1. 新小说初步定位名称能创建文件夹。
2. 文件夹名能处理 Windows 非法字符。
3. 字幕能解析。
4. 素材包能生成。
5. 章节状态能记录“等待继续”。
6. 用户没说继续时不能写下一章。
7. 对话占比能计算。
8. 包含对白的整段会被算作对白内容。
9. 对话占比低于 50% 时审核失败。
10. 原文残留词能被检出。
```

---

## 18. 验收标准

Codex 完成后必须满足：

```text
1. `.codex-plugin/plugin.json` 存在。
2. skills 下有 17 个 Skill。
3. 每个 SKILL.md 文件前部有中文功能说明。
4. 每个 SKILL.md 只服务一个节点。
5. prompts 和 skills 分开。
6. scripts 文件数量精简，按流程分组。
7. 没有本地 LLM 相关代码。
8. README.md 详细。
9. CHANGELOG.md 有版本号、日期、更新内容。
10. VERSION 存在。
11. references/file_responsibility_table.md 存在。
12. references/skill_prompt_mapping.md 存在。
13. references/github_reference_matrix.md 存在。
14. `python -m scripts.cli create-project --name "第一人称乡村逆袭爽文_田螺夜市供货"` 可运行。
15. `python -m scripts.cli run-subtitle-to-chapter --name "第一人称乡村逆袭爽文_田螺夜市供货" --input examples/sample_subtitle.srt --rules examples/sample_rewrite_rules.json --chapter 1` 可运行。
16. 生成第一章后，chapter_state.json 必须显示 waiting_for_continue=true。
17. 不允许自动生成第二章。
18. audit 必须检查 dialogue_ratio > 0.5。
19. pytest 通过。
```

---

## 19. 最终使用方式

字幕输入：

```bash
python -m scripts.cli run-subtitle-to-chapter --name "第一人称乡村逆袭爽文_田螺夜市供货" --input input/demo.srt --rules input/rewrite_rules.json --chapter 1
```

视频输入：

```bash
python -m scripts.cli run-video-to-chapter --name "第一人称乡村逆袭爽文_田螺夜市供货" --input input/demo.mp4 --rules input/rewrite_rules.json --chapter 1
```

继续下一章：

```bash
python -m scripts.cli write-chapter --project "第一人称乡村逆袭爽文_田螺夜市供货" --chapter 2
python -m scripts.cli gate-chapter --project "第一人称乡村逆袭爽文_田螺夜市供货" --chapter 2
```

最终导出：

```bash
python -m scripts.cli export --project "第一人称乡村逆袭爽文_田螺夜市供货"
```

最终文件：

```text
workspace/第一人称乡村逆袭爽文_田螺夜市供货/13_final/final_story.md
workspace/第一人称乡村逆袭爽文_田螺夜市供货/13_final/final_chapter_index.md
workspace/第一人称乡村逆袭爽文_田螺夜市供货/13_final/final_audit_summary.md
workspace/第一人称乡村逆袭爽文_田螺夜市供货/13_final/export_index.md
```
