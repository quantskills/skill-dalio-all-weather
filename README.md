# A 股全天候资产配置研究

> 用宏观增长—通胀象限、反波动率风险预算和季度再平衡，研究 A 股环境下的全天候资产配置。

![Python](https://img.shields.io/badge/Python-3.11%20%7C%203.12-3776AB)
![License](https://img.shields.io/badge/License-MIT-green)
![Project](https://img.shields.io/badge/QUANTSKILLS-Community%20Project-blue)

## 项目简介

本项目把公开的全天候配置思想落到中国市场数据上，覆盖股票、长债、中债、黄金和商品五类资产。它可以生成当前配置、增长—通胀象限诊断和带交易成本的季度回测，并把数据来源、决策日期、实际执行日期、权重、换手和指纹完整保存下来。

默认配置以 30% 股票、40% 长债、15% 中债、7.5% 黄金和 7.5% 商品为基准，再依据过去 60 个交易日的波动率调整风险预算。这里采用的是反波动率方法，不是通过协方差优化求解的严格等风险贡献（ERC）。

本项目只用于研究与教学，不构成投资建议、交易指令或实盘许可。

## 适用场景

- 生成指定日期的五类资产目标配置和风险贡献诊断；
- 判断增长上行/下行与通胀上行/下行四种宏观环境；
- 复核季度再平衡、持仓漂移、换手门槛和交易成本；
- 比较不同截止日、波动窗口或成本假设；
- 留存可重复核验的数据与源码指纹。

## 方法概览

### 基准配置

| 资产 | 基准权重 | 主要对应环境 |
|---|---:|---|
| 股票 | 30% | 增长上行 |
| 长债 | 40% | 增长下行、通胀下行 |
| 中债 | 15% | 增长下行 |
| 黄金 | 7.5% | 通胀上行 |
| 商品 | 7.5% | 通胀上行 |

每个目标日只使用前一交易日及更早的数据：

```text
raw_i = quadrant_base_i / trailing_volatility_i
target_i = raw_i / sum(raw)
```

系统另用同一 60 日窗口的协方差矩阵计算实际风险贡献：

```text
RC_i = w_i × (Σw)_i / (w'Σw)
```

报告会列出每类资产的风险贡献、最大绝对贡献和 HHI 集中度。集中度过高会触发警示，但不会被表述为“等风险”。

### 宏观象限

| 方向 | PandaData 接口 | 默认指标 | 可用时间近似 |
|---|---|---|---|
| 通胀 | `get_macro_pi` | `PI0000047`，CPI 同比 | 报告期后 15 个自然日 |
| 增长 | `get_macro_na` | `NA0000014`，不变价 GDP 累计同比 | 报告期后 30 个自然日 |
| 利率 | `get_macro_ir` | 用户显式指定单个代码 | 仅作可选诊断 |

象限信号使用 63 个交易日变化、60 日多数平滑和 20 日磁滞。供应商未提供宏观数据的历史发布版本，因此公布滞后只能减少期末前视，不能消除历史修订偏差。

四个象限使用下列偏好权重作为反波动率计算前的基准：

| 象限 | 股票 | 长债 | 中债 | 黄金 | 商品 |
|---|---:|---:|---:|---:|---:|
| 增长上行、通胀上行 | 25% | 20% | 15% | 20% | 20% |
| 增长上行、通胀下行 | 40% | 30% | 15% | 7.5% | 7.5% |
| 增长下行、通胀上行 | 15% | 25% | 20% | 25% | 15% |
| 增长下行、通胀下行 | 20% | 50% | 20% | 5% | 5% |

### 再平衡与记账

- 每个完整季度的最后共同交易日形成目标；
- 残缺的首季度不形成初始目标；
- 最后一个季度只有在查询窗口到达自然季末后才确认；
- 目标在下一共同交易日执行；
- 非执行日权重随资产收益自然漂移；
- 换手按“漂移后的实际权重到新目标”的 L1 距离计算；
- 默认按 `L1 换手 × cost_bps` 扣除一次成本。

默认情况下，季度内的象限变化只记录、不立即调仓。只有显式启用 `--rebalance-on-regime-change` 才增加象限切换调仓。

## 数据接口与替代链

项目使用 PandaData 的 `get_fund_daily`、`get_index_daily`、`get_future_daily`、`get_macro_pi`、`get_macro_na`，并可选使用 `get_macro_ir`。

仓库不附带行情、宏观数据、缓存或第三方安装包。使用者需要自行取得 PandaData 访问权限，并遵守相应的数据授权和使用条款。`panda_data==0.0.12` 的公开包元数据没有给出软件许可证或公开源码地址，相关边界见 [第三方说明](THIRD_PARTY_NOTICES.md)。

| 资产 | 逐日回退顺序 |
|---|---|
| 股票 | `510300.SH / 510500.SH → 000300.SH / 000905.SH` |
| 长债 | `511260.SH → 000012.SH → T_DOMINANT.CFE` |
| 中债 | `511010.SH → 000012.SH → TF_DOMINANT.CFE` |
| 黄金 | `518880.SH → AU_DOMINANT.SHF` |
| 商品 | `501018.SH / 159985.SZ → 000827.SH → CU_DOMINANT.SHF / M_DOMINANT.DCE` |

收益优先按 `log(close/pre_close)` 计算；接口没有 `pre_close` 时才退到 `log(close).diff()`。实际采用的代码、接口和收益口径会写入 `used_symbols.json`。

`000012.SH` 是广义国债指数替代，不代表精确的 10 年或 5 年久期。中国市场的长久期债券工具和商品广度与经典美国全天候组合并不相同。

## 快速开始

已验收环境为 Windows、PowerShell、Python 3.11 和 3.12。当前版本要求 Python >=3.11 且 <3.13；Linux 与 macOS 尚未纳入正式验收。

```powershell
git clone https://github.com/quantskills/skill-dalio-all-weather.git
Set-Location skill-dalio-all-weather

py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --require-hashes -r requirements-lock.txt
```

真实数据连接必须显式配置由 PandaData 数据服务方提供并维护的受信 HTTPS 地址。锁定的第三方库自带默认地址使用 HTTP，本项目会直接拒绝；使用者和本仓库不能代替服务方开启 HTTPS，也不存在关闭此限制的兼容开关。

```text
PANDADATA_BASE_URL=https://服务方提供的地址
```

认证可使用短期令牌：

```text
PANDADATA_TOKEN=你的短期令牌
```

也可使用账号密码在 HTTPS 上换取进程内令牌：

```text
PANDADATA_USER=你的账号
PANDADATA_PASSWORD=你的密码
```

配置只从当前进程环境变量或用户级 `~/.pandadata.env` 读取；项目目录和当前工作目录中的 `.env` 不会被读取。真实凭证、令牌、缓存和私有结果都不应提交到仓库。

为规避第三方库的凭证落盘行为，命令会在整个进程内禁用 `user.json` 的读取、写入、自动登录和删除，强制证书校验，并拒绝重定向。账号密码不会进入数据客户端。令牌到期后，命令不会从磁盘自动刷新；请结束进程，更新令牌或重新认证后再运行。

先运行启动健康检查。它会检查 Python 版本、SDK 兼容性、DNS、TLS 握手、证书、主机名和凭证配置，但不会显示凭证；随后再检查真实登录：

```powershell
python scripts\cli.py health-check
python scripts\cli.py check-login

# 服务方排查 HTTPS 时，可暂时只检查运行环境和传输
python scripts\cli.py health-check --transport-only

python scripts\cli.py current-allocation --end 20260710

python scripts\cli.py backtest `
  --start 20180101 --end 20260710 `
  --rebalance Q --vol-window 60 --cost-bps 5 `
  --turnover-threshold 0.05

python scripts\cli.py diagnose-regime `
  --start 20100101 --end 20260710
```

利率诊断必须显式给出单个代码，避免无条件读取整张利率表：

```powershell
python scripts\cli.py diagnose-regime `
  --start 20100101 --end 20260710 `
  --ir-symbol IR_EXPLICIT
```

作为 Skill 使用时，可直接提出：

```text
以 2026-07-10 为截止日，生成 A 股全天候当前配置和风险贡献诊断。
回测 2018-01-01 至 2026-07-10 的季度全天候组合，并说明数据替代和限制。
```

## 数据质量与复现约定

- `stock / bond_long / bond_mid / gold / commodity` 五类资产缺一不可；
- 每类资产在并集日历上的覆盖率必须至少为 80%；
- 通过覆盖率检查后，只保留五类资产共同有收益的交易日；
- 回测额外读取请求起点前 400 个自然日作为预热，预热收益不计入绩效；
- 每次运行记录配置、实际数据链、依赖版本、数据 SHA-256 和源码 SHA-256；
- 相同参数目录已存在但数据或源码指纹不同时，程序拒绝覆盖。

## 输出

当前配置输出：

- `allocation_YYYYMMDD.csv`
- `allocation_YYYYMMDD_metadata.json`

回测目录：

```text
backtest_YYYYMMDD_<run_id>/
├── metrics.json
├── report.md
├── equity.csv
├── equity.png
├── daily_return.csv
├── weights_used.csv
├── turnover.csv
├── decision_dates.csv
├── target_weights.csv
├── executed_rebalances.csv
├── by_quadrant.csv
└── used_symbols.json
```

`decision_dates.csv` 和 `target_weights.csv` 用于独立核验完整季度末的决策、样本内下一共同交易日和目标权重。`run_id` 绑定日期、频率、波动窗口、成本、换手门槛和象限切换开关，避免不同研究配置互相覆盖。

## 已知限制

- 宏观历史修订版本不可得，仍可能存在修订值偏差；
- A 股缺乏与美国市场完全对应的长久期国债 ETF；
- 广义国债指数不能复制精确久期，商品替代也不是完整商品指数；
- `pre_close` 不能替代严格复权、期货展期和可交易价格研究；
- 保证金、展期细节、滑点、税费、容量和停牌未完整模拟；
- 成本采用权重变化的研究近似；
- 历史回测和测试通过都不保证未来收益。
- 真实数据运行依赖 PandaData 数据服务方提供并维护可访问的 HTTPS 地址；只有 HTTP 地址时不会连接。
- 第三方 PandaData 包的许可证和公开维护来源未在其包元数据中说明，使用者需自行确认授权。

## 验证

离线测试不会读取用户级凭证，也不会调用真实网络接口：

```powershell
python -m pytest tests -q
python -m compileall -q scripts tests
```

锁定依赖的算法与安全测试可离线复现；真实数据验收属于有条件验证，需要有效数据权限和服务方 HTTPS 端点。发布前还应运行代码规范、安全扫描和依赖漏洞审计。

详细定义见 [研究方法](references/methodology.md) 和 [数据映射](references/data-mapping.md)。

## 项目状态与维护

本仓库目前为 beta，真实数据能力处于有条件验证状态。本仓库是 QUANTSKILLS Community Project，初始维护者为 Duzey；后续问题和变更通过 GitHub Issues 与 Pull Requests 处理。它不代表 QUANTSKILLS 官方认证、验证或背书。

方法思想来源于 Ray Dalio 和 Bridgewater 公开介绍的全天候配置理念。本项目是独立研究实现，与 Ray Dalio 或 Bridgewater Associates 无隶属、合作或授权关系。

## License

本项目采用 [MIT License](LICENSE)。
