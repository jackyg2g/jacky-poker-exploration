# Holdem Exploration 架构图

## 1. 一句话概括

这是一个以 `spot_key` 为统一索引、以离线训练产物为在线服务核心的数据与策略平台。

## 2. 全局架构图

```text
                         ┌──────────────────────────────┐
                         │        Hand Histories         │
                         │   GGPoker / raw .txt files    │
                         └──────────────┬───────────────┘
                                        │
                                        ▼
                    ┌──────────────────────────────────────────┐
                    │          Rust Parser: core/parser        │
                    │  - parse hand text                       │
                    │  - batch process                         │
                    │  - output CSVs                           │
                    └──────────────┬───────────────┬───────────┘
                                   │               │
                                   │               └─────────────────────┐
                                   ▼                                     │
                    ┌──────────────────────────────┐                     │
                    │ temp/full_parse_output/      │                     │
                    │ - hands.csv                  │                     │
                    │ - players.csv                │                     │
                    │ - actions.csv                │                     │
                    └──────────────┬───────────────┘                     │
                                   │                                     │
                                   ▼                                     │
                    ┌──────────────────────────────────────────┐          │
                    │          training/population.py          │          │
                    │  - aggregate action frequencies          │          │
                    │  - build population tendencies           │          │
                    └──────────────┬───────────────────────────┘          │
                                   │                                      │
                                   ▼                                      │
                    ┌──────────────────────────────┐                      │
                    │ training/population_data.json │                      │
                    │ training/population_metadata  │                      │
                    └──────────────┬───────────────┘                      │
                                   │                                      │
                                   ▼                                      │
          ┌────────────────────────────────────────────────────────────┐   │
          │                    training pipeline                       │   │
          │                                                            │   │
          │  deviation.py  -> adjustment.py -> validation.py           │   │
          │                                                            │   │
          │  compare GTO vs population                                 │   │
          │  generate proprietary exploit strategy                     │   │
          │  validate via Monte Carlo                                  │   │
          └──────────────┬─────────────────────────────────────────────┘   │
                         │                                                 │
                         ▼                                                 │
        ┌──────────────────────────────────────────────────────┐           │
        │              training/deploy.py                      │           │
        │  - create versioned strategy artifacts              │           │
        │  - write metadata                                   │           │
        │  - update active version                            │           │
        └──────────────┬───────────────────────────────────────┘           │
                       │                                                   │
                       ▼                                                   │
  ┌──────────────────────────────────────────────────────────────────────┐  │
  │                         training/versions/                          │  │
  │  active.json                                                       │  │
  │  v1/strategy.json                                                  │  │
  │  v1/metadata.json                                                  │  │
  │  v2/...                                                            │  │
  │  v3/...                                                            │  │
  └───────────────────────┬─────────────────────────────────────────────┘  │
                          │                                                │
                          │                                                │
                          ▼                                                │
        ┌──────────────────────────────────────────────────────┐           │
        │            FastAPI + EngineService                   │           │
        │  api/main.py                                         │           │
        │  api/services/engine_service.py                      │           │
        │  - load GTO / population / active proprietary       │◄──────────┘
        │  - build spot_key                                   │
        │  - lookup 3-layer strategy                          │
        │  - expose reports / versions / diffs                │
        └──────────────┬──────────────────────────────┬────────┘
                       │                              │
                       │                              │
                       ▼                              ▼
         ┌──────────────────────────┐     ┌──────────────────────────┐
         │      Next.js Web UI      │     │          CLI             │
         │        web/src/          │     │        cli/main.py       │
         │  - overview              │     │  - lookup                │
         │  - analyze               │     │  - analyze               │
         │  - population            │     │  - stats                 │
         │  - versions              │     │  - version               │
         └──────────────────────────┘     └──────────────────────────┘
```

## 3. GTO 基线来源图

