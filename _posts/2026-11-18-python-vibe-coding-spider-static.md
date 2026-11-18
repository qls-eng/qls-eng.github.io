---
layout: default
title: Vibe Coding 实操：静态网页爬虫（请求—解析—持久化—重构四轮迭代）
date: 2026-11-18
categories: 实验内容
tags: [Python, 爬虫, requests, BeautifulSoup, pandas, Vibe Coding]
---

# Vibe Coding 实操：静态网页爬虫（请求—解析—持久化—重构四轮迭代）

> 本文按《Vibe Coding 爬虫开发实验》完成一个静态网页爬虫：用结构化 Prompt 驱动 AI 分阶段产出代码，每阶段独立运行验证后再迭代，全程没有一次性生成完整爬虫；最后按指导书步骤 4 把整份代码喂回 AI 做审查并重构成类。完整工程与过程记录见文末源码仓库。

## 实验背景

这类实验的评分点不在"最终能不能跑出一个 CSV"，而在**开发过程是否可追溯**：需求有没有讲清楚、任务有没有拆开、每段代码有没有验证、出错后怎么定位。一次性让 AI 吐出一整份爬虫，表面很快，但一旦某个字段抓不到，就得在几十行代码里靠猜。

实验环境：Python 3.14、PyCharm、`requests` + `beautifulsoup4` + `pandas`，AI 编程助手为网页版对话模型。

目标站点选了 `books.toscrape.com`。这是一个公开的爬虫练习沙盒，页面纯静态、分页规则规整、明确允许练习抓取，不需要处理登录态、验证码和反爬对抗，适合把注意力放在工程流程本身。

---

## 一、需求澄清

### 1. 结构化 Prompt

```
你是 Python 后端工程师。我需要编写一个静态网页爬虫。
目标：抓取【目标网站 URL】的列表页，提取标题、时间、作者、链接。
约束：使用 requests 和 BeautifulSoup；必须设置 User-Agent 伪装；
请求间隔不低于 2 秒；遇到网络超时需重试 3 次。
验收：输出 result.csv 文件，并在终端打印成功抓取条数。
```

四要素缺一不可：

| 要素 | 内容 |
|---|---|
| 角色 | Python 后端工程师 |
| 目标 | 抓取静态列表页，提取 4 个字段 |
| 约束 | 技术栈固定、UA 伪装、间隔 ≥ 2 秒、超时重试 3 次 |
| 验收 | 输出 CSV，终端打印条数 |

### 2. 字段映射

练习站点的数据结构与新闻列表不同，按"等效抽取"做映射：

| 实验要求 | 本项目字段 | 取值位置 |
|---|---|---|
| 标题 | `title`（书名） | `h3 > a` 的 `title` 属性 |
| 时间 | `price`（价格） | `p.price_color` 文本 |
| 作者 | `rating`（星级） | `p.star-rating` 的 class 第二值 |
| 链接 | `link` | `h3 > a` 的 `href` 经 `urljoin` 转绝对地址 |

### 3. 明确不做的事

不碰登录态、不做验证码识别、不用代理池、不做详情页二次抓取、不做并发加速（与"间隔 ≥ 2 秒"的约束目标冲突）。范围先划死，AI 才不会顺手加戏。

---

## 二、前三轮的迭代过程（请求 → 解析 → 持久化）

### 阶段一：只请求，不解析

Prompt 里明确写"现在只生成发送请求并获取 HTML 的代码，不要解析、不要写入 csv"。AI 返回的结构是：

- `UA_POOL`：4 条不同浏览器 UA，`create_session()` 随机取一条
- `_throttle()`：以 `time.monotonic()` 记录上次请求时刻，不足 2 秒就补足
- `fetch_html()`：对 `Timeout`、`ConnectionError` 最多重试 3 次并递增退避；`HTTPError` 直接放弃
- `fetch_list_pages()`：生成器逐页 `yield`，失败页返回 `None`

```python
def _throttle():
    global _last_request_at
    elapsed = time.monotonic() - _last_request_at
    if elapsed < REQUEST_INTERVAL:
        time.sleep(REQUEST_INTERVAL - elapsed)
    _last_request_at = time.monotonic()
```

这一轮**实际跑挂了**：代码里直接用 `HTTPError`、`RequestException`，但没有从 `requests.exceptions` 导入，PyCharm 直接标红，运行必抛 `NameError`。追加一条"只补导入、不要新增功能"的 Prompt 修复后，5 页全部返回 `status=200`，请求层通过。

