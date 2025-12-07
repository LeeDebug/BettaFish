# `python main.py --broad-topic` 执行流程分析

## 概述

当用户执行 `python main.py --broad-topic` 命令时，MindSpider 会运行话题提取模块（BroadTopicExtraction），从多个新闻源收集热点新闻，使用 AI 提取关键词并生成分析总结，最后将结果保存到数据库。

## 执行流程图

```
用户执行命令
    ↓
MindSpider/main.py 解析参数
    ↓
调用 run_broad_topic_extraction()
    ↓
在 BroadTopicExtraction 目录下执行子进程
    ↓
BroadTopicExtraction/main.py 启动
    ↓
创建 BroadTopicExtraction 实例
    ↓
执行 run_daily_extraction() 流程
    ├─ 步骤1: 收集新闻 (collect_and_save_news)
    ├─ 步骤2: 提取关键词和总结 (extract_keywords_and_summary)
    └─ 步骤3: 保存到数据库 (save_daily_topics)
    ↓
生成搜索关键词并保存到文件
    ↓
返回执行结果
```

## 详细执行步骤

### 1. 命令入口（MindSpider/main.py）

**位置**: `main.py` 第 421-422 行

```python
if args.broad_topic:
    spider.run_broad_topic_extraction(target_date, args.keywords_count)
```

**默认参数**:
- `target_date`: 如果未指定 `--date`，默认为今天
- `keywords_count`: 如果未指定 `--keywords-count`，默认为 100

### 2. 主程序调用（MindSpider/main.py）

**方法**: `run_broad_topic_extraction()` (第 196-229 行)

**执行内容**:
1. 构建子进程命令：
   ```python
   cmd = [
       sys.executable, "main.py",
       "--keywords", str(keywords_count)
   ]
   ```
2. 在 `BroadTopicExtraction` 目录下执行命令
3. 设置超时时间：1800 秒（30 分钟）
4. 等待子进程完成并返回结果

### 3. 子模块启动（BroadTopicExtraction/main.py）

**入口函数**: `main()` (第 287-325 行)

**参数解析**:
- `--sources`: 指定新闻源平台（可选）
- `--keywords`: 最大关键词数量（默认 100）
- `--quiet`: 简化输出模式（可选）
- `--list-sources`: 显示支持的新闻源（可选）

**执行流程**:
1. 解析命令行参数
2. 调用 `run_extraction_command()` 异步函数

### 4. 话题提取流程（BroadTopicExtraction/main.py）

**函数**: `run_extraction_command()` (第 236-285 行)

**执行步骤**:
1. 创建 `BroadTopicExtraction` 实例（使用上下文管理器）
2. 调用 `run_daily_extraction()` 执行核心流程
3. 打印提取结果
4. 获取搜索关键词并保存到文件

### 5. 核心提取流程（BroadTopicExtraction/main.py）

**方法**: `run_daily_extraction()` (第 59-154 行)

#### 步骤 1: 收集热点新闻

**调用**: `news_collector.collect_and_save_news()`

**执行内容** (`get_today_news.py`):
1. **新闻源选择**:
   - 如果未指定 `sources`，使用所有支持的新闻源（12 个平台）
   - 支持的新闻源包括：
     - `weibo`: 微博热搜
     - `zhihu`: 知乎热榜
     - `bilibili-hot-search`: B站热搜
     - `toutiao`: 今日头条
     - `douyin`: 抖音热榜
     - `github-trending-today`: GitHub趋势
     - `coolapk`: 酷安热榜
     - `tieba`: 百度贴吧
     - `wallstreetcn`: 华尔街见闻
     - `thepaper`: 澎湃新闻
     - `cls-hot`: 财联社
     - `xueqiu`: 雪球热榜

2. **API 调用**:
   - 基础 URL: `https://newsnow.busiyi.world`
   - 对每个新闻源调用: `/api/s?id={source}&latest`
   - 使用异步 HTTP 客户端（httpx）并发请求
   - 请求间隔：0.5 秒（避免请求过快）

3. **数据处理**:
   - 解析 JSON 响应
   - 提取新闻标题、URL、来源、排名等信息
   - 生成唯一新闻 ID（格式：`{source}_{item_id}_{date}`）

4. **数据库保存** (`database_manager.py`):
   - 表名: `daily_news`
   - 保存模式：覆盖模式（先删除当天已有数据，再插入新数据）
   - 保存字段：
     - `news_id`: 新闻唯一标识
     - `source_platform`: 新闻来源平台
     - `title`: 新闻标题（最大 500 字符）
     - `url`: 新闻链接
     - `crawl_date`: 爬取日期
     - `rank_position`: 排名位置
     - `add_ts`: 添加时间戳
     - `last_modify_ts`: 最后修改时间戳

**返回结果**:
```python
{
    'success': True/False,
    'news_list': [...],  # 新闻列表
    'total_news': 0,     # 总新闻数
    'successful_sources': 0,  # 成功源数
    'total_sources': 0,  # 总源数
    'collection_time': '...'  # 收集时间
}
```

#### 步骤 2: 提取关键词和生成总结

**调用**: `topic_extractor.extract_keywords_and_summary()`

**执行内容** (`topic_extractor.py`):
1. **构建新闻摘要文本**:
   - 将收集到的新闻列表格式化为文本
   - 格式：`{序号}. 【{来源}】{标题}`

2. **构建 AI 提示词**:
   - 任务 1：提取关键词（最多 `max_keywords` 个）
     - 要求：适合社交媒体搜索、热度高、讨论量大
   - 任务 2：撰写新闻分析总结（150-300 字）
     - 要求：概括主要内容、指出关注重点、分析社会现象

