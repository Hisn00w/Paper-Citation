---
name: paper-citation
description: 当用户需要查找学术论文引用、搜索相关文献参考资料或生成引用格式（GB/T 7714、APA、IEEE、MLA、BibTeX、LaTeX cite格式）时使用此技能。当用户提供描述研究概念的文字、在文档中标记[CITE]位置，或要求带有特定约束条件（年份限制、期刊质量、引用格式）的引用推荐时使用。
---

# 文献引用检索 (Paper Citation)

## 概述

此技能用于智能学术文献引用检索和格式化。支持三种输入模式：直接文本查询、文档中[CITE]标记替换、以及带约束条件的自然语言指令。技能通过语义搜索跨多个学术数据库（Semantic Scholar、OpenAlex），按被引量和期刊质量排序，并输出多种引用格式：GB/T 7714-2015（中国标准）、APA（第7版）、IEEE、MLA（第9版）、BibTeX或LaTeX \cite{}格式。

## 何时使用此技能

- 用户提供一段文字并要求相关引用时
- 用户文档中有[CITE]标记需要替换为实际引用时
- 用户请求带有特定约束条件（年份范围、期刊等级、引用数量）的引用时
- 用户需要LaTeX、Word或其他文档格式的引用时

## 工作流程决策树

```
用户输入
    │
    ├── 直接文本描述（无[CITE]标记）
    │       └── 将整段文字作为单一搜索查询
    │
    └── 包含[CITE]标记
            └── 提取每个[CITE]位置及其上下文
                    └── 独立处理每个位置

对每个查询：
    │
    ├── 步骤1：解析和优化
    │       └── 提取核心学术概念，翻译为英文
    │       └── 生成2-3个搜索变体（同义词、缩写）
    │
    ├── 步骤2：语义搜索
    │       └── 查询Semantic Scholar API（主检索）
    │       └── 查询OpenAlex API（补充检索）
    │       └── 合并结果并去重
    │
    ├── 步骤3：期刊质量评估
    │       └── 检查venue-rankings.md本地缓存
    │       └── 如未找到，联网搜索CSSCI/SCI/SSCI分区
    │
    ├── 步骤4：评分和排序
    │       └── 计算综合得分：
    │           - 被引量 (30%)
    │           - 期刊等级 (25%)
    │           - 语义相关性 (25%)
    │           - 发表年份 (20%)
    │
    ├── 步骤5：获取BibTeX
    │       └── 通过DOI从CrossRef/doi.org获取
    │
    └── 步骤6：格式化输出
            └── GB/T 7714-2015格式（中国标准）
            └── APA 7th格式（美国心理学会）
            └── IEEE格式（电气电子工程师学会）
            └── MLA 9th格式（现代语言协会）
            └── BibTeX格式
            └── LaTeX \cite{key1,key2}格式
```

## 步骤1：解析输入并生成搜索查询

### 输入模式检测

**模式A - 直接文本查询：**
如果输入不包含[CITE]标记：
- 将整段文字作为单一语义搜索查询
- 去除口语化填充，保留学术实质内容

**模式B - [CITE]标记替换：**
如果输入包含`[CITE]`标记：
- 定位每个`[CITE]`位置
- 提取前后各1-2句作为上下文
- 为每个标记创建独立查询
- 记录原始位置用于替换

### 查询优化

对每个提取的查询：

1. **翻译为英文**（如果输入是中文）：
   - 识别核心学术概念
   - 去除口语化表达
   - 保留技术术语

2. **生成搜索变体**（2-3个变体）：
   - 同义词替换（如"machine learning" → "deep learning"）
   - 缩写展开（如"VR" → "virtual reality"）
   - 相关概念扩展（如"user experience" → "UX usability"）

示例：
```
原文：虚拟现实技术在文化遗产数字化保护中的应用
变体1：virtual reality cultural heritage digital preservation
变体2：VR technology heritage conservation digitization
变体3：immersive technology museum artifact protection
```

## 步骤2：语义搜索

### 主检索：Semantic Scholar API

使用此API进行语义相关性排序：

```bash
curl -s "https://api.semanticscholar.org/graph/v1/paper/search?query={ENCODED_QUERY}&limit=20&fields=title,authors,year,citationCount,venue,externalIds,abstract,journal,publicationTypes,referenceCount,influentialCitationCount,tldr"
```