> 这是主链路三轮迭代里唯一一次返工，也是最有价值的一次：AI 给的代码"看起来完整"，但连导入都没自查。而且这个坑并没有被真正修掉——下面第三节的审查会指出它一直留着。

### 阶段二：加解析，只打印前 3 条

Prompt 约束"不要删除原有请求逻辑，不要写入 CSV，每页只打印前 3 条"。AI 新增 `parse_books()`，卡片定位用 `article.product_pod`，四个字段分别用 `select_one` 取，链接用 `urljoin` 拼绝对地址。

```python
for pod in soup.select("article.product_pod"):
    a = pod.select_one("h3 > a")
    title = a.get("title", "").strip() if a else ""
    href = a.get("href", "") if a else ""
    link = urljoin(base_url, href) if href else ""

    price_el = pod.select_one("p.price_color")
    price = price_el.get_text(strip=True) if price_el else ""

    rating_el = pod.select_one("p.star-rating")
    rating = ""
    if rating_el:
        classes = rating_el.get("class", [])
        if len(classes) > 1:
            rating = classes[1]
```

刻意要求"只打印前 3 条、不写文件"，是为了把"解析对不对"与"文件写没写对"两个问题隔离开。这一步跑通了，后面写盘出问题就只可能是写盘环节的原因。

运行结果：3 页 × 20 条，每页打印 3 条，字段与页面人工比对一致，未生成任何文件。

### 阶段三：加持久化，输出 CSV

Prompt 要求"原有请求、解析代码全部保留，不要改动"，只新增 pandas 汇总与写盘：

```python
all_books = []
for page, url, html in fetch_list_pages():
    if html is None:
        continue
    books = parse_books(html, url)
    all_books.extend(books)

df = pd.DataFrame(all_books, columns=["title", "price", "rating", "link"])
df.to_csv("result.csv", index=False, encoding="utf-8-sig")
```

两个参数是关键：`index=False` 去掉多余行号列，`utf-8-sig` 带 BOM，Excel 双击打开中文不乱码。运行后得到 61 行（表头 1 + 数据 60），列序与字段定义一致。

---

## 三、步骤 4：把整份代码喂回 AI 做重构（实验后补做）

指导书还有一步：把 AI 生成的全部代码再喂回 AI，让它指出潜在的性能瓶颈或安全隐患，重构为面向对象的类结构（如 `NewsCrawler` 类），并保持功能不变。

前三轮完成的是请求、解析、持久化主链路，**这一步在实验进行时并没有做**。补做时有个绕不开的问题：如果直接把代码改掉、再补一句"我重构过"，`docs/ai对话记录.md` 里的三轮记录就和代码、提交时间线对不上了——等于自己证明自己造假。所以这一轮作为**新的第四轮（R7）如实追加**，日期写 2026-11-18，开头明确标注"该步骤在实验进行时未执行，本轮为实验后补做"，前三轮内容一个字没动；git 提交信息也写成"补做步骤 4：重构为 NewsCrawler 类"。

### 1. 喂回去的 Prompt

```
请审查当前 fetcher.py，指出潜在的性能瓶颈或安全隐患，
并把全部代码重构为面向对象的类结构（类名 NewsCrawler），保持功能不变。

约束：
1. 不改抓取字段、不改 CSV 列顺序与编码、不改控制台打印内容；
2. 请求间隔 ≥2 秒、UA 池、超时重试 3 次的策略原样保留；
3. 重构后必须能跑出与重构前逐行一致的 result.csv。

验收：同一站点、同一页数运行，CSV 逐行比对一致。
```

### 2. AI 审查出来的五个问题

| 类别 | 问题 | 给出的建议 |
|---|---|---|
| 性能 | 每页重建 `Session` 的写法，TCP 连接无法稳定复用 | 用上下文管理持有连接池 |
| 隐患 | 模块级全局变量 `_last_request_at` 由 `global` 维护，多实例会互相串扰 | 收进实例属性，节流状态跟随实例 |
| 隐患 | `HTTPError`、`RequestException` 依赖隐式引用（阶段一正是因此抛 `NameError`） | 显式 `from requests.exceptions import ...` |
| 健壮性 | 页面级失败只打日志，没有统一汇总 | 由 `run()` 统一编排，失败页明确跳过 |
| 可读性 | 五个函数 + 主流程平铺，职责边界不清晰 | 按会话 / 请求 / 解析 / 持久化 / 编排分层 |

