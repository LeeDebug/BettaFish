# `python main.py --deep-sentiment --test` 执行流程分析

## 概述

当用户执行 `python main.py --deep-sentiment --test` 命令时，MindSpider 会运行深度情感爬取模块（DeepSentimentCrawling），从数据库中获取由 BroadTopicExtraction 模块提取的关键词，然后在多个社交媒体平台上进行内容爬取。`--test` 参数启用测试模式，限制爬取数量以便快速验证功能。

## 执行流程图

```
用户执行命令
    ↓
MindSpider/main.py 解析参数
    ↓
调用 run_deep_sentiment_crawling(test_mode=True)
    ↓
在 DeepSentimentCrawling 目录下执行子进程
    ↓
DeepSentimentCrawling/main.py 启动
    ↓
解析 --test 参数，调整爬取限制
    ↓
创建 DeepSentimentCrawling 实例
    ↓
执行 run_daily_crawling() 流程
    ├─ 步骤1: 获取关键词摘要 (get_crawling_summary)
    ├─ 步骤2: 获取关键词列表 (get_latest_keywords)
    └─ 步骤3: 执行全平台爬取 (run_multi_platform_crawl_by_keywords)
        ├─ 配置 MediaCrawler 数据库
        ├─ 为每个平台创建配置
        └─ 调用 MediaCrawler 进行爬取
    ↓
生成爬取报告
    ↓
返回执行结果
```

## 详细执行步骤

### 1. 命令入口（MindSpider/main.py）

**位置**: `main.py` 第 423-426 行

```python
elif args.deep_sentiment:
    spider.run_deep_sentiment_crawling(
        target_date, args.platforms, args.max_keywords, args.max_notes, args.test
    )
```

**参数说明**:
- `target_date`: 如果未指定 `--date`，默认为今天
- `args.platforms`: 如果未指定 `--platforms`，默认为所有支持的平台
- `args.max_keywords`: 如果未指定 `--max-keywords`，默认为 50
- `args.max_notes`: 如果未指定 `--max-notes`，默认为 50
- `args.test`: `--test` 标志，启用测试模式

### 2. 主程序调用（MindSpider/main.py）

**方法**: `run_deep_sentiment_crawling()` (第 231-277 行)

**执行内容**:
1. 构建子进程命令：
   ```python
   cmd = [sys.executable, "main.py"]
   
   if target_date:
       cmd.extend(["--date", target_date.strftime("%Y-%m-%d")])
   
   if platforms:
       cmd.extend(["--platforms"] + platforms)
   
   cmd.extend([
       "--max-keywords", str(max_keywords),
       "--max-notes", str(max_notes)
   ])
   
   if test_mode:
       cmd.append("--test")
   ```

2. 在 `DeepSentimentCrawling` 目录下执行命令
3. 设置超时时间：3600 秒（60 分钟）
4. 等待子进程完成并返回结果

### 3. 子模块启动（DeepSentimentCrawling/main.py）

**入口函数**: `main()` (第 190-281 行)

**参数解析**:
- `--date`: 目标日期 (YYYY-MM-DD)
- `--platform`: 指定单个平台
- `--platforms`: 指定多个平台
- `--max-keywords`: 每个平台最大关键词数量（默认 50）
- `--max-notes`: 每个平台最大爬取内容数量（默认 50）
- `--login-type`: 登录方式（默认 qrcode）
- `--test`: 测试模式标志

**测试模式处理** (第 241-245 行):
```python
if args.test:
    args.max_keywords = min(args.max_keywords, 10)  # 限制为最多10个
    args.max_notes = min(args.max_notes, 10)       # 限制为最多10个
    print("测试模式：限制关键词和内容数量")
```

**执行流程**:
1. 解析命令行参数
2. 如果是测试模式，调整参数限制
3. 创建 `DeepSentimentCrawling` 实例
4. 调用 `run_daily_crawling()` 执行核心流程

### 4. 核心爬取流程（DeepSentimentCrawling/main.py）

**方法**: `run_daily_crawling()` (第 30-97 行)

#### 步骤 1: 获取关键词摘要

**调用**: `keyword_manager.get_crawling_summary()`

**执行内容** (`keyword_manager.py`):
1. 从数据库 `daily_topics` 表获取指定日期的话题数据
2. 检查是否有数据
3. 返回摘要信息：
   ```python
   {
       'date': target_date,
       'keywords_count': 关键词数量,
       'summary': 话题总结,
       'has_data': True/False
   }
   ```

**验证**:
- 如果没有数据，返回错误并终止流程

#### 步骤 2: 获取关键词列表

**调用**: `keyword_manager.get_latest_keywords()`

**执行内容** (`keyword_manager.py` 第 60-111 行):
1. **优先获取指定日期的关键词**:
   - 从 `daily_topics` 表查询指定日期的话题数据
   - 解析 JSON 格式的关键词列表

2. **如果当天没有数据，获取最近的关键词**:
   - 查询最近 7 天的话题数据
   - 合并所有关键词并去重
   - 随机选择指定数量的关键词

3. **如果都没有，使用默认关键词**:
   - 返回预定义的默认关键词列表（科技、AI、编程等）

