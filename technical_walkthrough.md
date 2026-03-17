# Holdem Exploration 技术 Walk Through

## 1. 文档目的

这份文档用于对 `project-holdem-exploration` 仓库做一次技术层面的整体说明，帮助团队快速理解：

- 这个系统要解决什么问题
- 当前前后端、训练、求解、数据处理分别放在哪一层
- 各层之间如何衔接
- 当前系统真实的运行方式是什么
- 哪些能力已经接入主链路，哪些仍然属于预留或演进方向


## 2. 系统整体定位

这个系统本质上是一个面向 NLHE 策略研究和服务化展示的平台。它围绕 3 层策略展开：

1. `GTO baseline`
2. `Population tendency`
3. `Proprietary exploit strategy`

核心思想是：

- 先有一层近似 GTO 的基线策略
- 再从真实手牌数据中统计人群倾向
- 然后根据人群偏差生成 exploit 型 proprietary strategy
- 最后通过 API、CLI、Web UI 把这些结果暴露给用户

当前系统并不是一个“在线实时求解器”，而更像一个“离线训练 + 在线查询”的策略平台。


## 3. Repo 结构总览

仓库可以粗略拆成 6 个技术层：

```text
.
├── api/         FastAPI 服务层
├── cli/         命令行入口
├── core/        Rust 底层能力：parser / engine 抽象
├── solver/      GTO 参考数据与 PioSOLVER 集成
├── training/    population 统计、偏差分析、策略生成、验证、版本发布
├── web/         Next.js 前端
└── output/      文档与产出物
```

如果从运行链路来看，当前主路径是：

`训练产物 JSON -> EngineService -> FastAPI -> Next.js / CLI`


## 4. 当前真实运行时架构

### 4.1 服务启动流程

后端入口是 [api/main.py](../api/main.py)。

FastAPI 启动时会：

1. 创建 `EngineService`
2. 在启动阶段把它挂到 `app.state.engine`
3. 让各个 router 在请求中共享这一个 engine 实例

也就是说，API 层本身非常薄，业务逻辑主要集中在 `EngineService`。

### 4.2 当前数据来源

当前线上主路径的数据来源并不是数据库，而是磁盘上的 JSON 产物：

- `solver/gto_baseline.json`
- `training/population_data.json`
- `training/proprietary_strategy.json`
- `training/versions/v*/strategy.json`
- `training/versions/v*/metadata.json`

`EngineService` 会在启动时把这些文件全部加载进内存，然后后续请求都从内存中查询。

### 4.3 为什么说它是 artifact-first

虽然项目依赖里有：

- `sqlalchemy`
- `psycopg2-binary`
- `redis`

而且 `docker-compose.yml` 里也定义了 Postgres 和 Redis 服务，但从当前代码实现看：

- 没有实际的 ORM model
- 没有 repository 层
- API 请求没有访问数据库
- 也没有 Redis 作为 cache 或队列被正式接入

所以目前它更像一个“以训练产物为中心”的系统，而不是“以数据库为中心”的系统。


## 5. 后端 API 层

### 5.1 技术栈

后端使用：

- Python
- FastAPI
- Pydantic

依赖定义在 [pyproject.toml](../pyproject.toml)。

### 5.2 路由划分

API 分成三组：

#### `strategy`

- `POST /strategy/lookup`
- `GET /strategy/version`
- `GET /strategy/versions`

职责：

- 查询单点三层策略
- 获取当前 active version
- 获取全部版本列表

#### `analysis`

- `POST /analysis/hand`
- `POST /analysis/spot`

职责：

- 对单个 hand 或单个 spot 做简化分析
- 本质上仍然建立在 `lookup_strategy()` 结果之上

#### `reports`

- `GET /reports/population`
- `GET /reports/deviations`
- `GET /reports/strategy-diff/{v1}/{v2}`

职责：

- 输出 population summary
- 输出 deviation ranking
- 比较两个版本的策略差异

### 5.3 Schema 设计

所有请求和响应模型在 `api/schemas.py` 中集中定义，核心模型包括：

- `GameState`
- `ActionFrequencies`
- `StrategyResponse`
- `PopulationSummary`
- `VersionInfo`
- `DeviationReport`

这使得：