第二条和第三条值得单独记一笔：导入问题是阶段一**真实跑挂过**的那个坑，这说明同类问题不会因为多迭代几轮就自动消失，只要没人回头审查，它就一直躺在代码里。

### 3. 重构后的结构

整份代码收敛成一个 `NewsCrawler` 类，五个层次：

| 层次 | 成员 | 职责 |
|---|---|---|
| 会话层 | `__enter__` / `__exit__` / `close` / `_create_session` / `session` | 持有并复用 Session，退出即关闭连接池 |
| 请求层 | `_throttle` / `fetch_html` / `fetch_list_pages` | 节流、UA 伪装、超时重试、编码兜底、分页 |
| 解析层 | `parse_books`（静态方法） | 列表页四字段抽取 |
| 持久化层 | `save_csv` | 汇总写入 `utf-8-sig` 的 CSV |
| 编排层 | `run` | 串起抓取、打印预览、落盘、统计 |

模块级常量全部变成构造参数，命令行补上 `--pages` 与 `--out`：

```python
with NewsCrawler(max_pages=3, request_interval=2.0, output_csv="result.csv") as crawler:
    crawler.run()
```

### 4. 怎么证明"功能真的没变"

不靠感觉，靠产物比对。重构前先把 `result.csv` 备份一份，重构后用同样参数 `--pages 3` 重跑，再用 `csv` 模块逐行读两份文件比较：

```
共抓取图书：60 条
已写入 result.csv（utf-8-sig），共 60 行

before rows: 61 | after rows: 61
identical: True
```

61 行（表头 + 60 条数据）、每行每个字段全等。更直接的旁证是：提交后 `git status` 里 `result.csv` 根本没有出现改动，说明重构版生成的文件连字节都没变。

异常路径也重跑了一遍：无效域名仍然重试 3 次、退避 2 秒→4 秒、最后返回 `None`，日志文案与重构前一致，重试语义没被这次重构碰坏。

### 5. 重构中冒出来的新问题

`run()` 的 `finally` 里关闭了会话，`with` 退出时 `__exit__` 又关一次——重复关闭。`close()` 做了幂等处理（关闭后把 `_session` 置 `None`），所以不报错，但这是设计冗余。这类问题单元测试根本测不出来，只有把整条调用路径读一遍才会看见。

---

## 四、运行结果

```
[INFO] 请求成功 [1/3] http://books.toscrape.com/catalogue/page-1.html (status=200, len=51400)
...
===== 第 1 页（共 20 条），展示前 3 条 =====
  [1] 书名: A Light in the Attic
      价格: £51.77
      星级: Three
      链接: http://books.toscrape.com/catalogue/a-light-in-the-attic_1000/index.html
...
共抓取图书：60 条
已写入 result.csv（utf-8-sig），共 60 行
```

CSV 内容：

```csv
title,price,rating,link
A Light in the Attic,£51.77,Three,http://books.toscrape.com/catalogue/a-light-in-the-attic_1000/index.html
Tipping the Velvet,£53.74,One,http://books.toscrape.com/catalogue/tipping-the-velvet_999/index.html
```

---

## 五、异常场景实测

不只跑通顺流程，把异常也真跑了一遍：

| 场景 | 构造方式 | 实测现象 | 结论 |
|---|---|---|---|
| 字段缺失 | 手写缺书名/价格的卡片 HTML | 缺失字段全部返回 `""`，未抛异常 | 通过 |
| 无效域名 | `http://no-such-host-abc123.invalid/page.html` | 恰好重试 3 次，退避 2 秒、4 秒，最后返回 `None` | 通过 |
| 页面 404 | `page-9999.html` | 只记一条 ERROR 直接放弃，未进入重试 | 通过 |
| 编码 | 读 CSV 首字节 | `EF BB BF`，Excel 打开无乱码 | 通过 |
| 全部失败 | 未构造 | 会写出仅含表头的空 CSV，不报错 | 风险，已知 |

第三条值得说一句：重试只该作用于网络层的瞬时故障，404 是确定性错误，重试只会加重站点负担。AI 在这一点上的分支设计是对的。

---

## 六、复盘

### 1. 人工审查抓到的问题

1. 异常类未导入（阶段一直接跑挂）
2. UA 随机池对同一会话的指纹并不稳定，但对练习站点无实际副作用，保留
3. 价格带 `£`、星级是英文单词，本阶段按原始数据留存，未清洗
4. 解析失败返回空列表、不抛异常 —— 会造成"抓取 0 条也像正常结束"的**静默失败**，目前靠"每页条数 + 总条数"两处输出人工发现
5. 步骤 4 补做时才发现：阶段一的导入坑一直没被彻底消除，重构时才算真正修干净