4. **限制关键词数量**:
   - 如果关键词数量超过 `max_keywords`，随机选择指定数量
   - 测试模式下最多 10 个关键词

**返回结果**: 关键词列表（List[str]）

#### 步骤 3: 执行全平台关键词爬取

**调用**: `platform_crawler.run_multi_platform_crawl_by_keywords()`

**执行内容** (`platform_crawler.py` 第 333-447 行):

##### 3.1 初始化统计信息

```python
total_stats = {
    "total_keywords": len(keywords),
    "total_platforms": len(platforms),
    "total_tasks": len(keywords) * len(platforms),
    "successful_tasks": 0,
    "failed_tasks": 0,
    "total_notes": 0,
    "total_comments": 0,
    "keyword_results": {},
    "platform_summary": {}
}
```

##### 3.2 对每个平台执行爬取

**支持的平台**:
- `xhs`: 小红书
- `dy`: 抖音
- `ks`: 快手
- `bili`: B站
- `wb`: 微博
- `tieba`: 百度贴吧
- `zhihu`: 知乎

**对每个平台执行** (`run_crawler()` 方法，第 203-293 行):

1. **配置 MediaCrawler 数据库** (`configure_mediacrawler_db()`):
   - 读取 MediaCrawler 的数据库配置文件
   - 根据 MindSpider 的数据库配置（MySQL 或 PostgreSQL）更新配置
   - 写入新的数据库配置到 `MediaCrawler/config/db_config.py`

2. **创建基础配置** (`create_base_config()`):
   - 读取 `MediaCrawler/config/base_config.py`
   - 修改关键配置项：
     - `PLATFORM`: 平台名称
     - `KEYWORDS`: 关键词列表（逗号分隔）
     - `CRAWLER_TYPE`: "search"（关键词搜索）
     - `SAVE_DATA_OPTION`: "db"（MySQL）或 "postgresql"（PostgreSQL）
     - `CRAWLER_MAX_NOTES_COUNT`: 最大爬取数量（测试模式下为 10）
     - `ENABLE_GET_COMMENTS`: True（启用评论爬取）
     - `CRAWLER_MAX_COMMENTS_COUNT_SINGLENOTES`: 20（每条内容最多 20 条评论）
     - `HEADLESS`: True（无头模式）

3. **执行 MediaCrawler**:
   - 构建命令：
     ```python
     cmd = [
         sys.executable, "main.py",
         "--platform", platform,
         "--lt", login_type,  # 登录类型：qrcode/phone/cookie
         "--type", "search",
         "--save_data_option", save_data_option
     ]
     ```
   - 在 `MediaCrawler` 目录下执行命令
   - 超时时间：3600 秒（60 分钟）

4. **收集统计信息**:
   - 记录爬取耗时
   - 记录返回码
   - 判断成功/失败状态

##### 3.3 汇总统计结果

- 统计成功/失败的任务数
- 统计总内容数和评论数
- 生成各平台的统计摘要
- 生成每个关键词在各平台的爬取结果

### 5. MediaCrawler 执行流程

MediaCrawler 是一个独立的爬虫框架，负责实际的平台内容爬取：

1. **登录验证**:
   - 根据 `--lt` 参数选择登录方式
   - `qrcode`: 二维码登录（默认）
   - `phone`: 手机号登录
   - `cookie`: Cookie 登录

2. **关键词搜索**:
   - 在每个平台上搜索配置的关键词
   - 获取搜索结果列表

3. **内容爬取**:
   - 爬取每个关键词的搜索结果
   - 限制每个关键词的爬取数量（测试模式下为 10）
   - 爬取内容的详细信息（标题、作者、内容、点赞数等）

4. **评论爬取**:
   - 如果启用，爬取每条内容的评论（最多 20 条）

5. **数据保存**:
   - 根据 `SAVE_DATA_OPTION` 配置保存到数据库
   - 保存到 MindSpider 的数据库（MySQL 或 PostgreSQL）

### 6. 结果输出

**输出内容**:
1. **爬取摘要**:
   - 日期
   - 关键词数量
   - 话题总结

2. **爬取统计**:
   - 总任务数（关键词数 × 平台数）
   - 成功任务数/失败任务数
   - 总内容数
   - 总评论数
   - 成功率

3. **各平台统计**:
   - 每个平台的成功关键词数
   - 每个平台的内容数
   - 每个平台的评论数

4. **详细结果**:
   - 每个关键词在各平台的爬取结果
   - 错误信息（如果有）

## 测试模式特性

当使用 `--test` 参数时：

1. **参数限制**:
   - `max_keywords`: 最多 10 个（即使指定更多也会被限制）
   - `max_notes`: 最多 10 个（每个关键词在每个平台最多爬取 10 条内容）

2. **用途**:
   - 快速验证功能是否正常
   - 测试登录是否成功
   - 验证数据库连接和配置
   - 减少爬取时间，避免长时间等待