- API 层契约稳定
- Web 前端可以镜像这些类型
- CLI 和测试也能复用同一套数据模型


## 6. EngineService：当前系统核心

`EngineService` 是整个后端运行时的中枢。

它负责：

- 加载 GTO / population / proprietary 数据
- 构造和解析 `spot_key`
- 根据 `GameState` 查三层策略
- 生成 reasoning
- 计算 deviations
- 获取版本信息
- 对比不同版本之间的策略变化

### 6.1 Spot Key 设计

当前系统通过一个 canonical key 来统一索引策略：

```text
nlhe_cash:{positions}:{stack_depth}:{board_texture}:{action_sequence}
```

例如：

```text
nlhe_cash:BTN_vs_BB:deep:flop:open_3bet
```

这个 key 是整个系统跨层联通的关键：

- 训练输出按它组织
- API 查询按它命中
- 前端结果页按它展示
- deviation / version diff 也按它汇总

### 6.2 查询逻辑

对一次 `lookup_strategy()` 来说，流程大致是：

1. 用 `GameState` 构建 `spot_key`
2. 在 GTO 层取对应 spot
3. 在 population 层取对应 spot
4. 在 active proprietary version 中取对应 spot
5. 如果没命中，走 default / demo fallback
6. 生成一段 reasoning 文本
7. 返回统一的 `StrategyResponse`

### 6.3 当前 reasoning 的性质

当前 reasoning 是规则生成的，不是 LLM，也不是高级解释模型。

它主要基于：

- population 相对 GTO 的最大偏差
- proprietary 相对 GTO 的主要调整方向
- `ev_gain`

然后拼出一段可读说明。

这意味着解释层当前是轻量的、工程化的、可控的，但不算特别智能。


## 7. 数据处理与训练链路

训练链路位于 `training/` 目录，是这个 repo 最核心的“策略生成”部分。

### 7.1 Pipeline 总览

完整训练流程定义在 `training/run_training.py`，分成 6 步：

1. 加载或生成 GTO baseline
2. 计算 population tendencies
3. 分析 deviations
4. 生成 proprietary adjustments
5. Monte Carlo validation
6. deploy 成新版本

这是典型的离线批处理 pipeline，而不是在线学习系统。

### 7.2 TrainingConfig

`training/config.py` 中的 `TrainingConfig` 统一管理两类东西：

#### 统计 / 算法参数

- `MIN_SAMPLE_SIZE`
- `CONFIDENCE_THRESHOLD`
- `ALPHA`
- `HOLDOUT_RATIO`
- `MIN_FREQ_BOUND`
- `MAX_FREQ_BOUND`
- `MONTE_CARLO_ITERATIONS`

#### 文件路径

- GTO baseline
- population data
- metadata
- version output
- parser 中间结果

这使训练阶段是高度“文件系统驱动”的。


## 8. Population Tendency 计算

`training/population.py` 的职责，是把真实样本转成结构化的 population tendency 数据。

### 8.1 输入来源

它优先读取完整 parser 输出目录中的：

- `hands.csv`
- `players.csv`
- `actions.csv`

如果这套 CSV 不存在，则退回到较弱的聚合分析输入。

### 8.2 统计过程

它会识别典型 spot，例如：

- preflop RFI
- vs RFI
- BB defend
- flop c-bet
- vs c-bet
- turn barrel
- river bet / defend

最终把大量 hand-level 数据压缩为：

```json
spot_key -> {
  action_freqs,
  sample_size,
  confidence
}
```

### 8.3 这层的角色

这层本质上是：

- 从原始行为数据到策略分布数据的统计抽象层

它不是 solver，也不是模型训练器，而是训练前的数据聚合器。


## 9. Deviation Analysis

`training/deviation.py` 负责比较：

- GTO baseline
- population tendency

来识别 exploitable leaks。

### 9.1 做了什么

对每个共有 spot：

1. 计算 action-level delta
2. 做卡方检验
3. 判定是否显著
4. 分类 deviation 类型
5. 估计 EV opportunity

### 9.2 输出

输出的是一组 `DeviationReport`，其中包含：

- `spot_key`
- `deviation_type`
- `delta`
- `p_value`
- `significant`
- `ev_opportunity`
- `sample_size`
- `description`

