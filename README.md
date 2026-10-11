# 中国商品期货贴水跟踪

跟踪 SHFE / DCE / CZCE / INE / GFEX 主要活跃商品的**主力连续**、活跃期限结构和近 3 年分位。

**在线页面：** https://zaviichen.github.io/commodity-basis/

静态站托管在 GitHub Pages。页面只读仓库里的 `data/latest.json`。GitHub Action 在每个工作日北京时间 16:00 运行 `scripts/fetch_basis.py`，把新浪公开行情写入快照和历史库存。

股指分红矫正贴水在姊妹页：https://zaviichen.github.io/cffex-basis/

## 主力是什么

每个品种**只有一条**连续序列，叫主力。某个交易日它等于当时最活跃的那一张合约（持仓加成交最高），不是近月，也不是另挂一条「交易所主力」。

```
得分 = 持仓量 + 成交量
主力 = 得分最高的未到期合约
```

得分相同：持仓量更高者优先，其次成交量，再次更近的到期月。

领先合约换手的那天，序列换到新合约上，并用比率拼接去掉换月缺口。页面上写的合约代码就是这条序列**当天的腿**。

## 次活跃远月，不是相邻月

对照腿是到期**晚于**主力、且得分最高的合约。豆粕常见 1 月对 5 月，而不是对紧邻的 3 月。曲线表里其余行是得分靠前的活跃月份，用来看期限结构，不代表第二条主力。

没有更远的活跃月份时，年化贴水为空。

## 贴水怎么定义

不使用现货。公开现货报价往往不是交割品（铁矿品牌、干湿吨、厂库升贴水），拿来减期货会假装精确。

```
价差     = 主力价 − 次活跃远月价          正 = 远月更便宜
年化贴水 = 价差 / 主力价 × 365 / 两合约到期日之差
```

- 正数 = 贴水（backwardation，近高远低）。多头从主力换到次活跃远月，是卖高买低。
- 负数 = 升水（contango）。多头换月支付升水税。
- 年化只用自然日 365，不扣利率，不用 252。
- 两个到期日相距不足 7 个自然日时不年化。
- 历史日线用结算价（结算为 0 时用收盘）。页面曲线用最新价。

到期日是工作日近似，**没有剔除法定节假日**（长假附近会差几天）：

| 规则 | 品种 |
|------|------|
| 合约月 15 日，遇周末顺延到周一 | 上期所螺纹、热卷、铜、铝、锌、镍、黄金、白银 |
| 交割月前一月的最后一个工作日 | 原油、燃料油、低硫燃料油 |
| 合约月第 10 个工作日 | 大商所、郑商所、广期所品种 |

## 两个分位

格子里只写百分数（`72%`）。样本不足 30 个交易日显示「—」。悬停可看样本数。

窗口都是行情日往前 3 个日历年，**含起始日、不含行情日当天**。

### 贴水分位

同一品种主力序列上，年化贴水 ≤ 今天的交易日占比。

**分位越高，远月折价越深**（比这段历史里更多的日子更贴水）。分位越低，贴水更浅或已经升水。

### 价格分位

比率后复权的主力价格 ≤ 今天可交易价的占比。拼接锚定在今天：最后一个点就是主力的现价，换月缺口不算进涨跌。

**分位越高，价格越接近近 3 年高位。** 这是价格位置，不是贴水。

## 品种

黑色 `i j jm rb hc`，油脂油料 `m y p OI RM`，有色贵金属 `cu al zn ni au ag`，能源 `sc fu lu`，化工 `TA MA SA FG UR eg l v pp`，农产品 `c cs CF SR AP`，广期所 `lc si`。

活跃月份由当天的持仓和成交决定，不写死 1/5/9。

## 本地运行

只需 Python 3.11+ 标准库。

```bash
python3 scripts/test_roll.py
python3 scripts/fetch_basis.py --rebuild-history   # 拉近三年日K并拼接，较慢
python3 scripts/fetch_basis.py                     # 只刷新快照并追加当天
python3 -m http.server 8741
```

打开 http://127.0.0.1:8741/ 。`--only m,rb` 只调试部分品种，结果写到 `data/debug.json`，不覆盖站点快照。

## 更新频率

`.github/workflows/update-basis.yml`

- 定时：`0 8 * * 1-5`（UTC）= 北京时间周一至周五 16:00，每日一次
- 也可在 Actions 页手动 `workflow_dispatch`
- 默认只拉最新报价并追加库存；全量重拉加 `--rebuild-history`
- `data/latest.json` 或 `data/history/` 有变化才提交

行情接口（均带 `Referer: https://finance.sina.com.cn`）：

- 快照：`https://hq.sinajs.cn/list=nf_M2701,...`
- 日 K：`InnerFuturesNewService.getDailyKLine`（字段 `s` 结算、`v` 成交、`p` 持仓）

## 目录

```
commodity-basis/
├── index.html
├── data/latest.json
├── data/history/series.json
├── scripts/universe.py
├── scripts/roll.py
├── scripts/fetch_basis.py
├── scripts/test_roll.py
├── .github/workflows/update-basis.yml
├── .nojekyll
└── README.md
```

## GitHub Pages

仓库公开，Pages 从 `main` 分支根目录 `/` 发布。站点地址为 https://zaviichen.github.io/commodity-basis/ 。

若 Pages 尚未打开：仓库 **Settings → Pages → Build and deployment → Deploy from a branch**，Branch 选 `main`、目录选 `/ (root)`，保存。

命令行（需要有 `repo` 权限的 GitHub 登录）：

```bash
gh repo create zaviichen/commodity-basis --public --source=. --remote=github --push
gh api --method POST /repos/zaviichen/commodity-basis/pages \
  --input - <<'EOF'
{"build_type":"legacy","source":{"branch":"main","path":"/"}}
EOF
```

## 为什么不能照搬股指

商品没有分红。曲线是仓储、资金成本和便利收益的结果，库存松了就会从贴水翻成升水税。交割月有限仓和仓单，个人客户往往不能把近月持有到最后交易日，纸面价差不一定成交得了。

远月便宜也可能是「看空远期」（供给扩张、需求见顶），不完全是现货紧缺的便利收益。铁矿还要区分品牌和干湿吨。豆粕、油脂、苹果的斜率里混着季节。黄金这类好储存的品种，持有成本下升水才是常态。

所以这里把年化贴水当成**可比较的曲线状态**，并给出它在近 3 年里的位置。它不是股指那种分红调整后的结构性折价，也不能当成稳定股息。

## 免责声明

仅供研究参考，不构成投资建议。行情来自新浪财经公开接口，可能延迟或缺失。到期日未逐日剔除法定节假日。
