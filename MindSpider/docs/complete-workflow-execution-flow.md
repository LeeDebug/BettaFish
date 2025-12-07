# `python main.py --complete --test` 执行流程分析

## 概述

当用户执行 `python main.py --complete --test` 命令时，MindSpider 会运行完整的工作流程，包括两个核心模块：
1. **BroadTopicExtraction（话题提取）**：从多个新闻源收集热点新闻，使用 AI 提取关键词并生成分析总结
2. **DeepSentimentCrawling（深度情感爬取）**：基于提取的关键词，在多个社交媒体平台上进行内容爬取

`--test` 参数启用测试模式，在情感爬取阶段限制爬取数量以便快速验证功能。

## 执行流程图

```
用户执行命令
    ↓
MindSpider/main.py 解析参数
    ↓
调用 run_complete_workflow(test_mode=True)
    ↓
┌─────────────────────────────────────┐
│ 第一步：话题提取 (BroadTopicExtraction) │
└─────────────────────────────────────┘
    ↓
在 BroadTopicExtraction 目录下执行子进程
    ↓
执行话题提取流程
    ├─ 步骤1: 收集新闻 (collect_and_save_news)
    ├─ 步骤2: 提取关键词和总结 (extract_keywords_and_summary)
    └─ 步骤3: 保存到数据库 (save_daily_topics)
    ↓
生成搜索关键词并保存到文件
    ↓
话题提取完成 ✓
    ↓
┌─────────────────────────────────────┐
│ 第二步：情感爬取 (DeepSentimentCrawling) │
└─────────────────────────────────────┘
    ↓
在 DeepSentimentCrawling 目录下执行子进程
    ↓
解析 --test 参数，调整爬取限制
    ↓
执行情感爬取流程
    ├─ 步骤1: 获取关键词摘要 (get_crawling_summary)
    ├─ 步骤2: 获取关键词列表 (get_latest_keywords)
    └─ 步骤3: 执行全平台爬取 (run_multi_platform_crawl_by_keywords)
        ├─ 配置 MediaCrawler 数据库
        ├─ 为每个平台创建配置
        └─ 调用 MediaCrawler 进行爬取
    ↓
生成爬取报告
    ↓
完整工作流程完成 ✓
```

## 详细执行步骤

### 1. 命令入口（MindSpider/main.py）

**位置**: `main.py` 第 427-431 行

```python
elif args.complete:
    spider.run_complete_workflow(
        target_date, args.platforms, args.keywords_count, 
        args.max_keywords, args.max_notes, args.test
    )
```

**默认参数**:
- `target_date`: 如果未指定 `--date`，默认为今天
- `platforms`: 如果未指定 `--platforms`，默认为所有支持的平台
- `keywords_count`: 如果未指定 `--keywords-count`，默认为 100
- `max_keywords`: 如果未指定 `--max-keywords`，默认为 50
- `max_notes`: 如果未指定 `--max-notes`，默认为 50
- `test_mode`: `--test` 标志，启用测试模式

### 2. 完整工作流程（MindSpider/main.py）

**方法**: `run_complete_workflow()` (第 279-305 行)

**执行内容**:
1. 记录工作流程开始信息
2. 设置目标日期（默认为今天）
3. 记录配置信息（日期、平台、测试模式）

#### 第一步：话题提取

**调用**: `run_broad_topic_extraction()` (第 196-229 行)

**执行流程**:
1. 构建子进程命令：
   ```python
   cmd = [
       sys.executable, "main.py",
       "--keywords", str(keywords_count)  # 默认 100
   ]
   ```

2. 在 `BroadTopicExtraction` 目录下执行命令
3. 设置超时时间：1800 秒（30 分钟）
4. 等待子进程完成

**详细步骤**（参考 `broad-topic-execution-flow.md`）:

1. **收集热点新闻**:
   - 从 12 个新闻源获取今日热点
   - 通过 API 异步获取新闻数据
   - 保存到 `daily_news` 表（覆盖模式）

2. **AI 提取关键词和总结**:
   - 使用 DeepSeek AI 分析收集的新闻
   - 提取最多 100 个关键词（可配置）
   - 生成 150-300 字的新闻分析总结

