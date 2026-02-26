+++
date = '2026-02-26T17:03:09+08:00'
draft = false
title = '从零到部署：我是如何用 Next.js + Python 构建一个全功能英语学习应用的'

+++

# 从零到部署：我是如何用 Next.js + Python 构建一个全功能英语学习应用的

## 前言

作为一个长期与英语打交道的开发者，我一直觉得市面上的背单词应用要么太重、要么太贵、要么不够开放。于是我决定自己动手，做一个开源的英语学习平台 —— **OpenWords**。它不仅能背单词，还能每日精读外刊，而且完全免费、无需注册、数据不出浏览器。

这篇文章会完整剖析 OpenWords 的技术实现，涵盖前端架构、间隔重复算法、爬虫系统和部署策略，希望对有类似想法的同学有所帮助。

## 一、项目全貌

OpenWords 分为两大功能板块：

1. **背单词** —— 涵盖高考、四六级、考研、托福、雅思、GRE、医学共 8 大词库，基于 SM-2 间隔重复算法，支持自定义词库上传（TXT/CSV/DOCX/PDF）。
2. **每日外刊精读** —— 自动抓取 Guardian、BBC、VOA、The Conversation 的文章，提供中英双语阅读、选词即译、长难句 AI 解析等功能。

技术栈上，前端使用 **Next.js 16 (App Router) + React 19 + TypeScript + Tailwind CSS 4**，后端数据处理使用 **Python 爬虫 + SQLite**，部署在 **Vercel**。

## 二、数据层设计：双数据库策略

这个项目最有意思的设计决策之一是采用了「双数据库」架构：

### vocab.db —— 76 万词条的词典数据库

词库数据来源于开源项目 ECDICT（76 万+ 英汉双解词条），通过 `build_db.py` 脚本聚合多个考试词表后生成一个 12MB 的 SQLite 文件。它在服务端通过 `better-sqlite3` 以只读模式加载：

```typescript
// src/lib/db.ts
export function getDb(): Database.Database | null {
  if (initialized) return db;
  initialized = true;
  try {
    const BetterSqlite3 = require("better-sqlite3");
    const dbPath = path.join(process.cwd(), "vocab.db");
    db = new BetterSqlite3(dbPath, { readonly: true, fileMustExist: true });
  } catch {
    db = null;
  }
  return db;
}
```

注意这里用了 `require()` 动态加载而不是 ES import —— 这是关键。因为 `better-sqlite3` 是原生 C++ 模块，在 Vercel 的 serverless 环境下可能不可用。通过 try-catch 包裹并返回 null，让应用在没有 vocab.db 的环境下也能优雅降级，而不是直接崩溃。

### articles.db → 静态 JSON —— 外刊数据的「编译时」策略

外刊数据走了另一条路：Python 爬虫将文章写入 `articles.db`，但部署时并不直接使用这个数据库，而是通过 `export_web_json.py` 脚本「编译」成静态 JSON 文件：

- `article-data/index.json` —— 所有文章的元数据索引（标题、来源、难度、日期等）
- `article-data/detail/{id}.json` —— 每篇文章的完整内容（段落 + 句子 + 翻译）

这些 JSON 文件随代码一起提交到 Git，部署时作为静态资源分发。这样做的好处很明显：**Vercel 无需数据库即可提供外刊功能**，serverless 冷启动快，读取性能也极好。前端通过 Server Actions 直接 `fs.readFileSync` 读取 JSON：

```typescript
// src/lib/article-actions.ts
function loadIndex(): ArticleMeta[] {
  if (_indexCache) return _indexCache;
  try {
    const raw = fs.readFileSync(path.join(DATA_DIR, "index.json"), "utf-8");
    const data = JSON.parse(raw) as { articles: ArticleMeta[] };
    _indexCache = data.articles;
    return _indexCache;
  } catch {
    return [];
  }
}
```

还加了一层内存缓存 `_indexCache`，在 serverless 函数的生命周期内避免重复读文件。

