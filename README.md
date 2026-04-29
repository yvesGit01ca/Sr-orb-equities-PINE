# Sr-orb-equities-PINE

//@version=5
// =============================================================================
// SR Opening Range Breakout Strategy — US Equities
// Port of the MT5 SR Level Trade logic, adapted for individual US stocks.
//
// Logic:
//   1. On each new session, capture the first N bars after the cash open.
//   2. Define rLine = highest high, sLine = lowest low of that window.
//   3. If range width <= maxRange (ATR-based), arm the setup.
//   4. Entry modes:
//        - STOP:   place BuyStop at rLine + buffer, SellStop at sLine - buffer
//        - MARKET: enter at market on the first bar that closes outside the box
//        - BOTH:   user toggle
//   5. SL/TP are symmetric multiples of the range width.
//   6. Optional: session filter, gap skip, trailing stop, equity pause.
// =============================================================================

strategy(
     title               = "SR Opening Range Breakout — Equities",
     shorttitle          = "SR ORB EQ",
     overlay             = true,
     initial_capital     = 10000,
     default_qty_type    = strategy.percent_of_equity,
     default_qty_value   = 10,
     pyramiding          = 0,
     calc_on_every_tick  = false,
     process_orders_on_close = false,
     commission_type     = strategy.commission.percent,
     commission_value    = 0.03,
     slippage            = 2)

// -----------------------------------------------------------------------------
// INPUTS
// -----------------------------------------------------------------------------
grpRange   = "Opening Range"
grpEntry   = "Entry & Exits"
grpRisk    = "Risk & Sizing"
grpFilter  = "Filters"
grpTS      = "Trailing Stop"
grpPause   = "Equity Pause"
grpVis     = "Visuals"

// --- Opening range --------------------------------------------------------
orMinutes       = input.int(30,  "Opening Range Duration (minutes)", minval = 5, maxval = 240, group = grpRange,
                             tooltip = "How many minutes from the cash open to measure. Classic ORB uses 30 or 60 minutes.")
useATRMaxRange  = input.bool(true, "Limit range width using ATR", group = grpRange)
atrLen          = input.int(14,  "ATR Length (daily)",           minval = 5,  group = grpRange)
atrMaxMult      = input.float(0.5, "Max range width = ATR ×",    minval = 0.1, step = 0.1, group = grpRange,
                             tooltip = "Skip the day if the opening range is wider than this multiple of daily ATR. 0.5 = half of 14-day ATR.")

// --- Entry & exits --------------------------------------------------------
entryMode = input.string("Stop", "Entry Mode", options = ["Stop", "Market", "Both"], group = grpEntry,
                          tooltip = "Stop = classic pending orders at the range edges. Market = enter at close of the bar that breaks out. Both = arm both methods simultaneously.")
bufferPts = input.float(0.0, "Buffer (price units)", minval = 0.0, step = 0.01, group = grpEntry,
                          tooltip = "Extra distance beyond rLine/sLine before triggering, in the symbol's price units (dollars for US stocks).")
slMult    = input.float(1.0, "SL Multiplier × range",   minval = 0.1, step = 0.1, group = grpEntry)
tpMult    = input.float(1.5, "TP Multiplier × range",   minval = 0.1, step = 0.1, group = grpEntry)
expireHrs = input.float(6.0, "Expire pending after (hours)", minval = 0.5, step = 0.5, group = grpEntry)

// --- Risk & sizing --------------------------------------------------------
useRiskPct = input.bool(true,  "Size by risk % of equity", group = grpRisk)
riskPct    = input.float(1.0,  "Risk % per trade", minval = 0.1, maxval = 5.0, step = 0.1, group = grpRisk)
fixedQty   = input.int  (100,  "Fixed shares (if risk % disabled)", minval = 1, group = grpRisk)

// --- Filters --------------------------------------------------------------
useSession   = input.bool(true,  "Trade only during regular cash hours", group = grpFilter)
sessionTime  = input.session("0930-1600", "Session (exchange time)", group = grpFilter)
cutoffTime   = input.session("0930-1500", "No new entries after (closes early)", group = grpFilter,
                               tooltip = "New entries are blocked outside this window to avoid end-of-day breakouts with no room to extend.")
useGapSkip   = input.bool(true, "Skip days with large overnight gap", group = grpFilter)
maxGapPct    = input.float(2.0, "Max overnight gap (%)", minval = 0.1, step = 0.1, group = grpFilter,
                             tooltip = "Skip the day if |today's open - prior close| / prior close > this percent.")

// --- Trailing stop --------------------------------------------------------
useTS     = input.bool(false, "Enable Trailing Stop", group = grpTS)
tsStart   = input.float(1.0,  "TS Start × range",    minval = 0.1, step = 0.1, group = grpTS,
                          tooltip = "Activate trailing once price is this many range-widths in profit.")
