# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AlphaGPT 是一套"因子挖掘 + 实盘执行"的量化交易系统，核心思路是用 Transformer 模型自动生成可解释的因子公式（token 序列），通过回测打分筛选，再将高分公式用于实时扫描与交易执行。

项目包含两套独立的体系：
- **加密货币体系**（主流程）：面向 Solana meme 生态，完整的数据管线 + 模型训练 + 实盘执行
- **A股体系**（独立脚本）：`code/main.py` 和 `times.py`，使用 Tushare 数据源

## Commands

### 环境安装
```bash
pip install -r requirements.txt                    # 核心依赖
pip install -r requirements-optional.txt           # 可选依赖（A股脚本需要）
```

### 加密货币体系
```bash
# 1. 数据管线（需要 .env 配置 BIRDEYE_API_KEY）
python -m data_pipeline.run_pipeline

# 2. 模型训练（生成 best_meme_strategy.json）
python -m model_core.engine

# 3. 实盘运行（需要 Solana 钱包私钥配置）
python -m strategy_manager.runner

# 4. Dashboard
streamlit run dashboard/app.py
```

### A股体系
```bash
# 独立脚本，需要 Tushare Token
python times.py           # 带 Token 的版本
python code/main.py       # Token 需手动填入
```

## Architecture

### 两套体系的核心差异

| 维度 | 加密货币体系 | A股体系 |
|------|-------------|---------|
| 数据源 | Birdeye/DexScreener API + PostgreSQL | Tushare API + Parquet 缓存 |
| 特征维度 | 6-12维（含流动性、FDV等链上指标） | 5维（RET, RET5, VOL_CHG, V_RET, TREND） |
| 算子集 | 12个（含 GATE, JUMP, DECAY 等高级算子） | 11个（含时序算子 MA20, STD20 等） |
| 模型架构 | Looped Transformer + MTPHead + LoRD 正则化 | 标准 TransformerEncoder |
| 回测逻辑 | 考虑流动性冲击、滑点、Meme 特性 | Open-to-Open 收益 + Sortino 评分 |
| 执行层 | Solana RPC + Jupiter 聚合器 | 无（仅回测） |

### 共享的核心设计

**公式表示**：Token 序列 = 特征 + 算子，使用 Polish Notation 编码，由 StackVM 执行成因子信号

**训练流程**：
1. Transformer 生成公式 token 序列
2. StackVM 执行公式得到因子值
3. 回测评分作为 reward
4. Policy Gradient 更新生成器

### 模块职责

```
model_core/
├── alphagpt.py    # Transformer 模型定义（含 LoRD、MTPHead 等高级组件）
├── ops.py         # 算子定义（OPS_CONFIG）
├── factors.py     # 特征工程（FeatureEngineer, MemeIndicators）
├── vm.py          # StackVM 公式执行器
├── backtest.py    # 回测评估（MemeBacktest）
├── engine.py      # 训练主循环（AlphaEngine）
└── data_loader.py # 数据加载

data_pipeline/
├── providers/     # 数据源适配器（Birdeye, DexScreener）
├── fetcher.py     # 异步数据拉取
├── processor.py   # 数据清洗
└── db_manager.py  # PostgreSQL/TimescaleDB 写入

strategy_manager/
├── runner.py      # 实盘主循环
├── portfolio.py   # 持仓管理（JSON 持久化）
├── risk.py        # 风控引擎
└── config.py      # 策略参数（止损、止盈、仓位）

execution/
├── trader.py      # 交易执行（买入/卖出）
├── jupiter.py     # Jupiter 聚合器封装
└── rpc_handler.py # Solana RPC 客户端
```

### 关键配置

**加密货币体系** (`model_core/config.py`):
- `MAX_FORMULA_LEN = 12`：公式最大长度
- `BATCH_SIZE = 8192`：训练批次
- `MIN_LIQUIDITY = 5000`：最低流动性阈值
- `BASE_FEE = 0.005`：基础费率

**A股体系** (`times.py` / `code/main.py`):
- `MAX_SEQ_LEN = 8`：公式最大长度
- `COST_RATE = 0.0005`：交易成本（万五）
- 80/20 训练/测试集划分

**策略参数** (`strategy_manager/config.py`):
- `BUY_THRESHOLD = 0.85`：买入信号阈值
- `SELL_THRESHOLD = 0.45`：卖出信号阈值
- `STOP_LOSS_PCT = -0.05`：止损线
- `MAX_OPEN_POSITIONS = 3`：最大持仓数

## External Dependencies

- **PostgreSQL/TimescaleDB**：加密货币数据存储
- **Birdeye API**：Solana 代币数据
- **Jupiter Aggregator**：DEX 聚合交易
- **Solana RPC**：链上交互
- **Tushare**：A股数据（需要 Token）

## Key Files

- `best_meme_strategy.json`：训练输出的最优公式（需先运行训练）
- `training_history.json`：训练过程记录
- `data_cache_final.parquet`：A股数据缓存
- `.env`：环境变量配置（API Keys、数据库连接、钱包私钥）