**关键提取字段：**
- `paperId`：唯一标识符
- `title`：论文标题
- `authors`：作者列表
- `year`：发表年份
- `citationCount`：总被引量
- `venue`/`journal`：发表期刊
- `externalIds.DOI`：用于获取BibTeX的DOI
- `abstract`：论文摘要
- `tldr`：AI生成的摘要
- `influentialCitationCount`：高影响力引用

### 补充检索：OpenAlex API

用于更广泛的期刊覆盖：

```bash
curl -s "https://api.openalex.org/works?search={ENCODED_QUERY}&per_page=20&select=id,doi,title,authorships,publication_year,cited_by_count,primary_location,type,biblio"
```

**关键字段：**
- `id`：OpenAlex ID
- `title`：论文标题
- `authorships`：作者信息
- `publication_year`：年份
- `cited_by_count`：被引量
- `primary_location.source.display_name`：期刊名
- `doi`：DOI标识符

### 结果合并

1. 将两组结果规范化为统一格式
2. 按DOI去重（如无DOI则按标题+年份）
3. 保留前20个唯一结果

## 步骤3：期刊质量评估

### 本地缓存查询

首先检查`references/venue-rankings.md`中的期刊等级：

**预填充领域：**
- Design Studies（设计学）
- HCI / Human-Computer Interaction（人机交互）
- Cultural Heritage / Tourism（文旅/文化遗产）
- Computer Science / AI / Machine Learning / Deep Learning（计算机/人工智能/机器学习/深度学习）
- Large Language Models / NLP（大语言模型/自然语言处理）
- Computer Systems / Networks / Software Engineering（计算机系统/网络/软件工程）
- Cyber Security / Privacy（网络安全/隐私）
- Chinese Computer Journals（中文计算机期刊）

**等级分类：**
- A+：顶级（Nature/Science、CHI、TOCHI等）
- A：高级（CSCW、UIST、IJHCS等）
- B：良好级（Interacting with Computers等）
- C：认可级
- CSSCI：中文社会科学引文索引
- 北核：北京大学核心期刊
- SCI-Q1/Q2/Q3/Q4：SCI分区
- SSCI-Q1/Q2/Q3/Q4：SSCI分区

### 未知期刊的联网搜索

如果期刊不在本地缓存中：

1. **搜索查询模板：**
   - `"{venue name}" CSSCI 来源期刊`
   - `"{venue name}" 北大核心 期刊`
   - `"{venue name}" SCI impact factor quartile`
   - `"{venue name}" SSCI Q1 Q2`

2. **使用WebSearch工具**查找等级信息

3. **缓存新发现**到venue-rankings.md供将来使用

## 步骤4：论文评分和排序

计算每篇论文的综合得分（0-100）：

```
得分 = (被引量得分 × 0.30) +
       (期刊得分 × 0.25) +
       (相关性得分 × 0.25) +
       (时效性得分 × 0.20)
```

### 各项得分

**被引量得分（0-100）：**
- 对数归一化：`min(100, 20 × log10(citationCount + 1))`
- influentialCitationCount > 10时额外加10分

**期刊得分（0-100）：**
| 等级 | 得分 |
|------|-------|
| A+ | 100 |
| A / SCI-Q1 / SSCI-Q1 | 90 |
| B / SCI-Q2 / SSCI-Q2 | 75 |
| C / SCI-Q3 / SSCI-Q3 | 60 |
| SCI-Q4 / SSCI-Q4 | 45 |
| CSSCI / 北核 | 70 |
| 其他认可期刊 | 30 |
| 未知 | 20 |

**相关性得分（0-100）：**
- 基于API的语义相关性排名
- 排名1-5：90-100
- 排名6-10：70-89
- 排名11-20：50-69

**时效性得分（0-100）：**
- 当年：100
- 1-2年前：90
- 3-5年前：75
- 6-10年前：60
- >10年：40（经典论文仍有价值）

### 应用用户约束

如果用户指定了约束条件：
- **年份范围**：评分前过滤
- **最大结果数**：排序后返回前N个
- **最低等级**：过滤低于阈值的期刊

