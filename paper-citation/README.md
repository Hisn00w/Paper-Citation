# Paper Citation Skill / 文献引用检索技能

智能文献引用检索工具 - 支持语义搜索、期刊等级评估和6种引用格式输出（GB/T 7714、APA、IEEE、MLA、BibTeX、LaTeX）。

---

## 安装 / Installation

### 方法1：直接克隆到 Claude Skills 目录

```bash
# Windows
cd %USERPROFILE%\.claude\skills
git clone https://github.com/Hisn00w/Paper-Citation.git paper-citation

# macOS/Linux
cd ~/.claude/skills
git clone https://github.com/Hisn00w/Paper-Citation.git paper-citation
```

### 方法2：手动下载

1. 下载本仓库 ZIP 文件
2. 解压到 `C:\Users\[用户名]\.claude\skills\paper-citation` (Windows) 或 `~/.claude/skills/paper-citation` (macOS/Linux)

### 方法3：通过 Claude Code 安装

在 Claude Code 中执行：
```
/skill add paper-citation
```

或使用 skill 文件：
```bash
cd %USERPROFILE%\.claude\skills  # Windows
cd ~/.claude/skills              # macOS/Linux

# 下载 skill 压缩包
# 解压后重启 Claude Code
```

---

## 功能特点

- **三种输入模式**：直接文本查询、[CITE]标记替换、带约束条件搜索
- **双API检索**：Semantic Scholar + OpenAlex，覆盖更全
- **中文数据库回退**：知网、万方、维普等中文期刊检索支持
- **智能评分**：时间归一化被引量 + 高影响力引用乘数 + 期刊等级 + 时效性
- **六种引用格式**：GB/T 7714-2015、APA 7th、IEEE、MLA 9th、BibTeX、LaTeX \cite{}
- **期刊等级速查**：内置HCI/设计学/文化遗产等方向期刊等级

---

## 使用方式

### 方式1：直接文本查询

直接输入描述研究内容的文字，系统会自动提炼学术概念并检索相关文献。

```
用户：帮我找关于虚拟现实在博物馆应用的文献
```

### 方式2：[CITE]标记替换

在正文中标记需要插入引用的位置，系统会针对每个位置检索并替换为\cite{}格式。

```
用户：虚拟现实技术正在改变博物馆的展示方式[CITE]。沉浸式体验能够提升观众的参与感[CITE]。
```

输出：
```latex
虚拟现实技术正在改变博物馆的展示方式\cite{smith2023,johnson2024}。沉浸式体验能够提升观众的参与感\cite{lee2024}。

\begin{thebibliography}{99}
\bibitem{smith2023} Smith J, et al. VR in Museums[J]. Museum Management, 2023.
\bibitem{johnson2024} Johnson A, et al. Virtual Reality Applications[C]//CHI 2024.
\bibitem{lee2024} Lee S, et al. Immersive Museum Experience[J]. Curator, 2024.
\end{thebibliography}
```

### 方式3：带约束条件搜索

可以附加自然语言指令，限定年份、期刊等级、数量等。

```
用户：找近5年发表在SCI Q1期刊上关于数字孪生的文献，需要3篇，输出BibTeX格式
```

## 支持的约束条件

- **年份范围**：近5年、2020-2024、近10年等
- **期刊等级**：SCI Q1/Q2/Q3/Q4、SSCI、CSSCI、北大核心
- **数量限制**：需要N篇、推荐3-5篇等
- **输出格式**：GB/T 7714、APA、IEEE、MLA、BibTeX、LaTeX cite格式

## 引用格式说明

### GB/T 7714-2015（中国标准）
适用于中文学术论文、学位论文。
```
[1] SMITH J, JOHNSON A. Title of Paper[J]. Journal Name, 2023, 45(3): 123-145.
```

### APA 7th Edition（美国心理学会）
适用于社会科学、心理学、教育学等领域。
```
Smith, J. A., & Johnson, B. C. (2023). Title of article. Journal Name, 45(3), 123-145.
```