```text
                    ┌─────────────────────────────┐
                    │   solver/gto_reference.py   │
                    │   prebuilt reference spots  │
                    └──────────────┬──────────────┘
                                   │
                                   │ fallback
                                   ▼
                    ┌─────────────────────────────┐
                    │   solver/batch_solver.py    │
                    │   solve or fallback         │
                    └──────────────┬──────────────┘
                                   │
                    if PioSOLVER   │   if unavailable
                         exists    │
                                   ▼
                    ┌─────────────────────────────┐
                    │    solver/upi_client.py     │
                    │    PioSOLVER UPI bridge     │
                    └──────────────┬──────────────┘
                                   │
                                   ▼
                    ┌─────────────────────────────┐
                    │  solver/gto_baseline.json   │
                    │  baseline artifact on disk  │
                    └─────────────────────────────┘
```

说明：

- 当前在线 API 主链路不会实时调 PioSOLVER。
- API 实际读取的是 `solver/gto_baseline.json` 这类落盘产物。

## 4. 训练链路图

```text
   solver/gto_baseline.json
              │
              │
              ▼
   training/population.py
   from parsed CSVs or aggregate data
              │
              ▼
   training/population_data.json
              │
              ▼
   training/deviation.py
   compare GTO vs population
              │
              ▼
   deviation reports
              │
              ▼
   training/adjustment.py
   generate proprietary frequencies
              │
              ▼
   proprietary strategy candidate
              │
              ▼
   training/validation.py
   Monte Carlo validation
              │
              ▼
   training/deploy.py
              │
              ▼
   training/versions/vN/*
```

## 5. 在线查询主路径

```text
User input
   │
   ▼
Web UI / CLI
   │
   ▼
FastAPI endpoint
   │
   ▼
EngineService.lookup_strategy()
   │
   ├── build_spot_key(GameState)
   │
   ├── load GTO spot
   │
   ├── load population spot
   │
   ├── load active proprietary spot
   │
   ├── generate reasoning
   │
   ▼
StrategyResponse
   │
   ▼
UI renders:
- GTO
- Population
- Proprietary
- EV gain
- reasoning
```

## 6. Spot Key 作为系统主索引

```text
GameState
  ├── position
  ├── villain_position
  ├── stack_bb
  ├── board
  └── actions
           │
           ▼
build_spot_key(...)
           │
           ▼
nlhe_cash:{positions}:{stack}:{texture}:{actions}
           │
           ├── GTO baseline lookup
           ├── Population lookup
           ├── Proprietary lookup
           ├── Deviation reports
           └── Version diff summaries
```

这意味着 `spot_key` 是这个 repo 里最关键的跨层契约。

## 7. 当前“已接入主链路”和“预留能力”对照图

```text
已接入主链路
──────────────────────────────────────────────
- FastAPI
- EngineService
- training artifacts on disk
- Next.js frontend
- CLI
- Rust parser generated CSV -> training input

预留 / 半接入能力
──────────────────────────────────────────────
- Postgres
- Redis
- Rust strategy engine as serving runtime
- PioSOLVER real-time or batch-fed baseline generation
- richer persistent storage beyond JSON artifacts
```

## 8. 数据存储现实图

```text
当前主存储
──────────────
JSON artifacts on disk

具体包括
──────────────
- solver/gto_baseline.json
- training/population_data.json
- training/proprietary_strategy.json
- training/versions/active.json
- training/versions/v*/strategy.json
- training/versions/v*/metadata.json

数据库基础设施
──────────────
- Postgres declared in docker-compose
- Redis declared in docker-compose

当前状态
──────────────
Prepared but not yet central to runtime serving path
```

## 9. 叙事

```text
1. 系统目标
   GTO baseline + population tendency + proprietary exploit strategy

2. 离线链路
   raw hands -> parser -> CSV -> population -> deviation -> adjustment -> validation -> version

3. 在线链路
   versioned artifacts -> EngineService -> FastAPI -> Web / CLI

4. 现状
   当前是 artifact-first serving
   DB / Redis / Rust engine / PioSOLVER integration 有布局，但不是主路径
```

## 10. 最后一句总结

```text
这个系统当前最准确的架构描述是：

一个以 spot_key 为统一索引、
以离线训练产物为在线服务核心、
同时预留更强 solver / storage / runtime 演进空间的
扑克策略研究与服务平台。
```