## 步骤5：获取BibTeX

对于排名靠前且有DOI的论文：

### 方法1：CrossRef API
```bash
curl -s "https://api.crossref.org/works/{DOI}/transform/application/x-bibtex"
```

### 方法2：DOI.org内容协商
```bash
curl -s -LH "Accept: application/x-bibtex" "https://doi.org/{DOI}"
```

### 备选：生成BibTeX
如果DOI解析失败，从可用元数据生成：
```bibtex
@article{key,
  title = {Paper Title},
  author = {Author1 and Author2},
  journal = {Journal Name},
  year = {2024},
  doi = {10.xxxx/xxxxx}
}
```

## 步骤6：格式化输出

### 格式1：GB/T 7714-2015（中国标准）

**期刊论文：**
```
[1] 作者. 文章标题[J]. 期刊名, 年份, 卷(期): 页码.
[2] SMITH J, JOHNSON A. Title of Paper[J]. Journal Name, 2023, 45(3): 123-145.
```

**会议论文：**
```
[3] 作者. 论文标题[C]//会议名称. 地点: 出版社, 年份: 页码.
```

### 格式2：APA 7th Edition（美国心理学会）

**期刊论文：**
```
Smith, J. A., & Johnson, B. C. (2023). Title of the article in sentence case. Journal Name in Title Case, 45(3), 123-145. https://doi.org/10.xxxx/xxxxx
```

**会议论文：**
```
Lee, S. M. (2024, May). Title of conference paper. In A. Editor & B. Chair (Eds.), Proceedings of Conference Name (pp. 56-67). Publisher. https://doi.org/10.xxxx/xxxxx
```

**书籍章节：**
```
Brown, K. L. (2022). Chapter title. In R. Editor (Ed.), Book Title (3rd ed., pp. 112-134). Publisher.
```

**APA格式要点：**
- 作者：姓在前，名首字母在后，& 连接最后两位作者
- 年份：放在括号内
- 标题：句子大小写（仅首单词和专有名词大写）
- 期刊名：标题大小写，斜体，卷号斜体，期号不斜体
- DOI：以 https://doi.org/ 开头

### 格式3：IEEE格式（电气电子工程师学会）

**期刊论文：**
```
[1] J. A. Smith and B. C. Johnson, "Title of paper in sentence case," IEEE Trans. Journal Name, vol. 45, no. 3, pp. 123-145, Mar. 2023, doi: 10.xxxx/xxxxx.
```

**会议论文：**
```
[2] S. M. Lee et al., "Title of conference paper," in Proc. IEEE Conf. Name (CONF), City, Country, May 2024, pp. 56-67.
```

**书籍：**
```
[3] K. L. Brown, Book Title in Title Case. City, State: Publisher, 2022.
```

**IEEE格式要点：**
- 作者：名首字母 + 姓，et al. 表示三位及以上作者
- 标题：双引号内，句子大小写
- 期刊名：斜体，通常缩写（如 IEEE Trans. Pattern Anal. Mach. Intell.）
- 卷号：vol. 前缀，斜体
- 期号：no. 前缀
- 页码：pp. 前缀
- 使用数字编号 [1], [2], [3]

### 格式4：MLA 9th Edition（现代语言协会）

**期刊论文：**
```
Smith, John A., and Barbara C. Johnson. "Title of Article in Title Case." Journal Name in Title Case, vol. 45, no. 3, 2023, pp. 123-145. Database, doi:10.xxxx/xxxxx.
```

**会议论文：**
```
Lee, Samuel M. "Title of Conference Paper." Proceedings of Conference Name, edited by Alice Editor, Publisher, 2024, pp. 56-67.
```

**书籍：**
```
Brown, Karen L. Book Title in Title Case. Publisher, 2022.
```

**MLA格式要点：**
- 作者：全名，姓在前，and 连接作者
- 标题：文章用双引号，期刊/书籍用斜体
- 期刊名：斜体，后接卷号、期号、年份
- 页码：pp. 前缀
- 容器概念：期刊、数据库等作为容器
- DOI：前缀 doi:（不加 https://）

### 格式5：BibTeX