3. **调用 AI API**:
   - API 客户端：OpenAI 兼容接口
   - 模型：`settings.MINDSPIDER_MODEL_NAME`（默认：deepseek-chat）
   - API 基础 URL：`settings.MINDSPIDER_BASE_URL`（默认：https://api.deepseek.com）
   - API Key：`settings.MINDSPIDER_API_KEY`
   - 参数：
     - `max_tokens`: 1500
     - `temperature`: 0.3

4. **解析 AI 返回结果**:
   - 期望格式：JSON
     ```json
     {
       "keywords": ["关键词1", "关键词2", ...],
       "summary": "新闻分析总结..."
     }
     ```
   - 如果 JSON 解析失败，使用手动解析（fallback）
   - 验证和清理关键词（去重、过滤无效词）

5. **错误处理**:
   - 如果 AI 调用失败，使用简单关键词提取（fallback）
   - 从新闻标题中提取关键词

**返回结果**:
```python
(keywords: List[str], summary: str)
```

#### 步骤 3: 保存到数据库

**调用**: `db_manager.save_daily_topics()`

**执行内容** (`database_manager.py`):
1. **数据准备**:
   - 将关键词列表转换为 JSON 字符串
   - 生成话题 ID：`summary_{date}`（格式：`summary_YYYYMMDD`）

2. **数据库操作**:
   - 表名: `daily_topics`
   - 检查当天是否已有记录
   - 如果存在：执行 UPDATE 操作
   - 如果不存在：执行 INSERT 操作
   - 保存字段：
     - `extract_date`: 提取日期
     - `topic_id`: 话题唯一标识
     - `topic_name`: 话题名称（固定为"每日新闻分析"）
     - `keywords`: 关键词 JSON 字符串
     - `topic_description`: 新闻分析总结
     - `add_ts`: 添加时间戳
     - `last_modify_ts`: 最后修改时间戳

**返回结果**: `True`（成功）或 `False`（失败）

### 6. 生成搜索关键词（BroadTopicExtraction/main.py）

**方法**: `get_keywords_for_crawling()` (第 188-216 行)

**执行内容**:
1. 从数据库获取当天的话题数据
2. 调用 `topic_extractor.get_search_keywords()` 处理关键词
3. 过滤条件：
   - 长度：1-20 字符
   - 不能是纯数字
   - 不能是纯英文（除非是专有名词）
   - 去重
4. 限制数量：默认最多 10 个

**保存到文件**:
- 文件路径: `data/daily_keywords.txt`
- 格式：每行一个关键词

### 7. 结果输出

**输出内容**:
1. **新闻收集结果**:
   - 总新闻数
   - 成功源数/总源数

2. **话题提取结果**:
   - 提取的关键词数量
   - 关键词列表（每行显示 5 个）
   - 新闻分析总结

3. **数据库保存结果**:
   - 保存成功/失败状态

4. **搜索关键词**:
   - 为 DeepSentimentCrawling 准备的搜索关键词
   - 关键词文件保存路径

## 依赖关系

### 外部依赖
1. **数据库**:
   - MySQL 或 PostgreSQL
   - 需要 `daily_news` 和 `daily_topics` 表

2. **API 服务**:
   - 新闻 API: `https://newsnow.busiyi.world`
   - AI API: DeepSeek API（或兼容 OpenAI 的 API）

3. **Python 包**:
   - `pymysql` 或 `psycopg`（数据库驱动）
   - `httpx`（HTTP 客户端）
   - `openai`（AI API 客户端）
   - `sqlalchemy`（数据库 ORM）
   - `loguru`（日志）
   - `pydantic-settings`（配置管理）

### 配置文件
- `config.py`: 数据库连接信息和 API 密钥
- 环境变量或 `.env` 文件

## 执行时间

- **超时设置**: 30 分钟（1800 秒）
- **实际耗时**:
  - 新闻收集：取决于新闻源数量和网络速度（通常 1-5 分钟）
  - AI 关键词提取：取决于新闻数量和 API 响应速度（通常 10-30 秒）
  - 数据库操作：通常几秒钟

## 错误处理

1. **新闻收集失败**:
   - 单个新闻源失败不影响其他源
   - 如果所有源都失败，流程终止

2. **AI 提取失败**:
   - 使用简单关键词提取作为 fallback
   - 生成默认总结文本

3. **数据库操作失败**:
   - 单条新闻保存失败不影响其他新闻
   - 话题保存失败会记录错误日志

## 输出文件

- `data/daily_keywords.txt`: 保存提取的关键词，供后续 DeepSentimentCrawling 模块使用

## 数据库表结构

### daily_news 表
存储每日收集的新闻数据

### daily_topics 表
存储每日话题分析结果（关键词和总结）

## 注意事项

1. **API 限制**: 注意新闻 API 和 AI API 的调用频率限制
2. **数据库连接**: 确保数据库连接正常，表结构已初始化
3. **配置检查**: 确保 `config.py` 或环境变量中配置了正确的 API 密钥和数据库信息
4. **网络要求**: 需要能够访问外部 API 服务
5. **日期处理**: 默认处理今天的新闻，可通过 `--date` 参数指定其他日期

## 相关文件

- `MindSpider/main.py`: 主程序入口
- `BroadTopicExtraction/main.py`: 话题提取模块主程序
- `BroadTopicExtraction/get_today_news.py`: 新闻收集器
- `BroadTopicExtraction/topic_extractor.py`: 话题提取器（AI）
- `BroadTopicExtraction/database_manager.py`: 数据库管理器
- `config.py`: 配置文件
- `schema/mindspider_tables.sql`: 数据库表结构定义

---

**文档生成时间**: 2024年
**分析基于**: MindSpider 项目代码库