### 9.3 当前算法特性

这部分已经具备明确的统计判断逻辑，但 EV 估算仍是启发式的，不是完整博弈树求值。


## 10. Proprietary Strategy 生成

`training/adjustment.py` 是当前 proprietary strategy 的核心算法。

### 10.1 核心公式

```text
prop_freq = gto_freq + alpha * delta * confidence_weight
```

它的直觉是：

- 如果 population 在某个 action 上明显偏离 GTO
- 且这个偏离样本足够大、统计上足够显著
- 那我们就沿 exploit 方向对 GTO 进行偏移

### 10.2 方向性规则

当前根据 deviation type 做定向调整，例如：

- `over_folding`：增加进攻频率
- `over_calling`：减少 bluff，偏向 value-heavy
- `under_raising`：提高主动下注 / raise 频率

### 10.3 后处理

生成后的频率会经过：

- clamp 到上下界
- normalize 成合法概率分布
- EV gain 估算
- reasoning 生成

### 10.4 这意味着什么

当前 proprietary strategy 不是神经网络输出，也不是 solver 在线算出来的 equilibrium，而是：

一个“统计显著性驱动的、规则化 exploit 调整器”。


## 11. Validation 层

`training/validation.py` 负责验证 proprietary strategy 是否真的优于 GTO baseline。

### 11.1 方法

当前使用的是简化 Monte Carlo：

- 对 spot 做 train / holdout split
- 按 hero / villain frequency 随机采样动作
- 用一个简化 payoff matrix 估 EV
- 对比 proprietary vs GTO 的 EV 差值

### 11.2 输出指标

包括：

- `aggregate_ev_gain`
- `avg_ev_gain_per_spot`
- `holdout_concordance`
- `degenerate_spots`

### 11.3 当前局限

它验证的是“频率级策略对抗”的收益差，而不是完整 game tree 上的严格 EV。

所以这一步更像工程验证，而不是理论上非常严谨的 solver 级验证。


## 12. 版本管理与部署

`training/deploy.py` 把训练结果转成可服务的版本化产物。

### 12.1 产物结构

每次 deploy 会生成：

- `training/versions/vN/strategy.json`
- `training/versions/vN/metadata.json`
- `training/versions/vN/diff_vs_previous.md`

并且更新：

- `training/versions/active.json`

同时还会写一份：

- `training/proprietary_strategy.json`

### 12.2 这层的意义

它让策略不只是一次性训练结果，而是一个可追踪、可比较、可切换的版本序列。

这也是 Web 里 `Versions` 页面和 `/reports/strategy-diff` 的基础。


## 13. Solver 层

`solver/` 目录提供的是 GTO 基线构建能力，以及与 PioSOLVER 的潜在集成。

### 13.1 组成

- `gto_reference.py`
- `gto_baseline.json`
- `upi_client.py`
- `batch_solver.py`
- `spot_generator.py`
- `strategy_exporter.py`

### 13.2 当前角色

这个目录有两种模式：

#### 模式 A：真实 PioSOLVER

如果本机存在 PioSOLVER binary：

- `PioSolverClient` 可以通过 UPI 协议拉起 solver
- 设置 pot / board / stack / bet sizes
- 建树并求解
- 导出 root strategy

#### 模式 B：Reference fallback

如果没有 PioSOLVER：

- 用 `gto_reference.py` 中预置的参考策略数据兜底

### 13.3 当前和主链路的关系

需要强调的是：

- solver 层具备比较强的潜力
- 但当前 API 主链路并不会实时调用 solver
- 线上服务仍然是读取已经生成好的 baseline JSON

所以 solver 更像“上游生产层”，不是“当前在线推理层”。


## 14. Rust Core 层

`core/` 里有两个重要方向：

### 14.1 `core/parser`

这是一个 Rust 写的高性能手牌解析器，支持：

- 遍历目录中的手牌文本
- 拆分单手
- 解析 header / seats / actions / board / summary
- 并行处理大量文件
- 输出 3 张 CSV：
  - `hands.csv`
  - `players.csv`
  - `actions.csv`
- 自动生成 PostgreSQL 导入 schema

这层非常适合大规模 hand history ingestion。

### 14.2 `core/engine`