3. **保存到数据库**:
   - 将关键词和总结保存到 `daily_topics` 表
   - 生成搜索关键词并保存到 `data/daily_keywords.txt` 文件

**验证**:
- 如果话题提取失败，终止整个流程
- 如果成功，继续执行第二步

#### 第二步：情感爬取

**调用**: `run_deep_sentiment_crawling()` (第 231-277 行)

**执行流程**:
1. 构建子进程命令：
   ```python
   cmd = [sys.executable, "main.py"]
   
   if target_date:
       cmd.extend(["--date", target_date.strftime("%Y-%m-%d")])
   
   if platforms:
       cmd.extend(["--platforms"] + platforms)
   
   cmd.extend([
       "--max-keywords", str(max_keywords),  # 默认 50
       "--max-notes", str(max_notes)          # 默认 50
   ])
   
   if test_mode:  # --test 参数
       cmd.append("--test")
   ```

2. 在 `DeepSentimentCrawling` 目录下执行命令
3. 设置超时时间：3600 秒（60 分钟）
4. 等待子进程完成

**详细步骤**（参考 `deep-sentiment-execution-flow.md`）:

1. **获取关键词摘要**:
   - 从数据库 `daily_topics` 表获取指定日期的话题数据
   - 检查是否有可用数据（使用第一步生成的数据）

2. **获取关键词列表**:
   - 从 `daily_topics` 表获取关键词
   - 如果当天没有数据，获取最近 7 天的数据
   - **测试模式限制**：最多 10 个关键词

3. **执行全平台爬取**:
   - 对每个平台（xhs、dy、ks、bili、wb、tieba、zhihu）执行爬取
   - 配置 MediaCrawler 使用 MindSpider 的数据库
   - 为每个平台创建爬取配置
   - 调用 MediaCrawler 进行实际爬取
   - **测试模式限制**：每个关键词在每个平台最多爬取 10 条内容

**验证**:
- 如果情感爬取失败，记录错误但话题提取已完成
- 如果成功，完成整个工作流程

### 3. 测试模式特性

当使用 `--test` 参数时，只影响第二步（情感爬取）的行为：

#### 测试模式参数限制

在 `DeepSentimentCrawling/main.py` 中（第 241-245 行）:
```python
if args.test:
    args.max_keywords = min(args.max_keywords, 10)  # 限制为最多10个
    args.max_notes = min(args.max_notes, 10)       # 限制为最多10个
    print("测试模式：限制关键词和内容数量")
```

**影响范围**:
- **关键词数量**：最多 10 个（即使指定更多也会被限制）
- **内容数量**：每个关键词在每个平台最多 10 条

**不影响**:
- 第一步（话题提取）不受测试模式影响，仍然提取最多 100 个关键词

#### 测试模式用途

1. **快速验证功能**:
   - 验证完整工作流程是否正常
   - 测试两个模块之间的数据传递

2. **测试登录**:
   - 验证各平台的登录是否成功
   - 减少等待时间

3. **验证配置**:
   - 验证数据库连接和配置
   - 验证 MediaCrawler 配置是否正确

4. **减少执行时间**:
   - 避免长时间等待
   - 快速获得反馈

## 完整执行流程对比

### 正常模式 vs 测试模式

| 阶段 | 正常模式 | 测试模式 |
|------|---------|---------|
| **第一步：话题提取** | | |
| 关键词提取数量 | 100（默认） | 100（不受影响） |
| 新闻源数量 | 12 个 | 12 个（不受影响） |
| **第二步：情感爬取** | | |
| 关键词使用数量 | 50（默认） | **10（限制）** |
| 每个关键词爬取数量 | 50（默认） | **10（限制）** |
| 平台数量 | 7 个（全部） | 7 个（全部，不受影响） |

### 执行时间对比

| 模式 | 预计耗时 |
|------|---------|
| 正常模式 | 30-90 分钟 |
| 测试模式 | 10-20 分钟 |

## 数据流转