tsDist    = input.float(1.0,  "TS Distance × range", minval = 0.1, step = 0.1, group = grpTS)

// --- Equity pause ---------------------------------------------------------
usePause       = input.bool(false, "Enable equity-based pause", group = grpPause)
ddPausePct     = input.float(5.0,  "Drawdown pause %",  minval = 0.5, step = 0.5, group = grpPause)
profitPausePct = input.float(10.0, "Profit target pause %", minval = 0.5, step = 0.5, group = grpPause)
pauseHours     = input.float(24,   "Pause duration (hours)", minval = 1, step = 1, group = grpPause)

// --- Visuals --------------------------------------------------------------
showBox    = input.bool(true, "Show range box",   group = grpVis)
showLines  = input.bool(true, "Show SR lines",    group = grpVis)
showEntries= input.bool(true, "Show entry labels", group = grpVis)
boxColor   = input.color(color.new(color.red, 80),   "Box color",     group = grpVis)
rColor     = input.color(color.new(color.red, 0),    "Resistance",    group = grpVis)
sColor     = input.color(color.new(color.green, 0),  "Support",       group = grpVis)

// -----------------------------------------------------------------------------
// SESSION / TIME LOGIC
// -----------------------------------------------------------------------------
inSession   = not useSession or not na(time(timeframe.period, sessionTime))
inEntryWin  = not na(time(timeframe.period, cutoffTime))
isNewDay    = ta.change(time("D")) != 0

// Opening-range window: first orMinutes of the session.
// We detect the first bar of the session using ta.change on session time.
sessionStart = ta.change(time(timeframe.period, sessionTime)) != 0 and inSession
var int sessionStartTime = na
if sessionStart
    sessionStartTime := time
minutesFromOpen = (time - nz(sessionStartTime)) / 60000
inORWindow = inSession and minutesFromOpen < orMinutes
orWindowJustEnded = inSession and minutesFromOpen >= orMinutes and minutesFromOpen - (time - time[1]) / 60000 < orMinutes

// -----------------------------------------------------------------------------
// STATE
// -----------------------------------------------------------------------------
var float rLine          = na
var float sLine          = na
var float rangeWidth     = na
var bool  setupValid     = false
var bool  tradedToday    = false
var int   armedTime      = na
var float dailyATR       = na
var float priorClose     = na
var float gapPct         = na
var bool  gapSkipToday   = false

// Trailing stop state (one active position at a time given pyramiding = 0)
var float tsStopLong     = na
var float tsStopShort    = na

// Equity pause state
var bool  paused         = false
var int   pauseUntil     = na
var float pauseBaseline  = na

// -----------------------------------------------------------------------------
// DAILY RESETS
// -----------------------------------------------------------------------------
if isNewDay
    rLine         := na
    sLine         := na
    rangeWidth    := na
    setupValid    := false
    tradedToday   := false
    armedTime     := na
    tsStopLong    := na
    tsStopShort   := na
    sessionStartTime := na

    // Capture prior close and today's open for gap calc (request.security
    // on daily data — use lookahead_off to prevent repainting).
    priorCloseD = request.security(syminfo.tickerid, "D", close[1], lookahead = barmerge.lookahead_off)
    todayOpenD  = request.security(syminfo.tickerid, "D", open,     lookahead = barmerge.lookahead_off)
    priorClose := priorCloseD
    gapPct := not na(priorCloseD) and priorCloseD != 0 ? math.abs(todayOpenD - priorCloseD) / priorCloseD * 100 : 0.0
    gapSkipToday := useGapSkip and gapPct > maxGapPct

// Daily ATR (refreshed from daily series, no lookahead)
dailyATR := request.security(syminfo.tickerid, "D", ta.atr(atrLen), lookahead = barmerge.lookahead_off)

// -----------------------------------------------------------------------------
// BUILD OPENING RANGE
// -----------------------------------------------------------------------------
// Track the running high/low during the OR window using bar-by-bar updates.
var float runHi = na
var float runLo = na

if inSession and sessionStart
    runHi := high
    runLo := low
else if inORWindow
    runHi := na(runHi) ? high : math.max(runHi, high)
    runLo := na(runLo) ? low  : math.min(runLo, low)

// When the OR window closes, finalize the setup.
orWindowClosing = inORWindow and not inORWindow[-1]  // would use barstate.isconfirmed & future bars — avoid
// Use an alternative: the first bar where inORWindow is false but session has started.
orFinalize = inSession and not inORWindow and na(rLine) and not na(runHi) and not na(runLo) and not tradedToday
if orFinalize and not gapSkipToday
    rLine      := runHi
    sLine      := runLo
    rangeWidth := rLine - sLine
    maxAllowed = useATRMaxRange and not na(dailyATR) ? dailyATR * atrMaxMult : rangeWidth + 1
    setupValid := rangeWidth > 0 and rangeWidth <= maxAllowed
    armedTime  := time

