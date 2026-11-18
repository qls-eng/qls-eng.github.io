---
layout: default
title: 课堂日常作业：函数、递归与组合数据（第 5~6 章）
date: 2026-11-01
categories: 课堂作业
tags: [Python, 函数, 递归, turtle, 科赫雪花, 蒙特卡罗, 集合, jieba, 词云]
---

# 课堂日常作业：函数、递归与组合数据（第 5~6 章）

> 本文汇总本阶段 Python 课堂日常作业。课堂作业统一存放在独立仓库 [**python-homework**](https://github.com/qls-eng/python-homework)，与各实验项目仓库分开管理；博客只做题目说明、思路讲解与源码索引。

## 作业清单

| 序号 | 题目 | 知识点 | 源码 |
|------|------|--------|------|
| 1 | `if __name__ == '__main__':` 的含义，why? | 模块入口、直接运行与导入的区别 | [笔记](https://github.com/qls-eng/python-homework/blob/main/notes/if_name_main.md) |
| 2 | P196 习题 5.1 字符田字格绘制 | 自定义函数、参数控制、循环嵌套 | [5.1_tianzige.py](https://github.com/qls-eng/python-homework/blob/main/ch5_function/5.1_tianzige.py) |
| 3 | 绘制科赫雪花，根据阶数生成（0~4 阶） | 递归函数、turtle 绘图 | [koch_snowflake.py](https://github.com/qls-eng/python-homework/blob/main/ch5_function/koch_snowflake.py) |
| 4 | 蒙特卡罗方法手搓图形 | 随机数、概率估算 π、turtle 散点 | [monte_carlo.py](https://github.com/qls-eng/python-homework/blob/main/ch5_function/monte_carlo.py) |
| 5 | 孤独的古诗词（P205 代码 6A） | 字符串处理、集合去重、`len(s) == len(set(s))` | [poem_solo.py](https://github.com/qls-eng/python-homework/blob/main/ch6_collection/poem_solo.py) |
| 6 | 统计"十五五"规划报告热词并生成词云 | jieba 分词、词频统计、wordcloud、中文字体 | [report_wordcloud.py](https://github.com/qls-eng/python-homework/blob/main/ch6_collection/report_wordcloud.py) |
| 7 | 自动轨迹绘制 | 指令数据解析、参数校验、turtle 转向与颜色 | [trace_draw.py](https://github.com/qls-eng/python-homework/blob/main/ch6_collection/trace_draw.py) |

---

## 一、`if __name__ == '__main__':` 的含义，why?

### 题目

解释 Python 中 `if __name__ == '__main__':` 的含义，以及为什么要这样写。

### 要点

每个 `.py` 文件被加载时都会生成内置变量 `__name__`：

- 文件**被直接运行**（`python xxx.py`）时，`__name__` 的值是 `'__main__'`；
- 文件**被其他文件 `import` 导入**时，`__name__` 的值是模块名（去掉 `.py` 的文件名）。

因此这行判断的作用是：**只有当前文件被直接运行时，才执行下面缩进的入口代码；被导入时不执行**。这样模块既能独立运行做自测，又能被其他程序干净地复用函数定义，而不会在导入时自动执行测试或启动代码。

完整讲解与示例见仓库笔记：[`notes/if_name_main.md`](https://github.com/qls-eng/python-homework/blob/main/notes/if_name_main.md)。

---

## 二、习题 5.1 字符田字格绘制（P196）

### 题目

编写程序，用字符方式打印田字格；**采用函数方式，以格子数 `n` 为参数**，能够根据参数绘制任意大小的 `n × n` 田字格。

### 思路

把田字格的绘制封装成函数 `draw_grid(n)`：

- 横线由 `+----` 重复 `n` 次后结尾一个 `+`；
- 竖线由 `|    ` 重复 `n` 次后结尾一个 `|`；
- 每个格子高度为 4 行，外层循环 `n` 次，先画一条横线，再画 4 行竖线，最后补一条底边。

核心代码：

```python
def draw_grid(n: int) -> None:
    """用字符打印 n×n 的田字格。"""
    if not isinstance(n, int) or n <= 0:
        print("宽度必须为大于 0 的正整数")
        return

    horizontal = "+----" * n + "+"
    vertical = "|    " * n + "|"

    for _ in range(n):
        print(horizontal)
        for _ in range(4):
            print(vertical)
    print(horizontal)


if __name__ == "__main__":
    n = int(input("请输入田字格格子数 n: "))
    draw_grid(n)
```

以 `n = 2` 为例，运行结果为 2×2 田字格：

```
+----+----+
|    |    |
|    |    |
|    |    |
|    |    |
+----+----+
|    |    |
|    |    |
|    |    |
|    |    |
+----+----+
```

完整源码：[`ch5_function/5.1_tianzige.py`](https://github.com/qls-eng/python-homework/blob/main/ch5_function/5.1_tianzige.py)。

---

## 三、科赫雪花（0~4 阶，递归）

### 题目

绘制科赫雪花，根据阶数生成，支持 **0~4 阶**。

### 思路

科赫曲线是典型的递归图形：

- **0 阶**：一条直线段；
- **高阶**：把一条线段三等分，把中间一段替换成等边三角形的两条边（向上凸起），再对每一小段递归处理；
- **雪花**：由三条科赫曲线首尾相接围成等边三角形。

递归函数中先把长度三等分，按"画一段 → 左转 60° → 画一段 → 右转 120° → 画一段 → 左转 60° → 画一段"的顺序完成凸起造型。

核心递归代码：

```python
def koch_curve(t, length: float, depth: int) -> None:
    if depth == 0:
        t.forward(length)
        return

    length /= 3
    koch_curve(t, length, depth - 1)
    t.left(60)
    koch_curve(t, length, depth - 1)
    t.right(120)
    koch_curve(t, length, depth - 1)
    t.left(60)
    koch_curve(t, length, depth - 1)


def koch_snowflake(t, depth: int, length: float = 320) -> None:
    for _ in range(3):
        koch_curve(t, length, depth)
        t.right(120)
```

运行后输入阶数（0~4）即可弹出 turtle 窗口绘制；阶数越高，雪花轮廓越细碎。

完整源码：[`ch5_function/koch_snowflake.py`](https://github.com/qls-eng/python-homework/blob/main/ch5_function/koch_snowflake.py)。

---

## 四、蒙特卡罗方法手搓图形

### 题目

使用蒙特卡罗方法手搓图形：在正方形内随机大量投点，落入内切圆内与圆外的点分别用不同颜色绘制，直观形成一个圆，并据此估算圆周率 π。

### 思路

在边长为 2 的正方形（坐标范围 `[-1, 1]`）内随机投点：

- 正方形面积为 4，内切单位圆面积为 π；
- 落入圆内的点 / 总点数 ≈ 圆面积 / 正方形面积 = π / 4；
- 因此 π ≈ 4 × 圆内点数 / 总点数。

圆内点画红色，圆外点画蓝色，散点聚集起来就"手搓"出一个圆。

核心代码：

```python
def monte_carlo_pi(t, n_points: int, scale: int = 200) -> float:
    inside = 0
    for _ in range(n_points):
        x = random.uniform(-1, 1)
        y = random.uniform(-1, 1)
        if x * x + y * y <= 1:
            inside += 1
            t.color("red")
        else:
            t.color("blue")
        t.penup()
        t.goto(x * scale, y * scale)
        t.pendown()
        t.dot(3)
    return 4 * inside / n_points
```

投点数量越大，估算值越接近 π（例如 5 万点时约为 3.13 左右）。完整源码：[`ch5_function/monte_carlo.py`](https://github.com/qls-eng/python-homework/blob/main/ch5_function/monte_carlo.py)。

---

## 五、孤独的古诗词（P205 代码 6A）

### 题目

一首古诗词去掉标点后，若**每个汉字都只出现一次**，称它为"孤独的古诗"，编写程序进行判定。

### 思路

教材代码 6A 的思路很短：把诗词去标点后转成集合，集合会自动去重，比较"字符个数"和"集合元素个数"即可。

```python
def is_lonely_poem(text: str) -> bool:
    poem = strip_punct(text)              # 只保留汉字
    return len(poem) == len(set(poem))    # 长度相等说明没有重复字
```

在此基础上扩展了两件事：一是把判定封装成函数，支持批量处理；二是对"不孤独"的诗**列出重复字及出现次数**，让判定结果可解释，而不只是一个布尔值。

### 实测结果

| 诗 | 作者 | 去标点字数 | 不同字数 | 重复字 | 判定 |
|---|---|---|---|---|---|
| 独坐敬亭山 | 李白 | 20 | 20 | 无 | True |
| 春晓 | 孟浩然 | 20 | 19 | 处（2 次） | False |
| 鹿柴 | 王维 | 20 | 19 | 人（2 次） | False |

三首都是五言绝句（20 字），差别就在那一个字：`处` 和 `人` 各重复一次，判定立刻翻转。这也回应了思考练习 6.3：换一首诗，输出的两个数字随之变化，只要出现任意一个重复汉字，两个数字就不相等。

完整源码：[`ch6_collection/poem_solo.py`](https://github.com/qls-eng/python-homework/blob/main/ch6_collection/poem_solo.py)。

---

## 六、统计"十五五"规划报告热词并生成词云

### 题目

统计规划报告中出现频率较高的词，按词频排序输出，并用 `wordcloud` 生成词云图片。

### 语料与流程

语料使用**公开新闻稿节选**，直接内置在代码的 `TEXT` 常量里（516 字），运行时不依赖外部 txt，开箱即可跑出词云。替换 `TEXT` 就能统计任意文本。

```
jieba 分词 → 过滤停用词与单字 → Counter 词频统计 → Top20 打印
          → report_hotwords.csv + report_wordcloud.png
```

实测数据：

- 分词得到 263 个词单元，过滤停用词与单字后保留 107 个不同词；
- 最高频的是「高水平」「安全」「保障」「强国」（各 3 次），其后「十五五」「经济社会」「高质量」「科技」「创新」等（各 2 次）；
- 词频分布很平，没有单一压倒性主题，符合规划文本覆盖面广的特点。

![十五五报告热词词云](/assets/images/python-homework/report_wordcloud.png)

### 两个关键处理

**（1）领域词要手动补进词典**

默认词典会把「十五五」切成「十五」+「五」，词云里就会冒出无意义的「十五」。解决办法是调用 `jieba.add_word()` 补充专有说法：

```python
CUSTOM_WORDS = ["十五五", "新质生产力", "现代化产业体系", "科技自立自强",
                "全国统一大市场", "碳达峰", "碳中和", "低空经济", ...]
for word in CUSTOM_WORDS:
    jieba.add_word(word)
```

**（2）中文字体有两个通道**

`WordCloud(font_path=...)` 只作用于词云内部文字；如果还要用 matplotlib 画标题或坐标轴，需要另设 `plt.rcParams['font.sans-serif'] = ['Microsoft YaHei']`。只设一个，汉字就会变方框。这和[实验四](https://qls-eng.github.io/qls-study-blog.github.io/2026/11/18/local-doc-review-agent.html)遇到的是同一个坑。

完整源码：[`ch6_collection/report_wordcloud.py`](https://github.com/qls-eng/python-homework/blob/main/ch6_collection/report_wordcloud.py)。

---

## 七、自动轨迹绘制

### 题目

给定一组指令数据，用 turtle 按数据自动绘制彩色轨迹。指令以逗号分隔，每行 6 个数值：

```
行进距离, 转向标志(0 左转 / 1 右转), 转向角度, R, G, B
```

### 思路

把"画什么"从代码里抽出来，交给数据决定：读一行 → 换颜色 → 转向 → 前进。程序只负责解析与执行，换一组数据就是另一幅图。

```python
for record in records:
    pen.pencolor(record["r"], record["g"], record["b"])   # 先换颜色
    if record["direction"] == 0:
        pen.left(record["angle"])
    else:
        pen.right(record["angle"])
    pen.fd(record["distance"])
```

内置的六条示例指令中，前四条每次左转 144°，正好画出五角星的五个角；后两条转 108° 与 72°，继续接成折线，颜色由红、绿、蓝、黄、青、品红依次切换。

### 数据校验不能省

指令来自外部文件时，格式错误会让 turtle 画出一堆莫名其妙的线。所以在解析阶段就把三类问题拦住：

- 字段数不等于 6；
- 转向标志不是 0 或 1；
- 颜色分量不在 0~1 之间。

校验失败直接打印行号并退出，不会带着脏数据去绘图。另外提供 `--no-gui` 参数，只解析并打印每条指令，方便在无图形环境（比如服务器）自测。

完整源码：[`ch6_collection/trace_draw.py`](https://github.com/qls-eng/python-homework/blob/main/ch6_collection/trace_draw.py)。

---

## 运行方式

仓库：[**python-homework**](https://github.com/qls-eng/python-homework)。第 2~5、7 题仅依赖 Python 标准库（`turtle`、`random`、`re`、`argparse`）；第 6 题需要 `jieba`、`wordcloud`、`matplotlib`。

```bash
pip install -r requirements.txt          # 仅第 6 题需要

python ch5_function/5.1_tianzige.py      # 字符田字格（命令行输出）
python ch5_function/koch_snowflake.py   # 科赫雪花（turtle 窗口，输入阶数 0~4）
python ch5_function/monte_carlo.py      # 蒙特卡罗估算 π（turtle 窗口，输入投点数）

python ch6_collection/poem_solo.py        # 孤独的古诗词判定
python ch6_collection/report_wordcloud.py # 热词统计 + 词云（生成 CSV 与 PNG）
python ch6_collection/trace_draw.py       # 自动轨迹绘制（turtle 窗口）
python ch6_collection/trace_draw.py --no-gui  # 只解析指令、不弹窗
```

> 图形题目依赖 turtle 窗口，需在本地带图形界面的环境运行；词云需要中文字体，程序会按候选列表自动查找 `msyh.ttc` / `simhei.ttf` / `simsun.ttc`，找不到时只输出词频表并提示。