这是更底层的策略和 spot 抽象层，提供：

- `ActionFreqs`
- `ThreeLayerStrategy`
- `StrategyStore`
- nearest-neighbor spot lookup
- board / stack / action categorization

### 14.3 当前接入状态

目前 Rust core 是“存在且有能力”的，但并不是当前 FastAPI 主路径上的直接依赖。

也就是说：

- Rust parser 已经很适合作为离线 ingestion 工具
- Rust engine 还没有完全替代 Python 的 `EngineService`

当前主服务仍然是 Python 读取 JSON 再返回。


## 15. 前端层

前端使用：

- Next.js 16
- React
- Tailwind CSS 4

### 15.1 主要职责

前端不是做复杂业务计算，而是做：

- 策略查询输入
- 版本、population、deviation 展示
- 结果可视化
- demo fallback 展示

### 15.2 与后端的关系

`web/src/lib/api.ts` 里定义了前端对后端的契约镜像，并封装了 fetch。

一个很实用的设计是：

- 如果后端请求失败
- 前端会自动进入 demo mode

所以这个前端即使后端不在线，也仍然可用来演示交互和信息架构。


## 16. CLI 层

`cli/main.py` 提供了一个本地命令行界面。

支持：

- 交互式分析
- spot lookup
- 查看 active version
- 查看 population stats
- 触发训练

CLI 并不是另一套系统，而是复用了：

- `api.schemas`
- `EngineService`

因此它更像一个开发者和研究人员的本地工作入口。


## 17. 测试与质量保障

当前测试主要集中在 `tests/test_integration.py`，覆盖了：

- EngineService 初始化
- strategy lookup
- spot key 格式
- active version 逻辑
- deviation 排序
- FastAPI 主要接口
- 基础异常场景

这说明当前测试更偏“集成 smoke test”，而不是深度算法验证测试。

训练层、solver 层、Rust parser 层还可以继续补更强的测试覆盖。


## 18. 当前系统的真实成熟度判断

如果用工程视角来评价，这个仓库目前是一个：

`已经具备完整骨架的策略研究平台原型`

它已经有：

- 数据 ingestion 层
- GTO 基线层
- population 统计层
- exploit 调整层
- validation 层
- versioning 层
- API 层
- Web 层

但它也明显还处于“工程演进中”：

- DB / Redis 基础设施已准备，但没有正式进入主链路
- Rust engine 没有完全接管服务层
- solver 在线求解没有接入 API 主路径
- EV 模型和 reasoning 仍偏启发式


## 19. 最适合对团队的总结说法

如果需要一句话总结给同事，可以这样说：

> 这个 repo 当前实现的是一个离线训练、文件产物驱动、在线查询展示的三层扑克策略平台。训练侧从 hand history 统计 population tendency，再基于 GTO 与 population 的偏差生成 proprietary exploit strategy，最终通过版本化产物、FastAPI 和 Next.js 提供查询和分析能力。


## 20. 后续演进方向

从技术角度看，这个系统接下来最自然的演进方向有 4 条：

### 20.1 数据层正式化

- 用 Postgres 承接 parser 输出和聚合结果
- 用 Redis 做缓存、异步任务状态或热点查询

### 20.2 服务层升级

- 让 API 层从“读 JSON”升级为“读 versioned store / DB / cache”
- 逐步把 `EngineService` 和 Rust core 做更深整合

### 20.3 算法层升级

- 提升 deviation -> proprietary 的策略生成精度
- 提升 EV 估算严谨度
- 将 solver 结果更系统地纳入 baseline 构建

### 20.4 平台化

- 增加训练任务编排
- 增加版本审批 / 回滚 / 比较流程
- 增加更完整的 observability 和数据 lineage


## 21. 结论

这个 repo 最重要的价值，不是某一个单点模块，而是它已经把“数据、策略、服务、展示”几层打通了。

当前主链路虽然仍然偏轻量、偏文件驱动，但系统骨架已经很清楚：

- 上游可以继续增强 parser / solver / training
- 中间层可以继续增强 versioned serving
- 下游可以继续增强 Web / CLI / 分析体验

所以从团队协作角度，这已经不是一个孤立 demo，而是一个可以继续工程化扩展的基础平台。