// Expire the setup if pending orders haven't triggered in expireHrs.
setupExpired = setupValid and not na(armedTime) and (time - armedTime) > expireHrs * 3600000
if setupExpired
    setupValid := false
    strategy.cancel("BuyStop")
    strategy.cancel("SellStop")

// -----------------------------------------------------------------------------
// EQUITY PAUSE
// -----------------------------------------------------------------------------
if usePause
    // Initialize baseline once
    if na(pauseBaseline)
        pauseBaseline := strategy.equity
    // Resume if pause expired — re-baseline
    if paused and not na(pauseUntil) and time >= pauseUntil
        paused        := false
        pauseUntil    := na
        pauseBaseline := strategy.equity
    // Check triggers
    if not paused
        ddHit   = (pauseBaseline - strategy.equity) / pauseBaseline * 100 > ddPausePct
        profHit = (strategy.equity - pauseBaseline) / pauseBaseline * 100 > profitPausePct
        if ddHit or profHit
            paused     := true
            pauseUntil := time + int(pauseHours * 3600000)
            strategy.close_all(comment = "Pause")
            strategy.cancel_all()

tradingAllowed = not paused and not gapSkipToday and setupValid and inSession and inEntryWin and not tradedToday

// -----------------------------------------------------------------------------
// POSITION SIZING
// -----------------------------------------------------------------------------
calcQty(float entryPrice, float stopPrice) =>
    float qty = fixedQty
    if useRiskPct and not na(entryPrice) and not na(stopPrice) and entryPrice != stopPrice
        riskMoney = strategy.equity * riskPct / 100.0
        perShare  = math.abs(entryPrice - stopPrice)
        qty := perShare > 0 ? math.floor(riskMoney / perShare) : fixedQty
    math.max(1, qty)

// -----------------------------------------------------------------------------
// ENTRIES
// -----------------------------------------------------------------------------
useStop   = entryMode == "Stop"   or entryMode == "Both"
useMarket = entryMode == "Market" or entryMode == "Both"

// --- STOP orders (submit once when setup arms) ----------------------------
if tradingAllowed and useStop and strategy.position_size == 0
    buyEntry  = rLine + bufferPts
    sellEntry = sLine - bufferPts
    buySL     = rLine - rangeWidth * slMult
    buyTP     = rLine + rangeWidth * tpMult
    sellSL    = sLine + rangeWidth * slMult
    sellTP    = sLine - rangeWidth * tpMult

    qtyBuy  = calcQty(buyEntry,  buySL)
    qtySell = calcQty(sellEntry, sellSL)

    strategy.entry("BuyStop",  strategy.long,  qty = qtyBuy,  stop = buyEntry,  comment = "Stop L")
    strategy.exit ("XL-Stop",  from_entry = "BuyStop",  stop = buySL,  limit = buyTP)

    strategy.entry("SellStop", strategy.short, qty = qtySell, stop = sellEntry, comment = "Stop S")
    strategy.exit ("XS-Stop",  from_entry = "SellStop", stop = sellSL, limit = sellTP)

// --- MARKET breakout entries (evaluated each bar) -------------------------
brokeUp   = useMarket and tradingAllowed and strategy.position_size == 0 and close > rLine + bufferPts
brokeDown = useMarket and tradingAllowed and strategy.position_size == 0 and close < sLine - bufferPts

if brokeUp
    buySL = rLine - rangeWidth * slMult
    buyTP = rLine + rangeWidth * tpMult
    strategy.entry("BuyMkt", strategy.long, qty = calcQty(close, buySL), comment = "Mkt L")
    strategy.exit ("XL-Mkt", from_entry = "BuyMkt", stop = buySL, limit = buyTP)
    tradedToday := true
    if useStop
        strategy.cancel("SellStop")

if brokeDown
    sellSL = sLine + rangeWidth * slMult
    sellTP = sLine - rangeWidth * tpMult
    strategy.entry("SellMkt", strategy.short, qty = calcQty(close, sellSL), comment = "Mkt S")
    strategy.exit ("XS-Mkt",  from_entry = "SellMkt", stop = sellSL, limit = sellTP)
    tradedToday := true
    if useStop
        strategy.cancel("BuyStop")

// Mark tradedToday once a stop order fills (detect transition flat -> in position)
if strategy.position_size != 0 and strategy.position_size[1] == 0
    tradedToday := true
    // Cancel the opposite pending
    if strategy.position_size > 0
        strategy.cancel("SellStop")
    else
        strategy.cancel("BuyStop")

