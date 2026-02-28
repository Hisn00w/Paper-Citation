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
- **智能评分**：综合考虑被引量、期刊等级、语义相关性和发表年份
- **六种引用格式**：GB/T 7714-2015、APA 7th、IEEE、MLA 9th、BibTeX、LaTeX \cite{}
- **期刊等级速查**：内置HCI/设计学/文化遗产等方向期刊等级

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
2. **查询优化**：翻译为英文，生成2-3个检索变体
3. **语义检索**：调用Semantic Scholar和OpenAlex API
4. **期刊评估**：查询期刊等级（本地缓存+联网搜索）
5. **智能评分**：综合多维度指标排序
6. **获取BibTeX**：通过DOI从CrossRef获取官方引用
7. **格式化输出**：按指定格式生成引用

## 文件结构

```
paper-citation/
├── SKILL.md                    # 技能主文档（详细工作流程）
├── README.md                   # 使用说明
├── references/
│   ├── venue-rankings.md       # 期刊等级速查表
│   └── api-reference.md        # API接口文档
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
curl -s "https://api.semanticscholar.org/graph/v1/paper/search?query={QUERY}&limit=20&fields=title,authors,year,citationCount,venue,externalIds,abstract"
```

### OpenAlex API
```bash
curl -s "https://api.openalex.org/works?search={QUERY}&per_page=20&select=id,doi,title,authorships,publication_year,cited_by_count"
```

### CrossRef API (BibTeX)
```bash
curl -s -LH "Accept: application/x-bibtex" "https://doi.org/{DOI}"
```