```bibtex
@inproceedings{smith2023example,
  title = {Example Paper Title},
  author = {Smith, John and Johnson, Alice},
  booktitle = {Proceedings of CHI 2023},
  year = {2023},
  pages = {1--10},
  doi = {10.1145/xxxxxx}
}
```

### 格式6：LaTeX \cite{}

用于[CITE]替换模式：
- 将每个`[CITE]`替换为`\cite{key1,key2,...}`
- 引用键由第一作者姓氏+年份派生（如`smith2023`）
- 在末尾提供参考文献部分

### 格式选择指南

| 格式 | 适用场景 | 特点 |
|------|----------|------|
| GB/T 7714-2015 | 中国学术论文、学位论文 | 中文学术标准，支持中英文 |
| APA 7th | 社会科学、心理学、教育学 | 作者-年份制，强调时效性 |
| IEEE | 工程技术、计算机科学 | 数字编号，期刊名缩写 |
| MLA | 人文学科、文学研究 | 作者-页码制，强调原文 |
| BibTeX | LaTeX文档管理 | 数据库格式，便于管理 |
| LaTeX cite | LaTeX文档引用 | 与BibTeX配合使用 |

## 资源文件

### references/venue-rankings.md

主要学术领域的期刊和会议等级参考。包含以下预填充等级：
- Human-Computer Interaction (HCI / 人机交互)
- Design Studies（设计学）
- Cultural Heritage and Tourism（文化遗产与旅游）
- Computer Science / AI / Machine Learning / Deep Learning（计算机/人工智能/机器学习/深度学习）
- Large Language Models / NLP（大语言模型/自然语言处理）
- Computer Systems / Networks / Software Engineering（计算机系统/网络/软件工程）
- Cyber Security / Privacy（网络安全/隐私）
- Chinese Computer Journals（中文计算机期刊）

评估检索论文的期刊质量时加载此文件。

### references/api-reference.md

以下API的文档和响应模式：
- Semantic Scholar API
- OpenAlex API
- CrossRef API

## 使用示例

### 示例1：直接文本查询

用户："帮我找关于虚拟现实在博物馆应用的文献"

处理流程：
1. 翻译："virtual reality museum application"
2. 生成变体："VR museum"、"immersive museum experience"
3. 搜索两个API
4. 按得分排序
5. 以GB/T 7714格式返回前5篇

### 示例2：[CITE]替换

用户："虚拟现实技术正在改变博物馆的展示方式[CITE]。沉浸式体验能够提升观众的参与感[CITE]。"

处理流程：
1. 提取查询1："虚拟现实技术正在改变博物馆的展示方式"
2. 提取查询2："沉浸式体验能够提升观众的参与感"
3. 分别搜索每个查询
4. 替换标记：`\cite{author2023}`和`\cite{smith2024}`
5. 提供完整参考文献

### 示例3：带约束搜索

用户："找近5年发表在SCI Q1期刊上关于数字孪生的文献，需要3篇，输出BibTeX格式"

处理流程：
1. 应用年份过滤：2020-2025
2. 应用期刊过滤：仅SCI-Q1
3. 搜索"digital twin"
4. 以BibTeX格式返回前3篇

### 示例4：指定国际引用格式

用户："找关于大语言模型在医疗应用的文献，输出APA格式"

处理流程：
1. 搜索"large language models healthcare medical application"
2. 按综合得分排序
3. 以APA 7th格式返回引用

### 示例5：IEEE格式用于工程领域

用户："找关于边缘计算在物联网中应用的论文，3篇，用IEEE格式"

处理流程：
1. 搜索"edge computing IoT Internet of Things"
2. 筛选高被引工程类论文
3. 以IEEE编号格式返回引用

### 示例6：MLA格式用于人文研究

用户："找关于数字人文在档案管理中应用的研究，输出MLA格式"

处理流程：
1. 搜索"digital humanities archival management"
2. 以MLA 9th格式返回引用

## 重要提示

1. **搜索学术API前务必将中文查询翻译为英文**
2. **验证DOI解析** - 部分论文可能元数据不完整
3. **缓存新发现的期刊等级**以便将来使用
4. **遵守API速率限制** - 请求之间使用合理延迟
5. **优雅处理缺失数据** - 部分论文可能缺少摘要或DOI
