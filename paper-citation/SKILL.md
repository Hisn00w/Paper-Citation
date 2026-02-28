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

3. **中文语境保留**（当用户明确要求中文期刊时）：
   - 保留原始中文查询词作为变体
   - 添加中文期刊限定词（如"核心期刊"、"CSSCI"）

示例：
```
原文：虚拟现实技术在文化遗产数字化保护中的应用
变体1：virtual reality cultural heritage digital preservation
变体2：VR technology heritage conservation digitization
变体3：immersive technology museum artifact protection
```

## 步骤2：语义搜索

### 检索策略决策

根据用户意图选择检索路径：

```
用户输入
    │
    ├── 明确要求中文期刊/核心期刊/CSSCI
    │       └── 优先中文数据库检索（步骤2B）
    │       └── 英文API作为补充
    │
    └── 未指定或要求国际期刊
            └── 优先英文API检索（步骤2A）
            └── 中文数据库作为补充
```

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

### 中文数据库检索回退机制（步骤2B）

当用户明确要求中文期刊（如"核心期刊"、"CSSCI"、"北大核心"）或英文API检索结果不足时，启用中文数据库检索：

**触发条件：**
- 用户明确指定中文期刊等级（CSSCI、北核、CSCD等）
- 用户查询涉及纯中文语境概念（如"双减政策"、"乡村振兴"、"新质生产力"）
- Semantic Scholar + OpenAlex 返回结果少于5篇相关论文

**中文检索策略：**

1. **保留中文查询词**
   - 使用原始中文查询（不翻译为英文）
   - 添加中文期刊限定词：`核心期刊`、`CSSCI来源期刊`

2. **使用 WebSearch 检索中文数据库**

   **检索目标平台：**
   - 中国知网（CNKI）
   - 万方数据
   - 维普期刊
   - 百度学术

   **搜索查询模板：**
   ```
   site:cnki.net "查询关键词" CSSCI
   site:wanfangdata.com.cn "查询关键词" 核心期刊
   "查询关键词" 知网 核心期刊
   "查询关键词" 百度学术
   ```

3. **解析检索结果**
   - 从搜索结果提取论文标题、作者、期刊、年份
   - 优先选择 CSSCI、北大核心期刊论文
   - 通过标题在 Semantic Scholar 查找对应的英文元数据（如有）

4. **与英文API结果合并**
   - 中文数据库结果优先排序（当用户明确要求中文期刊时）
   - 去重：同一论文的中英文版本视为一篇

**中文数据库检索示例：**

```
用户查询："数字孪生在智慧城市建设中的应用研究，要CSSCI期刊"

处理流程：
1. 识别触发条件：明确要求CSSCI
2. 中文检索："数字孪生 智慧城市 CSSCI 核心期刊"
3. 英文补充检索："digital twin smart city"
4. 合并结果，优先展示CSSCI期刊论文
```

**注意事项：**
- 中文数据库通常需要机构访问权限，使用公开可访问的搜索页面
- 部分中文期刊论文可能没有DOI，通过标题+作者+年份去重
- 中文期刊等级优先查询本地 venue-rankings.md 缓存

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

### 本地缓存写入规范

**重要：使用追加模式，禁止覆盖整个文件！**

当发现新的期刊等级信息时，按以下步骤追加到 `references/venue-rankings.md`：

**步骤1：确定领域分区**
- 根据期刊主题找到对应的领域分区（如 HCI、Design Studies、AI/ML 等）
- 如不存在相关分区，在文件末尾新建分区

**步骤2：使用标准表格格式追加**

```markdown
| Venue | Full Name | Rank | Notes |
|-------|-----------|------|-------|
| 期刊缩写 | 期刊全称 | 等级 | 备注 |
```

**步骤3：追加模板示例**

```markdown
### [新领域名称]

| Venue | Full Name | Rank | Notes |
|-------|-----------|------|-------|
| NewJournal | New Journal Name | SCI-Q2 | Added on 2024-XX-XX |
```

**写入规则：**
- ✅ **必须**：在现有表格后追加新行，保持表格结构完整
- ✅ **必须**：保持表头格式一致（`| Venue | Full Name | Rank | Notes |`）
- ❌ **禁止**：删除或修改已有期刊条目
- ❌ **禁止**：覆盖整个文件内容
- ⚠️ **注意**：添加后检查 Markdown 表格语法（| 对齐）

**追加位置示例：**
```markdown
| JUS | Journal of Usability Studies | C | UPA journal |
| CHI EA | CHI Extended Abstracts | C | Workshop papers |
| **NewJournal** | **New Journal Name** | **SCI-Q2** | **Added 2024** |  ← 追加到此处
```

