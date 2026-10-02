# market-data

Append-only public market data recorded by the Monitor service (src/marketdata/, collectors C1 + C5-lite in research/IDEA_BACKLOG.md).
Written by a bot; do not edit. Layout: `<source>/<YYYY-MM-DD>/<HH>.ndjson.gz` (UTC hour, one JSON record per line, gzip).
Record field `v` = layout version (1). Times are ms epoch. Decimals are HL strings, verbatim.

| source | cadence | record |
|---|---|---|
| hl-ctx | 1 min | `{v,t,dex,rows}`, one line per dex ("" = main, "xyz" = HIP-3). row = [coin, funding, openInterest, markPx, oraclePx, midPx, premium, dayNtlVlm (rounded to whole USD), impactBidPx, impactAskPx] |
| hl-book | 1 min | `{v,at,c,t,b,a}` per coin: at = tick time, t = HL book time, b/a = top levels [px, sz, nOrders], best first. Universe: BTC/ETH/SOL + top perps + top xyz by 24h volume, refreshed hourly |
| hl-funding | 5 min | `{v,t,rows}` HL `predictedFundings` (HL, Binance, Bybit): row = [coin, venue, fundingRate, nextFundingTime, intervalHours] |
| hl-hip3 | 5 min | `{v,t,dex,rows}`, one line per HIP-3 dex other than xyz (listed by perpDexs), same row layout as hl-ctx. Backlog #126 |
| hl-dexcfg | 1 h | `{v,t,dexes}` HL `perpDexs`: per dex {dex, deployer, oiCap, fundingMultiplier, fundingInterestRate, fundingClamp}, each [coin, value][] verbatim |
| hl-oicap | 1 min | `{v,t,dexes}` HL `perpsAtOpenInterestCap`: dexes = [dex ("" = main), coins at OI cap (sorted)][]; main + xyz every minute, other HIP-3 dexes every 5 min (hl-hip3 minute). A dex missing from a line = not polled / failed. Delisted coins are kept verbatim (HL lists them as capped). Backlog #138 |
| hl-outcome | 5 min | `{v,t,rows}` HIP-4 outcome coins from `spotMetaAndAssetCtxs` (both sides): row = [coin "#<outcome*10+side>", markPx, midPx, prevDayPx, dayNtlVlm, dayBaseVlm, circulatingSupply]. Backlog #131 |
| hl-outcome-meta | 5 min, on change (+ first of each hour) | `{v,t,meta}` HL `outcomeMeta` verbatim (outcomes, questions incl. settledNamedOutcomes, deployers). An outcome missing from a later line = settled/removed |
| hl-outcome-book | 5 min | same as hl-book, side-0 coin only (side 1 is the exact mirror: px -> 1-px, bid <-> ask). Recurring price binaries + top outcomes by 24h volume |
| hl-outcome-trade | 5 min | `{v,t,c,gap,rows}` new side-0 prints (`recentTrades`, deduped by tid) of outcomes whose 24h volume moved: row = [time, side, px, sz, tid]; gap = prints may be missing before the first row |
| hl-liq | 1 min | `{v,t,n,rows,truncated}` HL `liquidatable` (undocumented): positions eligible for liquidation, rows verbatim |

Read: `zcat hl-book/2026-10-01/*.ndjson.gz | jq -c 'select(.c=="BTC")'`. Clone only this branch: `git clone --single-branch -b market-data ...`.