## 三、SM-2 间隔重复算法

背单词的核心在于「什么时候复习」。OpenWords 实现了经典的 SM-2（SuperMemo 2）算法，这也是 Anki 等主流记忆软件的基础。

算法的核心逻辑在 `sm2.ts` 中，采用三级评分制：

- **0 = 不认识**：重置重复次数，1 分钟后再次出现
- **3 = 模糊**：给予部分信用，间隔以 1.2 倍缓慢增长
- **5 = 认识**：标准 SM-2 递进（1天 → 3天 → 乘以 easeFactor）

```typescript
// src/lib/sm2.ts
export function reviewCard(state: CardState, quality: Quality): CardState {
  const now = new Date();
  let { easeFactor, interval, repetitions } = state;

  if (quality === 0) {
    repetitions = 0;
    interval = 1 / 1440; // ~1 minute in days
  } else if (quality === 3) {
    if (repetitions === 0) {
      interval = 0.5; // 12 hours
    } else {
      interval = Math.max(interval * 1.2, 0.5);
    }
    repetitions += 1;
  } else {
    if (repetitions === 0) {
      interval = 1;
    } else if (repetitions === 1) {
      interval = 3;
    } else {
      interval = interval * easeFactor;
    }
    repetitions += 1;
  }
```

每次评分后，`easeFactor`（难度系数）会根据 SM-2 公式动态调整，最低不低于 1.3，防止间隔退化为零。整个计算过程是纯函数、不可变的，非常适合函数式编程风格。

## 四、客户端存储：IndexedDB 全面托管

OpenWords 的一个重要设计原则是**无需注册**。所有学习进度都存在浏览器的 IndexedDB 中，通过 `idb` 库封装。数据库包含 5 个 Object Store：

- `cardStates` —— SM-2 卡片状态（按 `wordId` 索引）
- `dailyStats` —— 每日学习/复习统计
- `settings` —— 用户偏好（口音、每日目标等）
- `customModules` —— 用户上传的自定义词库元信息
- `customWords` —— 自定义词库中的单词

IndexedDB 的 schema 升级也做了版本控制：v1 创建基础表，v2 新增自定义词库支持。这种渐进式升级保证了老用户不丢数据。

学习仪表盘（Dashboard）通过遍历所有 `cardStates` 来计算记忆保持率：

```typescript
// src/components/DashboardContent.tsx
function calcRetention(daysSinceReview: number, interval: number): number {
  if (interval <= 0) return 0;
  const stability = interval * 1.5;
  return Math.exp(-daysSinceReview / stability) * 100;
}
```

这是一个简化的艾宾浩斯遗忘曲线模型 —— 距离上次复习越久、间隔越短，保持率衰减越快。用这个公式就能在仪表盘上展示「平均记忆保持率」和「待复习单词数」。

## 五、外刊爬虫系统

爬虫是一个独立的 Python 包，位于 `data/crawler/`，架构非常清晰：

### 适配器模式的多源爬取

每个新闻源（Guardian、BBC、VOA、The Conversation）是一个独立的 Source 类，继承自 `BaseSource`，实现 `get_articles()` 方法返回 `RawArticle` 列表。主调度器只需遍历所有 Source：

```python
# data/crawler/main.py
for source in SOURCES:
    print(f"\n▶ {source.name.upper()}")
    try:
        articles = source.get_articles(limit=config.ARTICLES_PER_SOURCE)
    except Exception as exc:
        print(f"  ERROR: {exc}")
        continue
```

每个 Source 只负责解析自己网站的 HTML 结构，彼此完全解耦。要新增一个来源，只需写一个新的 Source 子类。

### 句子级分析管线

爬取到的文章不是直接存储，而是经过一条处理管线：