## 步骤4：论文评分和排序

计算每篇论文的综合得分（0-100）：

```
基础得分 = (被引量得分 × 0.30) +
           (期刊得分 × 0.25) +
           (相关性得分 × 0.25) +
           (时效性得分 × 0.20)

最终得分 = 基础得分 × 高影响力引用加成系数
```

### 各项得分

#### 被引量得分（0-100）- 时间归一化

为避免新论文因发表时间短而总被引量偏低，首先计算年均被引量：

**核心公式：**
```
年均被引量 = citationCount / max(1, 当前年份 - 发表年份 + 1)
```
（注：当年发表的论文，分母为 1）

**阶梯计分规则（模拟对数递减效应）：**

根据算出的"年均被引量"，按以下阶梯赋予得分（向下取整，无需精确计算小数）：

| 年均被引量范围 | 被引得分 | 论文学术热度评估 |
|----------------|----------|------------------|
| ≥ 100 次/年    | 100 分   | 现象级/顶级热度   |
| 50 - 99 次/年  | 90 分    | 极高热度          |
| 20 - 49 次/年  | 75 分    | 高热度            |
| 10 - 19 次/年  | 60 分    | 良好关注度        |
| 5 - 9 次/年    | 45 分    | 一般关注度        |
| < 5 次/年      | 30 分    | 基础传播或冷门    |

**示例对比：**

| 论文 | 年份 | 总被引 | 发表年数 | 年均被引 | 所在区间 | 被引得分 |
|------|------|--------|----------|----------|----------|----------|
| 经典论文A | 2018 | 500 | 8 | 62.5 | 50-99 | 90 分 |
| 新论文B | 2024 | 80 | 2 | 40.0 | 20-49 | 75 分 |
| 当年论文C | 2025 | 25 | 1 | 25.0 | 20-49 | 75 分 |

> 注：此阶梯计分法对 AI 助手极为友好——只需做一次简单除法再查表，无需对数等复杂浮点运算，同时通过阶段式递减（前段增长快、后段增长慢）完美模拟了对数曲线的边际收益递减效应，保障新旧论文的公平对比。

#### 高影响力引用加成（独立乘数因子）

Semantic Scholar 的 `influentialCitationCount` 反映引用质量，作为**独立的乘数因子**应用于最终得分：

| 高影响力引用数 | 加成系数 | 说明 |
|----------------|----------|------|
| ≥ 50 | × 1.30 | 领域里程碑论文，具有变革性影响 |
| 20-49 | × 1.20 | 重要影响论文，定义了研究方向 |
| 10-19 | × 1.10 | 显著影响论文，被广泛关注和讨论 |
| 5-9 | × 1.05 | 一定影响论文，产生了学术影响 |
| 0-4 | × 1.00 | 基础水平，正常学术传播 |

**应用方式：**
```
最终得分 = 基础得分 × 高影响力加成系数
```

**高影响力引用加权的重要性：**
- 区别于普通被引量，反映引用的"质量"而非"数量"
- 被领域专家标注为具有方法创新、理论突破或实际影响的引用
- 能有效识别真正推动领域发展的关键论文

**完整评分示例：**

| 论文 | 年份 | 总被引 | 年均被引 | 被引得分 | 期刊得分 | 时效性 | 基础分 | 高影响引用 | 加成 | 最终得分 |
|------|------|--------|----------|----------|----------|--------|--------|------------|------|----------|
| 论文A | 2024 | 50 | 25 | 42.9 | 90 | 95 | 67.1 | 8 | ×1.05 | 70.5 |
| 论文B | 2018 | 300 | 37.5 | 51.6 | 75 | 60 | 61.0 | 3 | ×1.00 | 61.0 |
| 论文C | 2020 | 500 | 100 | 60.2 | 90 | 75 | 76.3 | 35 | ×1.20 | 91.6 |
| 论文D | 2022 | 200 | 66.7 | 56.5 | 100 | 85 | 78.4 | 55 | ×1.30 | 101.9→100 |

> 论文D因高影响力引用超过50，获得30%加成，体现了高质量研究的影响力权重。

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

**评分优化示例：**

| 论文 | 年份 | 总被引 | 年均被引 | 高影响力引用 | 加成 | 被引得分 |
|------|------|--------|----------|--------------|------|----------|
| 论文A | 2024 | 50 | 50 | 8 | ×1.05 | 54.4 |
| 论文B | 2018 | 300 | 43 | 3 | ×1.00 | 52.0 |
| 论文C | 2020 | 500 | 100 | 35 | ×1.20 | 96.0 |

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