### IEEE格式（电气电子工程师学会）
适用于工程技术、计算机科学等领域。
```
[1] J. A. Smith and B. C. Johnson, "Title of paper," IEEE Trans. Journal, vol. 45, no. 3, pp. 123-145, 2023.
```

### MLA 9th Edition（现代语言协会）
适用于人文学科、文学研究等领域。
```
Smith, John A., and Barbara C. Johnson. "Title of Article." Journal Name, vol. 45, no. 3, 2023, pp. 123-145.
```

### BibTeX
适用于LaTeX文档管理。
```bibtex
@article{smith2023,
  title={Title of Paper},
  author={Smith, John and Johnson, Alice},
  journal={Journal Name},
  year={2023}
}
```

### LaTeX \cite{}
用于LaTeX文档内联引用。
```latex
\cite{smith2023}
```

## 工作流程

1. **解析输入**：识别[CITE]标记，提取查询内容
2. **查询优化**：翻译为英文，生成2-3个检索变体；保留中文查询用于中文数据库
3. **语义检索**：
   - 优先调用 Semantic Scholar 和 OpenAlex API
   - 中文语境下启用知网/万方/维普检索回退
4. **期刊评估**：查询期刊等级（本地缓存+联网搜索）
5. **智能评分**：时间归一化被引量 × 高影响力引用加成 + 期刊等级 + 时效性
6. **获取BibTeX**：通过DOI从CrossRef获取官方引用
7. **格式化输出**：按指定格式生成引用

## 智能评分机制

### 时间归一化被引量
为避免新论文因发表时间短而被低估，采用**年均被引量**：
```
发表年数 = max(1, 当前年份 - 发表年份 + 1)
年均被引量 = 总被引量 / 发表年数
被引量得分 = min(100, 20 × log10(年均被引量 × 2 + 1))
```

**示例对比：**
| 论文 | 年份 | 总被引 | 年均被引 | 说明 |
|------|------|--------|----------|------|
| 经典论文A | 2018 | 500 | 62.5 | 老论文总量高但年均适中 |
| 新论文B | 2024 | 80 | 40.0 | 新论文总量低但年均接近 |

### 高影响力引用加成
Semantic Scholar 的 influentialCitationCount 作为质量乘数应用于最终得分：

| 高影响力引用数 | 加成系数 | 说明 |
|----------------|----------|------|
| ≥ 50 | × 1.30 | 领域里程碑论文 |
| 20-49 | × 1.20 | 重要影响论文 |
| 10-19 | × 1.10 | 显著影响论文 |
| 5-9 | × 1.05 | 一定影响论文 |
| 0-4 | × 1.00 | 基础水平 |

**完整评分公式：**
```
基础得分 = (被引量得分 × 0.30) + (期刊得分 × 0.25) + (相关性得分 × 0.25) + (时效性得分 × 0.20)
最终得分 = 基础得分 × 高影响力加成系数
```

## 中文数据库检索回退

当用户明确要求中文期刊（CSSCI、北大核心等）或涉及纯中文语境概念时：

**触发条件：**
- 用户指定"核心期刊"、"CSSCI"、"北大核心"
- 查询涉及纯中文概念（如"双减政策"、"乡村振兴"、仅在中国研究的课题）
- 英文API返回结果不足

**检索策略：**
1. **保留中文查询词**：如果用户明确要求中文期刊，保留一组中文查询词作为变体
2. **使用 WebSearch 检索**：知网（CNKI）、万方、维普、百度学术等平台
3. **优先返回 CSSCI/北大核心期刊论文**
4. **与英文API结果合并去重**

**搜索查询模板：**
```
site:cnki.net "查询关键词" CSSCI
site:wanfangdata.com.cn "查询关键词" 核心期刊
"查询关键词" 知网 核心期刊
"查询关键词" 百度学术
```

## 本地缓存写入规范

发现新期刊等级时，使用**追加模式**写入 `references/venue-rankings.md`：

```markdown
| Venue | Full Name | Rank | Notes |
|-------|-----------|------|-------|
| NewJournal | New Journal Name | SCI-Q2 | Added on 2024-XX-XX |
```