```
┌─────────────────────────────────────────┐
│ 第一步：BroadTopicExtraction              │
├─────────────────────────────────────────┤
│ 输入: 无（从新闻 API 获取）                │
│ 处理: 收集新闻 → AI 提取关键词            │
│ 输出: daily_news 表 + daily_topics 表    │
└─────────────────────────────────────────┘
              ↓
    [数据库：daily_topics 表]
              ↓
┌─────────────────────────────────────────┐
│ 第二步：DeepSentimentCrawling            │
├─────────────────────────────────────────┤
│ 输入: daily_topics 表（关键词）          │
│ 处理: 获取关键词 → 多平台爬取             │
│ 输出: 各平台内容表（xhs_notes, dy_notes等）│
└─────────────────────────────────────────┘
```

## 依赖关系

### 模块依赖

1. **DeepSentimentCrawling 依赖 BroadTopicExtraction**:
   - 必须先执行第一步生成关键词
   - 从 `daily_topics` 表读取关键词数据
   - 如果第一步失败，第二步无法执行

2. **数据依赖**:
   - `daily_news` 表：存储新闻数据
   - `daily_topics` 表：存储关键词和总结（第一步生成，第二步使用）
   - 各平台内容表：存储爬取的内容（第二步生成）

### 外部依赖

1. **数据库**:
   - MySQL 或 PostgreSQL
   - 需要 `daily_news` 和 `daily_topics` 表

2. **API 服务**:
   - 新闻 API: `https://newsnow.busiyi.world`
   - AI API: DeepSeek API（或兼容 OpenAI 的 API）

3. **MediaCrawler**:
   - 位于 `DeepSentimentCrawling/MediaCrawler` 目录
   - 需要 Playwright 浏览器环境

4. **Python 包**:
   - `pymysql` 或 `psycopg`（数据库驱动）
   - `httpx`（HTTP 客户端）
   - `openai`（AI API 客户端）
   - `sqlalchemy`（数据库 ORM）
   - `loguru`（日志）
   - `pydantic-settings`（配置管理）
   - `playwright`（浏览器自动化）

## 执行时间

### 超时设置

- **第一步（话题提取）**: 30 分钟（1800 秒）
- **第二步（情感爬取）**: 60 分钟（3600 秒）
- **总超时时间**: 90 分钟（理论上限）

### 实际耗时

#### 正常模式
- **第一步**:
  - 新闻收集：1-5 分钟
  - AI 关键词提取：10-30 秒
  - 数据库操作：几秒钟
  - **总计**：约 2-6 分钟

- **第二步**:
  - 关键词获取：几秒钟
  - 平台爬取：取决于关键词数量和平台数量
  - 单个平台单个关键词：1-3 分钟
  - 7 个平台 × 50 个关键词：约 20-60 分钟
  - **总计**：约 25-70 分钟

- **完整流程**：约 30-90 分钟

#### 测试模式
- **第一步**：约 2-6 分钟（不受影响）
- **第二步**：
  - 7 个平台 × 10 个关键词：约 5-15 分钟
  - **总计**：约 7-21 分钟

- **完整流程**：约 10-25 分钟

## 错误处理

### 第一步失败

**情况**: 话题提取失败

**处理**:
- 记录错误日志
- **终止整个流程**（不执行第二步）
- 返回 `False`

**可能原因**:
- 新闻 API 不可用
- AI API 调用失败
- 数据库连接失败
- 网络问题

### 第二步失败

**情况**: 情感爬取失败

**处理**:
- 记录错误日志
- **但第一步已完成**（关键词已保存到数据库）
- 返回 `False`

**可能原因**:
- 没有找到关键词数据（第一步未执行或失败）
- MediaCrawler 配置错误
- 平台登录失败
- 数据库连接失败
- 网络问题

### 部分失败

**情况**: 单个平台或单个关键词爬取失败

**处理**:
- 记录错误信息
- 继续执行其他平台/关键词
- 在最终报告中统计成功/失败数量
- 不终止整个流程

## 数据存储

### MindSpider 数据库表

1. **daily_news 表**（第一步生成）:
   - 存储每日收集的新闻数据
   - 字段：news_id, source_platform, title, url, crawl_date, rank_position 等

2. **daily_topics 表**（第一步生成，第二步使用）:
   - 存储话题分析结果
   - 字段：extract_date, topic_id, keywords（JSON）, topic_description 等