// -----------------------------------------------------------------------------
// TRAILING STOP
// -----------------------------------------------------------------------------
if useTS and not na(rangeWidth) and rangeWidth > 0
    // Long side
    if strategy.position_size > 0
        avg = strategy.position_avg_price
        activateAt = avg + rangeWidth * tsStart
        if high >= activateAt
            candidate = close - rangeWidth * tsDist
            tsStopLong := na(tsStopLong) ? candidate : math.max(tsStopLong, candidate)
        if not na(tsStopLong)
            strategy.exit("TSL", stop = tsStopLong)
    else
        tsStopLong := na

    // Short side
    if strategy.position_size < 0
        avg = strategy.position_avg_price
        activateAt = avg - rangeWidth * tsStart
        if low <= activateAt
            candidate = close + rangeWidth * tsDist
            tsStopShort := na(tsStopShort) ? candidate : math.min(tsStopShort, candidate)
        if not na(tsStopShort)
            strategy.exit("TSS", stop = tsStopShort)
    else
        tsStopShort := na

// -----------------------------------------------------------------------------
// END-OF-SESSION FLATTENING (optional safety: close everything at cutoff)
// -----------------------------------------------------------------------------
endOfSession = useSession and na(time(timeframe.period, sessionTime)) and not na(time(timeframe.period, sessionTime)[1])
if endOfSession
    strategy.close_all(comment = "EOD")
    strategy.cancel_all()

// -----------------------------------------------------------------------------
// VISUALS
// -----------------------------------------------------------------------------
plotRLine = showLines and setupValid ? rLine : na
plotSLine = showLines and setupValid ? sLine : na
plot(plotRLine, title = "R Line", color = rColor, style = plot.style_linebr, linewidth = 2)
plot(plotSLine, title = "S Line", color = sColor, style = plot.style_linebr, linewidth = 2)

// Draw the opening-range box once per day
var box orBox = na
if orFinalize and setupValid and showBox
    if not na(orBox)
        box.delete(orBox)
    orBox := box.new(left = nz(sessionStartTime), top = rLine, right = time, bottom = sLine,
                     border_color = color.new(color.red, 40), bgcolor = boxColor, xloc = xloc.bar_time, extend = extend.right)
if not na(orBox) and setupValid
    box.set_right(orBox, time)

// Entry markers
plotshape(showEntries and strategy.position_size > 0 and strategy.position_size[1] == 0,
          title = "Long Entry",  style = shape.triangleup,   location = location.belowbar,
          color = color.new(color.green, 0), size = size.small)
plotshape(showEntries and strategy.position_size < 0 and strategy.position_size[1] == 0,
          title = "Short Entry", style = shape.triangledown, location = location.abovebar,
          color = color.new(color.red, 0),   size = size.small)

// Status table
var table statusTbl = table.new(position = position.top_right, columns = 2, rows = 7,
                                 bgcolor = color.new(color.black, 80), border_width = 1)
if barstate.islast
    table.cell(statusTbl, 0, 0, "Status",         text_color = color.white, bgcolor = color.new(color.blue, 60))
    table.cell(statusTbl, 1, 0, paused ? "PAUSED" : gapSkipToday ? "GAP-SKIP" : setupValid ? "ARMED" : "WAITING",
               text_color = paused ? color.red : setupValid ? color.lime : color.yellow)
    table.cell(statusTbl, 0, 1, "R",     text_color = color.white)
    table.cell(statusTbl, 1, 1, na(rLine) ? "—" : str.tostring(rLine, format.mintick), text_color = rColor)
    table.cell(statusTbl, 0, 2, "S",     text_color = color.white)
    table.cell(statusTbl, 1, 2, na(sLine) ? "—" : str.tostring(sLine, format.mintick), text_color = sColor)
    table.cell(statusTbl, 0, 3, "Range", text_color = color.white)
    table.cell(statusTbl, 1, 3, na(rangeWidth) ? "—" : str.tostring(rangeWidth, format.mintick), text_color = color.white)
    table.cell(statusTbl, 0, 4, "ATR(D)", text_color = color.white)
    table.cell(statusTbl, 1, 4, na(dailyATR) ? "—" : str.tostring(dailyATR, format.mintick), text_color = color.white)
    table.cell(statusTbl, 0, 5, "Gap %", text_color = color.white)
    table.cell(statusTbl, 1, 5, str.tostring(nz(gapPct), "#.##"), text_color = gapSkipToday ? color.red : color.white)
    table.cell(statusTbl, 0, 6, "Equity", text_color = color.white)
    table.cell(statusTbl, 1, 6, str.tostring(strategy.equity, "#.##"), text_color = color.white)

// -----------------------------------------------------------------------------
// ALERTS
// -----------------------------------------------------------------------------
alertcondition(brokeUp,   title = "Long Breakout",  message = "{{ticker}} broke above opening range high at {{close}}")
alertcondition(brokeDown, title = "Short Breakout", message = "{{ticker}} broke below opening range low at {{close}}")