### 2. 风险识别

- **DOM 脆弱性**：四个选择器全部硬编码，站点改版即失效且不报错
- **静默失败**：`all_books` 为空时仍写出空表 CSV
- **编码推断**：`apparent_encoding` 基于内容猜测，多语言混排仍有误判可能
- **合规边界**："2 秒间隔 + 受控页数"只是最低限度的礼貌策略，换成真实站点前需先看 `robots.txt` 与用户协议

### 3. 静态方案与动态方案（进阶部分实测数据）

指导书的进阶挑战是"抓一个动态渲染站点并比较两种方案"。目标换成 `quotes.toscrape.com/js/`，先用纯 `requests + BeautifulSoup` 取证：状态码 200、HTML 5806 字符、`div.quote` 数量 **0**、页面含 `var data`——静态解析在这种情况下不是"解析错了"，而是根本拿不到渲染后的 DOM。

两条项目实际跑通的路线：

| 维度 | 方案 B 内嵌数据反查 | 方案 A 无头浏览器渲染 |
|---|---|---|
| 做法 | 定位页面 `var data = [...]`，用 `JSONDecoder.raw_decode()` 精确解析 | 启动无头浏览器执行 JS，等卡片出现后解析 DOM |
| 2 页实测耗时 | **5.00 秒 / 20 条** | **19.96 秒 / 20 条** |
| 额外依赖 | 无（仅已有的 requests + bs4） | `websocket-client` + 本机 Edge/Chrome |
| 核心代码量 | 53 行 | 158 行（浏览器类 109 + 解析编排 49） |
| 结果一致性 | 两方案逐行字段（作者/正文/标签/页码）**完全一致** | 同左 |

耗时差约 4 倍、代码量差约 3 倍，拿到的数据却一模一样——这就是"先找数据源、最后才上浏览器"最直接的依据。

浏览器方案的落地过程比原理麻烦得多，几个真实的坑：

1. Selenium 4 的 `webdriver.Edge()` 抛 `NoSuchDriverException`，Selenium Manager 下载 `msedgedriver` 在本机就是下不动；
2. Playwright 的轮子（约 40MB + 浏览器运行时）从镜像装了 5 分钟没装完，放弃；
3. Edge 的 `--headless=old / --headless=new / --headless` 三种 `--dump-dom` 模式全部退出码 0、stdout 为空；
4. 最后改用浏览器自带的 DevTools 协议（`--remote-debugging-port` + `websocket-client`），零驱动、零额外二进制，1.42 秒完成启动并渲染出 10 张卡片；
5. 首次对比还因为第 1 页等 `div.quote` 25 秒超时失败（首访要等 DNS / TLS / 外部 JS），把超时放宽到 45 秒并对每页加一次重试才拿到干净结果。

判断顺序仍然不变：先看响应体有没有目标数据，有就静态解析；确认数据是 JS 生成后，先找内嵌数组或 XHR 接口；只有数据在客户端被二次加工（加密签名、多接口拼接）时，才升级到浏览器渲染。

### 4. 待改进

1. 加结构断言：解析后校验"每页条数 > 0"，否则记为失败页并告警
2. 空数据显式提示：写盘前判断 `all_books` 为空
3. 选择器外置成配置字典，站点改版只改配置
4. 数据清洗：价格转 `float`、星级映射 1–5 分
5. 日志落盘，保留每轮成功页数与失败页数

---

## 七、源码与文档

- 源码仓库：[python-spider-lab](https://github.com/qls-eng/python-spider-lab)
- 仓库内含：`fetcher.py`（`NewsCrawler` 类）、`dynamic_crawler.py`（进阶两方案，411 行）、`result.csv` / `quotes.csv`、`README.md`，以及 `docs/` 下的需求澄清清单、AI 协作过程记录（R1~R3 三轮迭代 + R7 步骤 4 重构 + R8 进阶挑战）、测试用例（TC-01~TC-17，含重构与两方案一致性回归）、复盘、进阶动态渲染方案对比共五份文档

### 一句话结论

AI 在单点实现上效率很高，但异常导入、静默失败、字段语义这类问题只能靠人工审查加真实运行发现。工程上的差距不在"能不能写出来"，而在"出错时能不能被看见、被定位"。
