# AI 统计顾问 (AI Statistical Consultant)

一个自动化统计检验推荐与解读工具：上传两组实验数据，自动判断该用哪种统计检验方法，
执行检验，并生成给非统计背景业务方看的人话报告。

## 项目动机

这个项目源于我在统计咨询工作中完成的一份作业——根据药企客户提供的山羊驱虫疫苗实验数据，
判断该用什么统计检验、解释统计概念、给出业务建议。我把当时的人工分析流程，
转化成了一套可复用的自动化工具。

## 核心设计理念：分清"规则逻辑"和"AI"的边界

这个项目刻意把流程拆成两个性质完全不同的部分：

| 模块 | 性质 | 为什么这么设计 |
|---|---|---|
| `decision_engine.py` 统计假设判断 | **确定性规则逻辑，不是AI** | 该用哪种检验方法直接影响业务决策的正确性，这种判断需要可解释、可重复、不能有"幻觉"风险，并且得同时符合统计学以及业务落地的逻辑，所以用统计学规则而非大模型来做 |
| `stat_tests.py` 检验计算 | **纯数学计算** | scipy统计库，结果可验证、可复现 |
| `report_generator.py` 报告生成 | **大语言模型（LLM）** | 把统计数字转译成业务语言是语言生成任务，恰好是LLM的强项，且就算输出有偏差，也不会影响统计结论本身的正确性 |

我没有让大模型来决定该用什么统计检验，因为这种判断如果出错可能误导业务决策；
我把大模型留给了它真正擅长、出错代价最低的环节——结果解释和报告撰写。

## 决策逻辑

```
两组独立样本数据
      │
      ▼
 正态性检验 (Shapiro-Wilk)
      │
   ┌──┴──┐
不满足正态  满足正态
   │        │
   ▼        ▼
Mann-Whitney  方差齐性检验 (Levene's test)
U 检验           │
（备选：       ┌──┴──┐
Bootstrap）   方差齐   方差不齐
                │        │
                ▼        ▼
          独立样本t检验  Welch's t检验
```

样本量过小（默认阈值：单组少于5个观测值）时，无论正态性如何，
都会保守地走向 Mann-Whitney U 检验路径，并提示 Bootstrap 重抽样法作为备选——
这种小样本下更稳健的方法不依赖任何分布假设。

## 快速开始

```bash
pip install -r requirements.txt

# 可选：配置自己的 Anthropic API key 以启用AI报告生成
export ANTHROPIC_API_KEY="你的key"

streamlit run app.py
```

没有配置 API key 也能运行——统计判断和检验计算照常工作，报告部分会
降级为基础模板，方便你先验证核心逻辑。

## 验证

项目自带的demo数据（`sample_data/goat_worms.csv`）来自我的统计咨询课作业，
原作业用SPSS计算的Mann-Whitney检验得到 p = 0.283，本项目用Python重新计算
得到 p = 0.2828，结果一致，验证了核心逻辑的正确性。

## 项目结构

```
stat_consultant_ai/
├── decision_engine.py     # 统计假设判断 + 检验方法推荐（规则引擎）
├── stat_tests.py           # 四种检验方法的实际计算
├── report_generator.py     # 调用LLM生成业务报告
├── app.py                  # Streamlit网页界面
├── sample_data/
│   └── goat_worms.csv      # demo数据
└── requirements.txt
```

## 后续可扩展方向

- 支持两组以上的多组比较（ANOVA / Kruskal-Wallis）
- 支持配对样本检验
- 自动生成PDF格式报告
- 支持效应量（effect size）计算，不只是看p值
