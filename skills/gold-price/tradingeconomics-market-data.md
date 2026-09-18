# Trading Economics 四项行情取数规则（默认）

用于金价、黄金日报、新闻汇总、当前走势和宏观联动分析的默认四项网页快照。页面是 Trading Economics 的网页参考报价：XAU 为 USD/盎司，DXY 为美元指数，不等同于 USD/CNY、官方基准或 COMEX 现货合约价格。

## 默认页面与字段

| 标的 | 页面 | 现价 | 日内字段 | 口径 |
|:---|:---|:---|:---|:---|
| XAU/USD | `https://zh.tradingeconomics.com/commodity/gold` | `market_last` | `market_daily_Pchg` | 相对日涨跌百分比 |
| DXY | `https://zh.tradingeconomics.com/united-states/currency` | `market_last` | `market_daily_Pchg` | 美元指数相对日涨跌百分比，不是 USD/CNY |
| Brent | `https://zh.tradingeconomics.com/commodity/brent-crude-oil` | `market_last` | `market_daily_Pchg` | 相对日涨跌百分比 |
| US10Y | `https://zh.tradingeconomics.com/united-states/government-bond-yield` | `market_last` | 优先 `market_daily_chg` | 收益率为百分比；绝对变化为百分点 `change_pp`，基点 `bp = change_pp * 100` |

`market_daily_Pchg` 为空或缺失时保持 `null`，不得用 `market_daily_chg` 推算相对百分比，也不得把 US10Y 的 Pchg 当作 change_pp。所有字段保留 `raw` 文本；空、缺失、非严格数字均为 `null`，不能把空字符串转成 0，也不能接受部分数字。正负号以文本为准，不能凭颜色判断方向。网页没有可靠源端报价时间时 `source_time_raw` 保持 `null`；HTTP `Date`/`Age`/`Last-Modified` 仅作为缓存元数据，不能冒充报价时间。四页失败相互隔离，不自动调用东方财富。

默认使用并行 HTTP 读取原始 HTML，不启动浏览器。每页请求超时 15 秒，失败最多重试 1 次；TLS 证书校验不得关闭。失败项标记待核，其余页面仍可返回。抓取北京时间必须由入口联网取时，不能用系统时钟填充。网页缓存新鲜度、源端报价时间和四页是否同一时点均未知时，不得称“实时”、精确 tick 或据此做分钟级因果判断；抓取时间也不能冒充源端时间。

黄金的 Trading Economics 数值继续按 `XAU-Gold-price-acquisition-rules.md` 与 GoldPrice.Today 或 gold-api 交叉验证：偏差超过 0.5% 引入第三个非东方财富来源，超过 1% 标异常；任一源不足则输出待核，不能称双源确认。OHLC 本规则不实现。仅用户明确要求东方财富报价、页面或核对时，才读取 `eastmoney-cloud-market-data.md`。

## Node 18+ 可运行示例

```js
const assert = require('node:assert/strict');

const pages = {
  xau: 'https://zh.tradingeconomics.com/commodity/gold',
  dxy: 'https://zh.tradingeconomics.com/united-states/currency',
  brent: 'https://zh.tradingeconomics.com/commodity/brent-crude-oil',
  us10y: 'https://zh.tradingeconomics.com/united-states/government-bond-yield',
};

function field(html, id) {
  const re = new RegExp('<span\\b[^>]*\\s+id\\s*=\\s*["\\\']' + id + '["\\\'][^>]*>([\\s\\S]*?)<\\/span>', 'i');
  const match = html.match(re);
  if (!match) return { raw: null, value: null };
  const raw = match[1].replace(/<[^>]*>/g, '')
    .replace(/&nbsp;|&#160;/gi, ' ').replace(/&minus;|&#8722;|−/gi, '-').trim();
  if (!raw) return { raw: '', value: null };
  const normalized = raw.replace(/%$/, '').trim();
  const numberPattern = /^[+-]?(?:(?:0|[1-9]\d*)(?:\.\d+)?|(?:\d{1,3}(?:,\d{3})+)(?:\.\d+)?)$/;
  const value = numberPattern.test(normalized) ? Number(normalized.replace(/,/g, '')) : null;
  return { raw, value: value !== null && Number.isFinite(value) ? value : null };
}

function emptySnapshot(key, status, error) {
  return { key, value: null, value_raw: null, change: null, change_raw: null,
    change_pct: null, change_pct_raw: null, change_pp: null, change_pp_raw: null,
    bp: null, source_time_raw: null, status, ...(error ? { error } : {}) };
}

function snapshot(html, key) {
  const last = field(html, 'market_last');
  const pct = field(html, 'market_daily_Pchg');
  const change = field(html, 'market_daily_chg');
  return {
    key, value: last.value, value_raw: last.raw,
    change_pct: key === 'us10y' ? null : pct.value,
    change: key === 'us10y' ? null : change.value,
    change_raw: change.raw,
    change_pct_raw: pct.raw,
    change_pp: key === 'us10y' ? change.value : null,
    change_pp_raw: key === 'us10y' ? change.raw : null,
    bp: key === 'us10y' && change.value !== null ? change.value * 100 : null,
    source_time_raw: null,
    status: last.value === null ? 'failed_missing_value'
      : (key === 'us10y' ? change.value : pct.value) === null
        ? 'partial_no_source_time' : 'ok_no_source_time',
  };
}

async function get(url, key) {
  for (let attempt = 0; attempt < 2; attempt++) {
    try {
      const response = await fetch(url, { signal: AbortSignal.timeout(15000) });
      if (!response.ok) throw new Error('HTTP ' + response.status);
      return { ...snapshot(await response.text(), key), attempts: attempt + 1 };
    } catch (error) {
      if (attempt === 1) return { ...emptySnapshot(key, 'failed', String(error)), attempts: attempt + 1 };
    }
  }
}

async function main() {
  const results = await Promise.all(Object.entries(pages).map(([key, url]) => get(url, key)));
  console.log(JSON.stringify(results, null, 2));
}

// Small contract self-check: missing/empty/zero/negative/thousands/invalid and US10Y pp→bp.
assert.deepEqual(field('<span id="x"></span>', 'x'), { raw: '', value: null });
assert.deepEqual(field('', 'x'), { raw: null, value: null });
assert.equal(field('<span data-id="x">123</span>', 'x').value, null);
assert.equal(field('<span id="x">0</span>', 'x').value, 0);
assert.equal(field('<span id="x">−1,234.50%</span>', 'x').value, -1234.5);
assert.equal(field('<span id="x">1,2</span>', 'x').value, null);
assert.equal(field('<span id="x">+-1</span>', 'x').value, null);
assert.equal(field('<span id="x">12abc</span>', 'x').value, null);
const us10y = snapshot('<span id="market_last">4.9810</span><span id="market_daily_Pchg">0.99%</span><span id="market_daily_chg">+0.0520</span>', 'us10y');
assert.equal(us10y.change_pct, null);
assert.equal(us10y.change_pct_raw, '0.99%');
assert.equal(us10y.change_pp, 0.052);
assert.equal(us10y.change_raw, '+0.0520');
assert.equal(us10y.bp, 5.2);
assert.equal(snapshot('<span id="market_last">100</span>', 'dxy').status, 'partial_no_source_time');

main();
```
