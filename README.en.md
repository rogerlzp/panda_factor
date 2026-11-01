# PandaFactor — PandaAI 量化因子库

[![Python](https://img.shields.io/badge/python-3.9%2B-blue)](https://www.python.org/)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

> 一个完整的量化因子研究平台：高性能算子库 + 因子编写引擎 + 可视化 + LLM 集成 + 数据中枢。
> 纯 Python / pandas 实现，无 C++ 扩展依赖。

[中文文档](README.md) · **English** · [在线体验](https://www.pandaai.online/factorhub/factorcompetition)

---

## What this is

A quant factor platform in eight Python packages. The core is a library of **103 high-performance operators** (`DELAY`, `MA`, `RANK`, `STDDEV`, `CORRELATION`, `HHV`, `DECAY_LINEAR`, …) that you compose into factors — but the platform also covers factor analysis, a web front end, a data hub, an LLM service, and the server that ties them together.

```
panda_factor/              ← the factor engine
panda_factor_server/       ← factor computation service
panda_data/                ← market data providers & cleaning
panda_data_hub/            ← data ingestion hub
panda_common/              ← shared config, DB handlers, logging
panda_llm/                 ← LLM service (models / routes / services)
panda_web/                 ← web front end
server/                    ← deployment entry points
```

---

## Core design

### Operators are the foundation

`FactorUtils` implements every operator as a `@staticmethod`, and `Factor.__init__` binds them onto instances — so a factor file gets the full operator set without importing anything:

```python
from panda_factor.generate.factor_base import Factor

class ComplexFactor(Factor):
    def calculate(self, factors):
        close, volume = factors['close'], factors['volume']

        returns  = close / DELAY(close, 20) - 1          # 20-day momentum
        vol      = STDDEV(close / DELAY(close, 1) - 1, 20)
        mom      = RANK(returns)                          # cross-sectional rank
        signal   = IF(vol > DELAY(vol, 1), 1, -1)

        return mom * signal
```

`calculate()` returns a `pd.Series` indexed by `['symbol', 'date']`.

### Two authoring modes

**Python mode** — full programmability, the recommended path:

```python
class CustomFactor(Factor):
    def calculate(self, factors):
        return result
```

**Formula mode** — no programming required; the platform parses and evaluates the expression:

```
RANK((CLOSE / DELAY(CLOSE, 20)) - 1) * STDDEV((CLOSE / DELAY(CLOSE, 1)) - 1, 20)
```

Both modes produce the same typed output, so a team can mix them freely.

### Cross-sectional ranking, done correctly

`RANK` normalises each date's cross-section to `[-0.5, 0.5]` and groups by `date` — not a time-series rank. Getting this wrong is a classic silent look-ahead bias, so it lives in the shared base class rather than being reimplemented per factor.

---

## Data

The platform ships with **~5 years of built-in market data**. Ingestion runs nightly at 20:00 and refreshes automatically.

| Source | Status |
|---|---|
| Tushare | ✅ live |
| RiceQuant | ✅ live |
| Xuntou (迅投) | ✅ live |
| Tqsdk | 🧪 testing |
| QMT | 🧪 testing |
| Wind | 🔄 integrating |
| Choice | 🔄 integrating |

---

## Tech stack

| Layer | Technologies |
|---|---|
| Web | FastAPI, Uvicorn, Flask, Pydantic |
| Data | MongoDB, Redis, MySQL |
| Compute | NumPy, pandas, SciPy, statsmodels |
| Visualisation | matplotlib, seaborn |
| Market data | rqdatac, tqsdk, tushare, chinese_calendar |
| Packaging | Docker (app + server), `setup.py` per package |

---

## Running locally

**For individual traders** — a pre-built database archive is available on request (it is large); unpack it and run `bin/db_start.bat`. MongoDB is required for the team/installation path; point `panda_common/config.yaml` at your instance.

```bash
pip install -r requirements.txt
```

---

## Related work

- **`panda_factor_core`** — the factor engine extracted as a standalone, dependency-light library: `operators.py` (module-level operator functions), a `Factor` ABC, and a `backtest/` module (strategy base classes + performance metrics) that the platform repo does not include.
- **[`dbt_quant_demo`](https://github.com/rogerlzp/dbt_quant_demo)** — the same factor panel rebuilt as a dbt project (staging → intermediate → mart) with semantic metrics and data-quality gates as first-class models.
- **[`mcp_quant_server`](https://github.com/rogerlzp/mcp_quant_server)** — a read-only MCP server that lets an LLM agent query the mart, with agent-harness design (permission boundaries, tool descriptions written for model consumption, data quality as a callable tool).

Together they trace one architecture: **compute (pandas) → transform (dbt) → serve to agents (MCP)**.

---

## License

**AGPL-3.0** — see [LICENSE](LICENSE).

The standalone extraction ([`panda_factor_core`](https://github.com/rogerlzp/panda_factor_core))
is a derivative work and carries the same license.
