# Taiwan equity API integration research

Verified 2026-10-29 by unauthenticated HTTPS GETs from this workspace and official schemas/pages. Research only; no application code. All indices below are **zero-based**. “Live” means JSON was actually retrieved; “schema” means only the official schema was checked. TPEx OpenAPI checks inspected the first complete object of a response, not necessarily the entire payload. No credentials, alternate identities, or access-control circumvention were used.

## Integration decisions

- **Latest OHLCV:** TWSE `STOCK_DAY_ALL`; TPEx `tpex_mainboard_quotes`. Both have observed `Date="1151028"`. TPEx historical `/afterTrading/otc` matches the latter's volume scope. Do not switch to `tpex_mainboard_daily_close_quotes` without changing the series definition: it includes additional trading sessions/types and has different volume.
- **Bulk history:** one request per market per trading day, not one request per symbol. TWSE `MI_INDEX` worked for 2026-10-27 and 2026-10-28; 2025-01-02 failed with HTTP 307. TPEx `/afterTrading/otc` worked for 2025-01-02 and 2026-10-28. This verifies selected dates, not uninterrupted coverage.
- **Institutional:** TWSE dated `T86`; TPEx dated `/insti/dailyTrade` or latest `tpex_3insti_daily_trading`. These are all-symbol tables containing per-symbol rows, not market-wide totals. Historical TWSE `T86` for 2026-10-27 failed with HTTP 307 even though 2026-10-28 worked.
- **Margin:** prefer dated TWSE `/marginTrading/MI_MARGN` to attach a verified trading date; its OpenAPI counterpart has no date. TPEx latest and dated margin tables both work. Margin quantities are trading units/lots, unlike institutional and OHLCV share counts.
- Persist successful dated data and display actual coverage. Older TWSE failures are observed access failures of undetermined cause, **not proof of a published historical cutoff**. No free multi-year archive download or guaranteed historical service was verified.

## Official schemas and request conventions

| Market | Official schema | API base |
|---|---|---|
| TWSE | <https://openapi.twse.com.tw/v1/swagger.json> | `https://openapi.twse.com.tw/v1` |
| TPEx | <https://www.tpex.org.tw/openapi/swagger.json> | `https://www.tpex.org.tw/openapi/v1` |

All OpenAPI paths below are GETs with **no date/range parameters declared**. Treat them as current snapshots, even where the description says “依日期查詢”; adding `date` does not create a documented historical API. TWSE Swagger describes some responses as objects, but live endpoints returned arrays of objects. Numeric values are strings.

TWSE website reports use `https://www.twse.com.tw/rwd/zh`; TPEx website reports use `https://www.tpex.org.tw/www/zh-tw`. These are official public website backends, not the OpenAPI contracts. TPEx routes were confirmed in the corresponding official HTML `tables.init({action:...})` configuration and `/rsrc/js/main.js` (`API_PATTERN="/www/{LANG}/{ACTION}"`). No API key was required for the successful tests. Browser CORS, production Worker egress, rate limits, and service guarantees were not verified.

## 1. Latest bulk OHLCV

Live URLs:

- TWSE: <https://openapi.twse.com.tw/v1/exchangeReport/STOCK_DAY_ALL> — full response parsed, 1,386 rows.
- TPEx: <https://www.tpex.org.tw/openapi/v1/tpex_mainboard_quotes> — live first object.
- TPEx alternative: <https://www.tpex.org.tw/openapi/v1/tpex_mainboard_daily_close_quotes> — live first object; a full-body attempt ended in `IncompleteRead`, so complete download reliability was not established.

| Canonical field | TWSE exact key | TPEx exact key |
|---|---|---|
| trading date | `Date` | `Date` |
| symbol / name | `Code` / `Name` | `SecuritiesCompanyCode` / `CompanyName` |
| open / high / low / close | `OpeningPrice` / `HighestPrice` / `LowestPrice` / `ClosingPrice` | `Open` / `High` / `Low` / `Close` |
| volume in shares | `TradeVolume` | `TradingShares` |
| traded value | `TradeValue` | `TransactionAmount` |
| trade count | `Transaction` | `TransactionNumber` |
| price change | `Change` | `Change` |

Actual TWSE object:

```json
{"Date":"1151028","Code":"00400A","Name":"主動國泰動能高息","TradeVolume":"24640364","TradeValue":"361129945","OpeningPrice":"14.54","HighestPrice":"14.73","LowestPrice":"14.46","ClosingPrice":"14.71","Change":"0.5300","Transaction":"4483"}
```