1. **句子拆分** —— 正则表达式按句号/问号/感叹号切分，同时排除 `Mr.`、`U.S.` 等缩写误切
2. **复杂句检测** —— 超过 25 词 或 包含 `which/although/nevertheless` 等从句标志词且超过 15 词 → 标记为复杂句
3. **翻译** —— 调用翻译服务获取中文译文
4. **结构化解析**（可选）—— 如果使用 DeepSeek 后端，对复杂句进行主谓宾拆解和从句类型识别

### 双翻译后端

翻译服务采用策略模式，两个实现共享同一个 `BaseTranslator` 接口：

- **GoogleTranslator** —— 免费，调用 Google 非官方翻译端点，无需 API Key，适合零成本运行
- **DeepSeekTranslator** —— 调用 DeepSeek LLM API，翻译质量更高，额外支持 `analyze_sentence()` 返回结构化语法分析

工厂函数 `get_translator()` 根据环境变量自动选择后端，实现了无缝切换。

## 六、前端交互亮点

### 翻转闪卡

`Flashcard` 组件用 Framer Motion 实现了真实的 3D 翻转效果 —— 正面显示单词和音标，点击翻转后显示释义，底部三个按钮分别对应「不认识 / 模糊 / 认识」三级评分。CSS 用 `backfaceVisibility: hidden` + `transform: rotateY(180deg)` 实现双面卡片效果。

### 选词即译

`SelectionTranslator` 是外刊阅读页的核心交互组件。用户在文章中选中任意文本后，组件会：

1. 计算选区位置，在下方弹出浮层
2. 判断是单词还是短语/句子
3. 单词 → 先查 vocab.db 词典（Server Action），显示音标、词性、释义和考试标签
4. 查不到或选中的是多词 → fallback 到翻译 API

整个过程有 300ms 的 debounce，避免快速选择时的多次请求。

### 长难句解析

文章中的复杂句会以蓝色背景高亮。点击后展开 `SentenceAnalyzer` 面板，展示：

- 句子主干（主语/谓语/宾语，分别用绿/蓝/紫色标签标注）
- 从句/修饰成分（定语从句、状语从句、分词短语等，用琥珀色标签）
- 结构说明（中文解释语法要点）

如果爬虫阶段已经预计算了分析结果，直接展示；否则提供一个「AI 深度解析」按钮，实时调用 DeepSeek API。

## 七、部署：Vercel 上的 Serverless 适配

部署到 Vercel 需要处理几个问题：

1. **vocab.db 太大（12MB）不提交到 Git** —— `db.ts` 的 try-catch 处理确保缺失时不崩溃，只是背单词功能不可用。
2. **article-data 需要打包进 serverless 函数** —— 在 `next.config.ts` 中配置 `outputFileTracingIncludes` 让 Vercel 把 JSON 数据一起打包。
3. **better-sqlite3 原生模块** —— 通过 `serverExternalPackages` 配置告知 Next.js 不要打包它，而是作为外部依赖加载。

外刊的更新流程也很简洁：本地跑爬虫 → 导出 JSON → git push → Vercel 自动部署。整个过程不需要数据库服务器、不需要 CI 脚本、不需要定时任务平台。

## 八、总结

OpenWords 的技术栈看似简单，但在架构决策上有不少值得玩味的地方：

- **双数据库策略**：vocab.db 走原生 SQLite，articles.db 「编译」为静态 JSON，各取所长
- **无注册设计**：IndexedDB 全面托管学习进度，隐私友好、部署简单
- **翻译双后端**：免费的 Google Translate 保底，DeepSeek LLM 锦上添花
- **爬虫与前端解耦**：Python 负责数据采集和预处理，Next.js 只消费静态数据
- **优雅降级**：从 DB 连接、原生模块到翻译服务，每个环节都有 fallback

这些选择背后的共同思路是：**让项目在最简环境下也能运行，在有条件时锦上添花**。对于个人开源项目而言，这种务实的工程态度可能比追逐最新技术更重要。

---

*项目地址：[github.com/Xlichard/OpenWords](https://github.com/Xlichard/OpenWords)，欢迎 Star 和贡献。*