3. **示例**:
   ```bash
   # 测试模式：最多 10 个关键词，每个关键词最多 10 条内容
   python main.py --deep-sentiment --test
   
   # 测试模式 + 指定平台
   python main.py --deep-sentiment --test --platforms xhs dy
   
   # 测试模式 + 自定义数量（会被限制为最多 10）
   python main.py --deep-sentiment --test --max-keywords 20 --max-notes 20
   ```

## 依赖关系

### 前置依赖
1. **BroadTopicExtraction 模块**:
   - 必须先运行 `python main.py --broad-topic` 生成关键词
   - 关键词存储在 `daily_topics` 表中

2. **数据库**:
   - MySQL 或 PostgreSQL
   - 需要 `daily_topics` 表（存储关键词）
   - MediaCrawler 会创建自己的表来存储爬取的内容

3. **MediaCrawler**:
   - 位于 `DeepSentimentCrawling/MediaCrawler` 目录
   - 需要安装其依赖（requirements.txt）
   - 需要 Playwright 浏览器环境

### 外部依赖
1. **Python 包**:
   - `pymysql` 或 `psycopg`（数据库驱动）
   - `sqlalchemy`（数据库 ORM）
   - `loguru`（日志）
   - `playwright`（浏览器自动化）

2. **浏览器环境**:
   - Playwright 浏览器（需要运行 `playwright install`）

3. **平台账号**:
   - 某些平台需要登录才能爬取
   - 首次使用需要扫码登录

## 执行时间

- **超时设置**: 60 分钟（3600 秒）
- **实际耗时**:
  - 测试模式：通常 5-15 分钟（取决于平台数量和网络速度）
  - 正常模式：取决于关键词数量、平台数量和爬取内容数量
  - 单个平台单个关键词：通常 1-3 分钟

## 错误处理

1. **关键词获取失败**:
   - 如果当天没有数据，尝试获取最近 7 天的数据
   - 如果都没有，使用默认关键词列表
   - 如果仍然失败，终止流程

2. **平台爬取失败**:
   - 单个平台失败不影响其他平台
   - 记录错误信息到统计结果
   - 继续执行下一个平台

3. **登录失败**:
   - MediaCrawler 会提示需要登录
   - 用户需要手动扫码登录
   - 登录信息会保存，下次使用无需重新登录

4. **数据库连接失败**:
   - 记录错误日志
   - 终止当前平台的爬取
   - 继续执行下一个平台

## 数据存储

### MindSpider 数据库表

1. **daily_topics 表**:
   - 存储话题分析结果（由 BroadTopicExtraction 生成）
   - 包含关键词列表和总结

### MediaCrawler 数据库表

MediaCrawler 会在数据库中创建平台特定的表来存储爬取的内容：

- `xhs_notes`: 小红书笔记
- `xhs_comments`: 小红书评论
- `dy_notes`: 抖音视频
- `dy_comments`: 抖音评论
- `wb_notes`: 微博内容
- `wb_comments`: 微博评论
- 等等...

## 注意事项

1. **首次使用**:
   - 需要先运行 `python main.py --broad-topic` 生成关键词
   - 需要安装 Playwright 浏览器：`playwright install`
   - 首次登录需要扫码，登录信息会保存

2. **测试模式建议**:
   - 首次使用建议先用 `--test` 模式测试
   - 确认登录正常后再使用正常模式
   - 测试模式可以减少爬取时间和数据量

3. **平台限制**:
   - 某些平台有反爬虫机制，可能限制爬取频率
   - 建议合理设置爬取数量，避免被限制
   - 如果遇到限制，等待一段时间后重试

4. **数据库配置**:
   - MediaCrawler 的数据库配置会自动从 MindSpider 配置中读取
   - 确保数据库连接信息正确
   - 确保数据库有足够的存储空间

5. **网络要求**:
   - 需要稳定的网络连接
   - 某些平台可能需要科学上网

6. **登录方式**:
   - 推荐使用 `qrcode`（二维码登录）
   - 某些平台可能不支持手机号登录
   - Cookie 登录需要手动获取 Cookie

## 相关文件

- `MindSpider/main.py`: 主程序入口
- `DeepSentimentCrawling/main.py`: 深度情感爬取模块主程序
- `DeepSentimentCrawling/keyword_manager.py`: 关键词管理器
- `DeepSentimentCrawling/platform_crawler.py`: 平台爬虫管理器
- `DeepSentimentCrawling/MediaCrawler/`: MediaCrawler 爬虫框架
- `config.py`: 配置文件
- `schema/mindspider_tables.sql`: 数据库表结构定义

## 执行示例

### 测试模式示例

```bash
# 基本测试（所有平台，测试模式）
python main.py --deep-sentiment --test

# 测试指定平台
python main.py --deep-sentiment --test --platforms xhs dy

# 测试指定日期
python main.py --deep-sentiment --test --date 2024-01-01

# 测试 + 自定义登录方式
python main.py --deep-sentiment --test --login-type cookie
```

### 正常模式示例

```bash
# 所有平台，默认参数
python main.py --deep-sentiment

# 指定平台和数量
python main.py --deep-sentiment --platforms xhs dy --max-keywords 20 --max-notes 30
```

---

**文档生成时间**: 2024年
**分析基于**: MindSpider 项目代码库