**规则：**
- ✅ 在现有表格后追加新行
- ❌ 禁止删除或修改已有条目
- ❌ 禁止覆盖整个文件

## 文件结构

```
paper-citation/
├── SKILL.md                    # 技能主文档（详细工作流程）
├── README.md                   # 使用说明
├── references/
│   ├── venue-rankings.md       # 期刊等级速查表
│   └── api-reference.md        # API接口文档
```

### 本地缓存写入规范

当发现新的期刊等级信息时，**必须使用追加模式**写入 `references/venue-rankings.md`：

**写入规则：**
- ✅ **必须**：在现有表格后追加新行，保持表格结构完整
- ✅ **必须**：保持表头格式一致（`| Venue | Full Name | Rank | Notes |`）
- ❌ **禁止**：删除或修改已有期刊条目
- ❌ **禁止**：覆盖整个文件内容

**追加模板示例：**
```markdown
| JUS | Journal of Usability Studies | C | UPA journal |
| CHI EA | CHI Extended Abstracts | C | Workshop papers |
| **NewJournal** | **New Journal Name** | **SCI-Q2** | **Added 2024** |  ← 追加到此处
```

## 预存期刊领域

- **HCI/人机交互**：CHI、UIST、CSCW、TOCHI、IMWUT等
- **设计学**：Design Studies、装饰、包装工程等
- **文化遗产**：Journal of Cultural Heritage、旅游学刊等
- **VR/AR**：IEEE VR、ISMAR、TVCG等
- **AI/ML/深度学习**：NeurIPS、ICML、ICLR、JMLR、TPAMI等
- **大语言模型/NLP**：ACL、EMNLP、NAACL、TACL、Computational Linguistics等
- **计算机系统/网络**：OSDI、SOSP、SIGCOMM、NSDI、TOCS、TON等
- **软件工程**：ICSE、FSE、PLDI、POPL、TSE、TOSEM等
- **数据库**：SIGMOD、VLDB、ICDE、TODS、TKDE等
- **网络安全**：S&P、CCS、USENIX Security、NDSS、TDSC、TIFS等
- **计算机视觉**：CVPR、ICCV、ECCV、IJCV、TIP等
- **中文计算机期刊**：计算机学报、软件学报、中国科学:信息科学等

## 注意事项

1. 中文查询会自动翻译为英文进行检索
2. 期刊等级优先查询本地缓存，查不到会联网搜索
3. 部分论文可能缺少DOI或BibTeX信息，会尝试自动生成
4. 请合理使用API，避免短时间内大量请求

## 示例

### 示例1：设计学文献检索

```
用户：帮我找关于服务设计在文旅融合中应用的文献，近3年的，要CSSCI期刊
```

### 示例2：LaTeX引用替换

```
用户：请将以下文字中的[CITE]替换为LaTeX引用格式：

情感计算技术能够识别用户的情绪状态[CITE]，在智能交互系统中具有重要应用[CITE]。
```

### 示例3：BibTeX导出

```
用户：找5篇关于数字人文在文化遗产保护中应用的高被引论文，输出BibTeX
```

### 示例4：APA格式输出

```
用户：找关于机器学习在教育评估中应用的文献，输出APA格式
```

### 示例5：IEEE格式输出

```
用户：找关于区块链在供应链管理中应用的论文，3篇，用IEEE格式
```

### 示例6：MLA格式输出

```
用户：找关于数字档案管理的研究，输出MLA格式
```

---

## API Reference

### Semantic Scholar API
```bash
curl -s "https://api.semanticscholar.org/graph/v1/paper/search?query={QUERY}&limit=20&fields=title,authors,year,citationCount,influentialCitationCount,venue,externalIds,abstract"
```

### OpenAlex API
```bash
curl -s "https://api.openalex.org/works?search={QUERY}&per_page=20&select=id,doi,title,authorships,publication_year,cited_by_count"
```

### CrossRef API (BibTeX)
```bash
curl -s -LH "Accept: application/x-bibtex" "https://doi.org/{DOI}"
```