### MediaCrawler 数据库表（第二步生成）

MediaCrawler 会在数据库中创建平台特定的表来存储爬取的内容：

- `xhs_notes`: 小红书笔记
- `xhs_comments`: 小红书评论
- `dy_notes`: 抖音视频
- `dy_comments`: 抖音评论
- `wb_notes`: 微博内容
- `wb_comments`: 微博评论
- `bili_notes`: B站视频
- `bili_comments`: B站评论
- `ks_notes`: 快手视频
- `ks_comments`: 快手评论
- `tieba_notes`: 贴吧帖子
- `tieba_comments`: 贴吧评论
- `zhihu_notes`: 知乎回答
- `zhihu_comments`: 知乎评论

## 注意事项

### 首次使用

1. **环境准备**:
   - 确保数据库已初始化（运行 `python main.py --init-db`）
   - 确保配置了正确的 API 密钥和数据库信息
   - 安装 Playwright 浏览器：`playwright install`

2. **执行顺序**:
   - 建议先用 `--test` 模式测试完整流程
   - 确认功能正常后再使用正常模式

3. **登录准备**:
   - 首次使用需要扫码登录各平台
   - 登录信息会保存，下次使用无需重新登录

### 测试模式建议

1. **快速验证**:
   - 使用 `--test` 模式快速验证完整流程
   - 确认两个模块之间的数据传递正常

2. **功能测试**:
   - 测试登录是否正常
   - 验证数据库配置是否正确
   - 检查数据是否正确保存

3. **性能测试**:
   - 测试模式可以快速获得反馈
   - 避免长时间等待

### 正常模式建议

1. **合理设置参数**:
   - 根据需求调整关键词数量
   - 根据网络情况调整爬取数量
   - 避免设置过大导致超时

2. **监控执行**:
   - 关注日志输出
   - 监控数据库存储空间
   - 注意 API 调用频率限制

3. **错误处理**:
   - 如果第一步失败，检查新闻 API 和 AI API
   - 如果第二步失败，检查 MediaCrawler 配置和登录状态

### 其他注意事项

1. **API 限制**:
   - 注意新闻 API 和 AI API 的调用频率限制
   - 合理控制爬取频率，避免被平台限制

2. **数据库空间**:
   - 确保数据库有足够的存储空间
   - 定期清理历史数据（如果需要）

3. **网络要求**:
   - 需要稳定的网络连接
   - 某些平台可能需要科学上网

4. **日期处理**:
   - 默认处理今天的新闻
   - 可通过 `--date` 参数指定其他日期
   - 注意：如果指定过去的日期，需要确保该日期有话题数据

## 执行示例

### 测试模式示例

```bash
# 基本测试（所有平台，测试模式）
python main.py --complete --test

# 测试指定平台
python main.py --complete --test --platforms xhs dy

# 测试指定日期
python main.py --complete --test --date 2024-01-01

# 测试 + 自定义关键词数量（第一步不受影响，第二步会被限制）
python main.py --complete --test --keywords-count 50 --max-keywords 20
```

### 正常模式示例

```bash
# 所有平台，默认参数
python main.py --complete

# 指定平台和数量
python main.py --complete --platforms xhs dy --max-keywords 30 --max-notes 40

# 指定日期
python main.py --complete --date 2024-01-01
```

### 分步执行示例

如果完整流程失败，可以分步执行：

```bash
# 第一步：话题提取
python main.py --broad-topic

# 第二步：情感爬取（测试模式）
python main.py --deep-sentiment --test
```

## 相关文档

- [broad-topic-execution-flow.md](./broad-topic-execution-flow.md): 话题提取模块详细流程
- [deep-sentiment-execution-flow.md](./deep-sentiment-execution-flow.md): 情感爬取模块详细流程

## 相关文件

- `MindSpider/main.py`: 主程序入口
- `BroadTopicExtraction/main.py`: 话题提取模块主程序
- `DeepSentimentCrawling/main.py`: 深度情感爬取模块主程序
- `config.py`: 配置文件
- `schema/mindspider_tables.sql`: 数据库表结构定义

---

**文档生成时间**: 2024年
**分析基于**: MindSpider 项目代码库