Actual TPEx `quotes` object:

```json
{"Date":"1151028","SecuritiesCompanyCode":"00411A","CompanyName":"主動統一前沿科技","Close":"10.21","Change":"+0.30","Open":"10.10","High":"10.25","Low":"10.10","TradingShares":"7174000","TransactionAmount":"73034400","TransactionNumber":"800","LatestBidPrice":"10.20","LatesAskPrice":"10.21","Capitals":"460576000","NextLimitUp":"9999.95","NextLimitDown":"0.01"}
```

`LatesAskPrice` is the official spelling. The alternative `daily_close_quotes` adds `Average` and `NextReferencePrice`; for the **same symbol/date** its `TradingShares="7233250"`, `TransactionAmount="73638449"`, `TransactionNumber="1147"`. The [official extended-session report](https://www.tpex.org.tw/web/stock/aftertrading/daily_close_quotes/stk_quote_result.php?l=zh-tw&o=htm) identifies its coverage as 等價、零股、盤後、鉅額交易. Do not splice these two volume series.

“All” includes securities beyond ordinary company shares. Preserve alphanumeric/leading-zero symbols; use an explicit universe filter. `ALLBUT0999` and TPEx `EW` exclude warrants/bull-bear certificates, not ETFs. Company master alone is not a complete security master.

## 2. Historical daily bulk: exact requests and layouts

### TWSE MI_INDEX

Verified request: <https://www.twse.com.tw/rwd/zh/afterTrading/MI_INDEX?date=20261028&type=ALLBUT0999&response=json>

Parameters: Gregorian `date=YYYYMMDD`; `type=ALLBUT0999`; `response=json`. Root shape: `{tables:[...],type,params,stat:"OK",date:"20261028"}`. **Find the table by fields containing `證券代號`, `開盤價`, `成交股數`; do not hardcode `tables[8]`** or use the market-summary table. `tables[8]` happened to be the quote table in the tested response. Its `data` contained 1,386 rows; table count/order and history can vary.

Exact quote-table fields:

```text
0 證券代號       1 證券名稱       2 成交股數       3 成交筆數
4 成交金額       5 開盤價         6 最高價         7 最低價
8 收盤價         9 漲跌(+/-)     10 漲跌價差      11 最後揭示買價
12 最後揭示買量 13 最後揭示賣價 14 最後揭示賣量  15 本益比
```

Actual first row, preserving strings:

```json
["00400A","主動國泰動能高息","24,640,364","4,483","361,129,945","14.54","14.73","14.46","14.71","<p style= color:red>+</p>","0.53","14.71","287","14.72","48","0.00"]
```

Use root `date`, OHLC indices `5,6,7,8`, shares `2`, value `4`, trades `3`. Sign in index 9 may contain HTML: extract text, never inject it into UI. Index 10 alone is not a signed price change. The OpenAPI `/exchangeReport/MI_INDEX` is described as **market summary**, not a replacement for this website quote table.

Access matrix: `20261028` OK; `20261027` OK (1,386 quote rows); `20250102` HTTP 307 with no usable redirect location. The [official report page](https://www.twse.com.tw/zh/trading/historical/mi-index.html) says coverage starts ROC 93/02/11 = **2004-02-11**. That advertised coverage does not establish automated access from this deployment.

Single-symbol monthly alternative: <https://www.twse.com.tw/rwd/zh/afterTrading/STOCK_DAY?date=20260901&stockNo=2330&response=json>. This test returned HTTP 307, so no response layout was validated. The [official page](https://www.twse.com.tw/zh/trading/historical/stock-day.html?stockNo=4739) advertises data from **2010-01-04**. Do not make this unverified fallback a prerequisite for bulk ingestion.

### TPEx afterTrading/otc — verified older-date bulk route

Verified requests:

- <https://www.tpex.org.tw/www/zh-tw/afterTrading/otc?date=2026/10/28&type=EW&response=json>
- <https://www.tpex.org.tw/www/zh-tw/afterTrading/otc?date=2025/01/02&type=EW&response=json>

Parameters: Gregorian slash date `YYYY/MM/DD`; `type=EW` = all securities excluding warrants/bull-bear certificates; `response=json`. Official form also offers `AL` = all securities, but AL was not tested. Root `{tables,date,flagField,stat:"ok"}`; root date is Gregorian compact `YYYYMMDD`. Find the table with title `上櫃股票每日收盤行情(不含定價)` and quote fields. `tables[0].data` held **1,020 rows** on 2026-10-28 and **943 rows** on 2025-01-02; `totalCount` matched returned row counts. No pagination argument was needed.

Exact 2026 field array, including whitespace/HTML:

```json
["代號","名稱","收盤 ","漲跌","開盤 ","最高 ","最低","成交股數  "," 成交金額(元)"," 成交筆數 ","最後買價","最後買量<br>(張數)","最後賣價","最後賣量<br>(張數)","發行股數 ","次日漲停價 ","次日跌停價"]
```

Map symbol `0`, name `1`, close `2`, signed change `3`, open/high/low `4/5/6`, shares `7`, value `8`, count `9`. In 2025, fields 11/13 used `(千股)` instead of `(張數)`; OHLCV positions matched. Actual first 2026 row:

```json
["00411A","主動統一前沿科技","10.21","+0.30","10.10","10.25","10.10","7,174,000","73,034,400","800","10.20","890","10.21","30","460,576,000","9,999.95","0.01"]
```

The [official page](https://www.tpex.org.tw/zh-tw/mainboard/trading/info/mi-pricing.html) advertises **2007-07 onward** and links a separate Jan–Jun 2007 archive. These earlier boundaries were not downloaded. `tpex_mainboard_quotes` and this route match the tested OHLCV row. Historical `daily_close_quotes` extended-session route was not verified.

## 3. Institutional flow per symbol

### TWSE T86 — live dated bulk

<https://www.twse.com.tw/rwd/zh/fund/T86?date=20261028&selectType=ALLBUT0999&response=json>

Gregorian `YYYYMMDD`; root `{stat:"OK",date,title,hints,fields,data,selectType,notes,total}`. **1,351** rows observed. Use `fields` and `data`, not `tables`. Volumes are **shares**. Exact columns:

| Index | Exact field |
|---|---|
| 0 / 1 | `證券代號` / `證券名稱` |
| 2 / 3 / 4 | `外陸資買進股數(不含外資自營商)` / `外陸資賣出股數(不含外資自營商)` / `外陸資買賣超股數(不含外資自營商)` |
| 5 / 6 / 7 | `外資自營商買進股數` / `外資自營商賣出股數` / `外資自營商買賣超股數` |
| 8 / 9 / 10 | `投信買進股數` / `投信賣出股數` / `投信買賣超股數` |
| 11 | `自營商買賣超股數` |
| 12 / 13 / 14 | `自營商買進股數(自行買賣)` / `自營商賣出股數(自行買賣)` / `自營商買賣超股數(自行買賣)` |
| 15 / 16 / 17 | `自營商買進股數(避險)` / `自營商賣出股數(避險)` / `自營商買賣超股數(避險)` |
| 18 | `三大法人買賣超股數` |

Map foreign net `4`, trust net `10`, dealer net `11`, combined net `18`. Do not add foreign-dealer column `7` again: combined net is `4+10+11`. Actual sample: symbol `2409`, foreign net `68,585,188`, trust `181,713`, dealer `9,162,727`, combined `77,929,628`. Trim trailing spaces from names. The [official report](https://www.twse.com.tw/zh/trading/foreign/t86.html) advertises **2012-05-02** onward and includes regular, odd-lot, after-hours fixed-price and block trades. Older tested date `20261027` failed with HTTP 307. No TWSE OpenAPI T86 path was found; `MI_QFIIS_*` holdings and market-total statistics are not substitutes.

### TPEx latest OpenAPI — exact awkward keys

<https://www.tpex.org.tw/openapi/v1/tpex_3insti_daily_trading> — live first object, `Date="1151028"`, identity `SecuritiesCompanyCode` / `CompanyName`. All quantities below are shares; preserve exact spaces/capitalization.

| Meaning | Exact JSON key |
|---|---|
| Foreign buy, excluding foreign dealers | `Foreign Investors include Mainland Area Investors (Foreign Dealers excluded)-Total Buy` |
| Foreign sell, excluding foreign dealers | ` Foreign Investors include Mainland Area Investors (Foreign Dealers excluded)-Total Sell` **(one leading space)** |
| Foreign net, excluding foreign dealers | `Foreign Investors include Mainland Area Investors (Foreign Dealers excluded)-Difference` |
| Foreign dealer buy / sell / net | `Foreign Dealers-Total Buy` / `Foreign Dealers-TotalSell` / `ForeignDealers-Difference` |
| Foreign inclusive buy / sell / net | `ForeignInvestorsIncludeMainlandAreaInvestors-TotalBuy` / `ForeignInvestorsIncludeMainlandAreaInvestors-TotalSell` / `ForeignInvestorsInclude MainlandAreaInvestors-Difference` |
| Trust buy / sell / net | `SecuritiesInvestmentTrustCompanies-TotalBuy` / `SecuritiesInvestmentTrustCompanies-TotalSell` / `SecuritiesInvestmentTrustCompanies-Difference` |
| Dealer total buy / sell / net | `Dealers-TotalBuy` / `Dealers-TotalSell` / `Dealers-Difference` |
| Dealer hedge sell only | `Dealers -TotalSell` **(space before hyphen)** |
| Combined net | `TotalDifference` |

Observed `00411A`: foreign net `1284000`, trust `0`, dealer `-1102021`, combined `181979`. Raw first object was checked for duplicate keys: none. Do not infer a full dealer own-account/hedging breakdown from this schema; use the website table below.

### TPEx dated institutional table — complete dealer split

<https://www.tpex.org.tw/www/zh-tw/insti/dailyTrade?date=2026/10/28&type=Daily&response=json>

Live: root `{columnNum,tables,template,date:"20261028",stat:"ok"}`; data table contains **905** rows. Field labels repeat `買進股數`, `賣出股數`, `買賣超股數`; use grouped positions, not a dictionary keyed only by labels. Group meanings verified against the official [page's multi-row header](https://www.tpex.org.tw/zh-tw/mainboard/trading/major-institutional/detail/day.html):

| Indices | Meaning (triples are buy / sell / net) |
|---|---|
| 0 / 1 | symbol / name |
| 2–4 | foreign excluding foreign dealers |
| 5–7 | foreign dealers |
| 8–10 | foreign inclusive |
| 11–13 | investment trusts |
| 14–16 | dealers, own account |
| 17–19 | dealers, hedging |
| 20–22 | dealers, total |
| 23 | combined institutional net |

Canonical net positions: foreign `4`, trust `13`, dealer `22`, combined `23`; combined = `4+13+22`. The page points to different historical layouts before **2014-12-01**; do not apply this 24-column parser unconditionally to older dates. Older-date requests to this route were not tested.

## 4. Margin per symbol

### TWSE dated report — preferred

<https://www.twse.com.tw/rwd/zh/marginTrading/MI_MARGN?date=20261028&selectType=ALL&response=json>

Live root `{stat:"OK",date:"20261028",tables:[...]}`. The first table is **market totals**, not securities. Select the table with `代號`, `名稱`, `資券互抵` and title containing `融資融券彙總`. **1,304** rows on 2026-10-28; **1,302** on 2026-10-27. Exact 16-column data layout:

| Index | Meaning / raw header |
|---|---|
| 0 / 1 | symbol `代號` / name `名稱` |
| 2 / 3 / 4 | margin buy `買進` / sell `賣出` / cash redemption `現金償還` |
| 5 / 6 / 7 | margin previous `前日餘額` / current `今日餘額` / next-day limit `次一營業日限額` |
| 8 / 9 / 10 | short covering `買進` / short sell `賣出` / stock redemption `現券償還` |
| 11 / 12 / 13 | short previous `前日餘額` / current `今日餘額` / next-day limit `次一營業日限額` |
| 14 / 15 | offset `資券互抵` / `註記` |

Headers duplicate between margin and short groups. Never zip them into one object without group prefixes. Actual `00400A` row:

```json
["00400A","主動國泰動能高息","268","209","2","14,475","14,532","533,160","0","0","0","0","0","533,160","6"," "]
```

Quantities are trading units, not shares; the report's market summary explicitly labels `融資(交易單位)` separately from money totals. Preserve `unit=trading_unit`; do not blanket-multiply by 1,000 across every security class. Balance change = current minus previous, not purchases alone; corporate actions/redemptions may also affect balances.

### TWSE latest OpenAPI alternative

<https://openapi.twse.com.tw/v1/exchangeReport/MI_MARGN> — full array parsed, 1,304 rows. Exact keys:

```text
股票代號 股票名稱
融資買進 融資賣出 融資現金償還 融資前日餘額 融資今日餘額 融資限額
融券買進 融券賣出 融券現券償還 融券前日餘額 融券今日餘額 融券限額
資券互抵 註記
```

**No `Date` key.** Record fetch time but do not label the data with another endpoint's date. In the observed `00400A` row, short quantities were `""`, while the dated report returned `"0"`; preserve unknown/blank separately from confirmed zero unless reconciliation is explicit.

### TPEx latest and historical-capable route

- Latest: <https://www.tpex.org.tw/openapi/v1/tpex_mainboard_margin_balance> — live first object, date `1151028`.
- Dated: <https://www.tpex.org.tw/www/zh-tw/margin/balance?date=2026/10/28&response=json> — live full JSON, root `{date:"20261028",tables,stat:"ok"}`, **919** rows. Historical dates on this route were not tested.

| Canonical field | Exact OpenAPI key | Dated row index |
|---|---|---|
| symbol / name | `SecuritiesCompanyCode` / `CompanyName` | 0 / 1 |
| margin previous | `MarginPurchaseBalancePreviousDay` | 2 |
| margin buy / sell / cash repayment | `MarginPurchase` / `MarginSales` / `CashRedemption` | 3 / 4 / 5 |
| margin current | `MarginPurchaseBalance` | 6 |
| margin finance-company portion | `MarginPurchaseBalanceBelongSecuritiesFinanceEnterprise` | 7 |
| margin utilization % / quota | `MarginPurchaseUtilizationRate` / `MarginPurchaseQuota` | 8 / 9 |
| short previous | `ShortSaleBalancePreviousDay` | 10 |
| short sell / cover / stock repayment | `ShortSale` / `ShortConvering` / `StockRedemption` | 11 / 12 / 13 |
| short current | `ShortSaleBalance` | 14 |
| short finance-company portion | `ShortSaleBalanceBelongSecuritiesFinanceEnterprise` | 15 |
| short utilization % / quota | `ShortSaleUtilizationRate` / `ShortSaleQuota` | 16 / 17 |
| offset / note | `Offsetting` / `Note` | 18 / 19 |

`ShortConvering` is the actual misspelling. Dated field labels explicitly say `前資餘額(張)`, `前券餘額(張)`, `資券相抵(張)`; utilization is percent. Sample `00411A`: margin previous/current `13160/13476`, short previous/current `91/1`. The [official page](https://www.tpex.org.tw/zh-tw/mainboard/trading/margin-trading/transactions.html) advertises **2007-01** onward and a separate 2003-08–2006-12 archive.

## 5. Valuations and company master

| Dataset | Official endpoint | Exact fields needed / verification |
|---|---|---|
| TWSE valuation | <https://openapi.twse.com.tw/v1/exchangeReport/BWIBBU_d> | Live, 1,087 rows: `Date`, `Code`, `Name`, `ClosePrice`, `DividendYield`, `DividendYear`, `PEratio`, `PBratio`, `FiscalYearQuarter`. **Date is Gregorian `20261028`**, unlike STOCK_DAY_ALL. Sample `1101`: close `28.00`, yield `2.86`, dividend year `114`, PE `""`, PB `0.91`, fiscal quarter `2026Q2`. |
| TWSE smaller valuation schema | <https://openapi.twse.com.tw/v1/exchangeReport/BWIBBU_ALL> | Schema: `Date`, `Code`, `Name`, `PEratio`, `DividendYield`, `PBratio`; live values not checked. |
| TPEx valuation | <https://www.tpex.org.tw/openapi/v1/tpex_mainboard_peratio_analysis> | Live: `Date`, `SecuritiesCompanyCode`, `CompanyName`, `PriceEarningRatio`, `DividendPerShare`, `YieldRatio`, `PriceBookRatio`. Sample `1240`: `Date=1151028`, PE `10.02`, dividend/share `0.50000000`, yield `0.93`, PB `1.59`. |
| TWSE companies | <https://openapi.twse.com.tw/v1/opendata/t187ap03_L> | Live, 1,098 rows: `出表日期`, `公司代號`, `公司名稱`, `公司簡稱`, `產業別`, `成立日期`, `上市日期`, `實收資本額`, `已發行普通股數或TDR原股發行股數`, `普通股每股面額`, `編制財務報表類型`, `網址`. |
| TPEx companies | <https://www.tpex.org.tw/openapi/v1/mopsfin_t187ap03_O> | Live: `Date`, `SecuritiesCompanyCode`, `CompanyName`, `CompanyAbbreviation`, `SecuritiesIndustryCode`, `DateOfIncorporation`, `DateOfListing`, `Paidin.Capital.NTDollars`, `IssueShares`, `ParValueOfCommonStock`, `PreparationOfFinancialReportType`, `WebAddress`. |

Master dates mix calendars: extract date is ROC `1151028`; establishment/listing dates are Gregorian (`19670218`, `20180808` for TPEx 1240; `19501229`, `19620209` for TWSE 1101). Industry codes are strings with leading zeros. `Symbol` in TPEx company master is the **English abbreviation**, not the stock code. Company master excludes many ETFs/warrants and is not a historical delisting/migration database. Yield fields are percentages, not fractions; missing/negative-earnings PE is not zero.

## 6. Monthly revenue and EPS/general financials

Revenue: [TWSE](https://openapi.twse.com.tw/v1/opendata/t187ap05_L), [TPEx](https://www.tpex.org.tw/openapi/v1/mopsfin_t187ap05_O). Both live; TWSE full response had 1,088 rows. **Both use these Chinese keys**, including TPEx:

```text
出表日期 資料年月 公司代號 公司名稱 產業別
營業收入-當月營收 營業收入-上月營收 營業收入-去年當月營收
營業收入-上月比較增減(%) 營業收入-去年同月增減(%)
累計營業收入-當月累計營收 累計營業收入-去年累計營收
累計營業收入-前期比較增減(%) 備註
```

Period `資料年月="11509"` means September 2026, not extraction date `出表日期="1151017"`. Sample TWSE `1101` monthly revenue `14321556`; TPEx `1240` monthly revenue `282821`. Percentage fields already contain percent values. Snapshot APIs expose no historical-period selector. The live [official MOPS September 2026 listed-company revenue report](https://mopsov.twse.com.tw/nas/t21/sii/t21sc03_115_9_0.html) explicitly labels amounts **千元 (thousands)**. Units are not declared in the JSON schema itself; keep the scale explicit rather than treating revenue values as individual currency units. TPEx amount scale and financial-statement currency/scale were not independently verified here.

EPS: [TWSE t187ap14_L](https://openapi.twse.com.tw/v1/opendata/t187ap14_L), [TPEx mopsfin_t187ap14_O](https://www.tpex.org.tw/openapi/v1/mopsfin_t187ap14_O), both live.

| Meaning | TWSE exact key | TPEx exact key |
|---|---|---|
| extract date / fiscal year / quarter | `出表日期` / `年度` / `季別` | `Date` / `Year` / `季別` |
| symbol / name / industry | `公司代號` / `公司名稱` / `產業別` | `SecuritiesCompanyCode` / `CompanyName` / `產業別` |
| EPS | `基本每股盈餘(元)` | `基本每股盈餘` |
| other financial fields | `普通股每股面額`, `營業收入`, `營業利益`, `營業外收入及支出`, `稅後淨利` | same Chinese keys |

Sample TWSE `1101`: extract `1151029`, year `115`, quarter `2`, EPS `0.38`. TPEx `1240`: extract `1151028`, year `115`, quarter `2`, EPS `2.85`. These are reporting-period snapshots, **not established TTM EPS or independently verified single-quarter EPS**. Preserve year/quarter and cumulative-period semantics; do not sum four cumulative reports.

General-company financial schemas (schema checked, live bodies not sampled):

- Income statement: [TWSE t187ap06_L_ci](https://openapi.twse.com.tw/v1/opendata/t187ap06_L_ci), [TPEx mopsfin_t187ap06_O_ci](https://www.tpex.org.tw/openapi/v1/mopsfin_t187ap06_O_ci). Identity/period keys: TWSE `出表日期,年度,季別,公司代號,公司名稱`; TPEx **`Date,Year,Season,SecuritiesCompanyCode,CompanyName`**. Shared exact metric keys: `營業收入`, `營業成本`, `營業毛利（毛損）`, `營業毛利（毛損）淨額`, `營業費用`, `營業利益（損失）`, `營業外收入及支出`, `稅前淨利（淨損）`, `所得稅費用（利益）`, `本期淨利（淨損）`, `淨利（淨損）歸屬於母公司業主`, `基本每股盈餘（元）`. The EPS key here uses **fullwidth** parentheses, unlike `t187ap14_L`.
- Balance sheet: [TWSE t187ap07_L_ci](https://openapi.twse.com.tw/v1/opendata/t187ap07_L_ci), [TPEx mopsfin_t187ap07_O_ci](https://www.tpex.org.tw/openapi/v1/mopsfin_t187ap07_O_ci). TPEx identity/period is **`Date,年度,季別,SecuritiesCompanyCode,CompanyName`**, not income-statement `Year,Season`. Shared metrics: `流動資產`, `非流動資產`, `資產總計`, `流動負債`, `非流動負債`, `負債總計`, `股本`, `資本公積`, `保留盈餘`, `庫藏股票`, `歸屬於母公司業主之權益合計`, `非控制權益`, `權益總計`, `每股參考淨值`.
- Official schemas also list sector-specific `bd` (securities/futures), `fh` (financial holding), `ins` (insurance), `mim` (mixed industries); TPEx additionally lists `basi` (banking). Do not assume the general-company layout covers all issuers. No complete historical financial/cash-flow API was verified in this bounded pass; consult MOPS filings rather than inventing a generic endpoint.

## 7. Holidays and corporate actions

**TWSE calendar:** <https://openapi.twse.com.tw/v1/holidaySchedule/holidaySchedule>, full JSON verified, 27 rows, keys `Name,Date,Weekday,Description`, ROC date. Example `{Name:"中華民國開國紀念日",Date:"1150101",Weekday:"四",Description:"依規定放假1日。"}`. **It is not just a holiday set**: rows include new-year opening `1150102`, last pre-Lunar-New-Year trading day `1150211`, post-holiday reopening `1150223`, and settlement-only dates. Classify by event semantics, not membership. No machine boolean or year parameter is declared. Scheduled calendar also cannot prove absence of emergency closures.

**TPEx calendar:** [official year selector](https://www.tpex.org.tw/zh-tw/announce/market/holiday.html). Its source calls `/www/zh-tw/bulletin/tradingDate`; the page converts `data.html` into a table. A GET with `date=2026/01/01&response=json` did not yield parseable JSON. Thus its exact request contract is **not live-validated**, and it is not a verified plain-array holiday OpenAPI. Use the official page/manual calendar fallback; do not invent `tpex_holiday`.

**TWSE ex-right/dividend announcements:** <https://openapi.twse.com.tw/v1/exchangeReport/TWT48U_ALL>, full JSON verified (22 rows). Exact keys `Date,Code,Name,Exdividend,StockDividendRatio,SubscriptionRatio,SubscriptionPricePerShare,CashDividend,SharesOffered,SharesEmpOwner,SharesholderOwner,StockHoldingRatio`. `Date` is the ROC **event date**, not snapshot date. Sample `00400A` event `1151106`, `Exdividend="息"`, cash dividend blank; [official report](https://www.twse.com.tw/exchangeReport/TWT48U?response=html) says this amount is awaiting announcement. Blank is not zero.

**TWSE calculated references:** <https://www.twse.com.tw/exchangeReport/TWT49U?response=json> — live JSON verified. Root `{stat,title,fields,data,extraNotes,notes,formula,strDate,endDate}`; observed `stat="OK"`, `strDate=endDate="20261029"`. Columns 0–10: `資料日期,股票代號,股票名稱,除權息前收盤價,除權息參考價,權值+息值,權/息,漲停價格,跌停價格,開盤競價基準,減除股利參考價`; columns 11–14: `詳細資料`, `最近一次申報資料 季別/日期`, `最近一次申報每股 (單位)淨值`, `最近一次申報每股 (單位)盈餘`. Sample indices 0–6: `["115年10月29日","2762","世界健身-KY","80.00","78.19","1.807197","息"]`; row date is a **Chinese ROC date**, while root dates are Gregorian compact. The [current official page](https://www.twse.com.tw/zh/announcement/ex-right/twt49u.html) declares backend action `/exRight/TWT49U` and form parameters `startDate,endDate`; date-range requests were not live-tested. Its [report notes](https://www.twse.com.tw/exchangeReport/TWT49U?lang=en&response=html) state data since 2003-05-05 and explicitly exclude some combined dividend/capital-reduction cases. Not a complete adjustment-factor feed.

**TPEx announcements:** <https://www.tpex.org.tw/openapi/v1/tpex_exright_prepost>, live first object. Exact keys: `ExRrightsExDividendDate` (ROC event date; spelling intentional), `SecuritiesCompanyCode`, `CompanyName`, `ExRrightsExDividend`, `StockDividendRatio`, `SubscriptionRatioToNewSharesIssued`, `SubscriptionPricePerShare`, `CashDividend`, `AllocatedForPublicUnderwriting`, `SubscribedByEmployees`, `SubscribedByExistingShareholders`, `SubscribedProRataInThousandShares`.

**TPEx calculated references:** <https://www.tpex.org.tw/openapi/v1/tpex_exright_daily>, live first object. Exact keys:

```text
Date SecuritiesCompanyCode CompanyName
ClosePriceBeforeExRightsDiviend ExRightsDiviendQuote StockDividend CashDividend
StockDividendPlusCashDividend ExRightsDiviend LimitUp LimitDown
OpeningReferencePrice DividendDeductedQuote CashDivdend StockDivdendThousandShares
CashCapitalIncreaseShares SubscriptionPricePerShare AllocatedForPublicUnderwriting
SubscribedByEmployees SubscribedByExistingShareholders SubscribedProRataThousandShares
```

Spelling errors are contractual keys. `CashDividend` is 息值, while `CashDivdend` is 現金股利; `StockDivdendThousandShares` is 每仟股無償配股. Sample `00726B` on `1151028`: prior close `32.22`, ex-reference `31.76`, cash-dividend value `0.460000`. These snapshot endpoints do not establish complete event history. Separate capital reductions, face-value changes, ETF splits/reverse splits and listing changes remain necessary for a trustworthy adjusted series; no complete free normalized endpoint for all those events was verified. Preserve raw OHLCV and label any derived adjusted series only after complete event coverage is validated.

## 8. Normalization and failure behavior

1. Treat dates by endpoint/field, not one global parser: ROC `YYYMMDD` → year +1911; ROC `YYYMM` for revenue; ROC fiscal `YYY`; Gregorian `YYYYMMDD` for website root dates and TWSE BWIBBU_d; Gregorian listing/establishment dates. Store normalized `YYYY-MM-DD` plus original source value. Trading timezone is Asia/Taipei.
2. Keep identifiers strings; trim names and header display whitespace. Numeric cells may have commas, signs, blanks, `--`, or non-numeric corporate-action markers. Missing price is null, never 0; do not synthesize an OHLC bar for suspended/no-trade days. Retain raw exceptional cells for audit. Treat repeated table headers with positional group mappings.
3. Store OHLCV volume and institutional flow in shares; margin balances in stated trading units/lots. Never label institutional net **shares** as money. Price/value currency must follow the instrument; foreign-currency ETFs exist.
4. Verify HTTP success, JSON content, `stat` (TWSE `OK`, TPEx `ok`), response date, required fields, row widths, and completeness before committing a day. HTML, redirects, partial downloads, missing rows, or empty results do not establish a market holiday. No-trade institutional/margin rows must not automatically be filled with zeros.
5. Poll snapshots sparingly, cache centrally, record `fetched_at`, `source_date`, report scope and provenance, and retry ordinary transient errors with backoff. Stop on blocking/access-denied responses. Do not rotate IPs, replay challenge tokens, or switch hosts to evade restrictions. Test the actual deployment's egress independently; this workstation's success does not prove browser/Worker access.
6. Historical fallback: retain daily snapshots going forward; use official report-page CSV export/manual import when available; expose gaps or a shorter lookback when retrieval fails. For guaranteed/deeper bulk history, evaluate the official [TWSE data shop](https://eshop.twse.com.tw/) and [TPEx data service](https://www.tpex.org.tw/zh-tw/service/data/overview.html) as **potential paid/licensed alternatives**, not free APIs. The [TWSE institutional product](https://eshop.twse.com.tw/zh/product/detail/c4c87ac184e44896a05fcab5a9d544ec) is a separate paid archive, not evidence that the public T86 backend can supply that history.

## Verification limits

Actual requests were bounded samples, not a load test or exhaustive history crawl. Verified older OHLCV coverage: TPEx 2025-01-02; TWSE 2026-10-27. TWSE 2025-01-02 MI_INDEX, 2026-09 STOCK_DAY, and 2026-10-27 T86 returned HTTP 307. No cause/cutoff was established. Financial-statement/TPEx revenue amount scale, all historical layouts, financial historical access, complete corporate-action history, CORS and production runtime connectivity remain unverified. All official source links are provided alongside the relevant mappings; no third-party endpoint or invented availability is relied upon.
