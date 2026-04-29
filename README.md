//@version=6
// =============================================================================
// SR Opening Range Breakout Strategy — US Equities (Pine Script v6)
// =============================================================================
// Port of the MT5 SR Level Trade logic, adapted for individual US stocks.
//
// Logic:
//   1. On each new session, capture the first N minutes after the cash open.
//   2. rLine = highest high, sLine = lowest low of that window.
//   3. If range width <= maxRange (ATR-based), arm the setup.
//   4. Entry modes:
//        - Stop:   pending stops at rLine + buffer / sLine - buffer
//        - Market: enter on the close of the bar that breaks the range
//        - Both:   arm both, whichever fires first wins
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
orMinutes      = input.int(30,  "Opening Range Duration (minutes)", minval = 5, maxval = 240, group = grpRange)
useATRMaxRange = input.bool(true, "Limit range width using ATR", group = grpRange)
atrLen         = input.int(14, "ATR Length (daily)", minval = 5, group = grpRange)
atrMaxMult     = input.float(0.5, "Max range width = ATR ×", minval = 0.1, step = 0.1, group = grpRange,
                              tooltip = "Skip the day if the opening range is wider than this multiple of daily ATR.")

// --- Entry & exits --------------------------------------------------------
entryMode = input.string("Stop", "Entry Mode", options = ["Stop", "Market", "Both"], group = grpEntry,
                          tooltip = "Stop = pending orders at the range edges. Market = enter at close of breakout bar. Both = arm both.")
bufferPts = input.float(0.0, "Buffer (price units)", minval = 0.0, step = 0.01, group = grpEntry,
                          tooltip = "Extra distance beyond rLine/sLine before triggering, in dollars.")
slMult    = input.float(1.0, "SL Multiplier × range", minval = 0.1, step = 0.1, group = grpEntry)
tpMult    = input.float(1.5, "TP Multiplier × range", minval = 0.1, step = 0.1, group = grpEntry)
expireHrs = input.float(6.0, "Expire pending after (hours)", minval = 0.5, step = 0.5, group = grpEntry)

// --- Risk & sizing --------------------------------------------------------
useRiskPct = input.bool(true, "Size by risk % of equity", group = grpRisk)
riskPct    = input.float(1.0, "Risk % per trade", minval = 0.1, maxval = 5.0, step = 0.1, group = grpRisk)
fixedQty   = input.int  (100, "Fixed shares (if risk % disabled)", minval = 1, group = grpRisk)

// --- Filters --------------------------------------------------------------
useSession  = input.bool(true, "Trade only during regular cash hours", group = grpFilter)
sessionTime = input.session("0930-1600", "Session (exchange time)", group = grpFilter)
cutoffTime  = input.session("0930-1500", "No new entries after", group = grpFilter,
                              tooltip = "Block new entries outside this window so EOD breakouts don't trigger.")
useGapSkip  = input.bool(true, "Skip days with large overnight gap", group = grpFilter)
maxGapPct   = input.float(2.0, "Max overnight gap (%)", minval = 0.1, step = 0.1, group = grpFilter)

// --- Trailing stop --------------------------------------------------------
useTS   = input.bool(false, "Enable Trailing Stop", group = grpTS)
tsStart = input.float(1.0, "TS Start × range",    minval = 0.1, step = 0.1, group = grpTS)
tsDist  = input.float(1.0, "TS Distance × range", minval = 0.1, step = 0.1, group = grpTS)

// --- Equity pause ---------------------------------------------------------
usePause       = input.bool(false, "Enable equity-based pause", group = grpPause)
ddPausePct     = input.float(5.0,  "Drawdown pause %", minval = 0.5, step = 0.5, group = grpPause)
profitPausePct = input.float(10.0, "Profit target pause %", minval = 0.5, step = 0.5, group = grpPause)
pauseHours     = input.float(24,   "Pause duration (hours)", minval = 1, step = 1, group = grpPause)

// --- Visuals --------------------------------------------------------------
showBox     = input.bool(true, "Show range box", group = grpVis)
showLines   = input.bool(true, "Show SR lines",  group = grpVis)
showEntries = input.bool(true, "Show entry labels", group = grpVis)
boxColor    = input.color(color.new(color.red, 80), "Box color", group = grpVis)
rColor      = input.color(color.new(color.red, 0), "Resistance", group = grpVis)
sColor      = input.color(color.new(color.green, 0), "Support",  group = grpVis)

// -----------------------------------------------------------------------------
// DAILY DATA (request.security with no lookahead, references precomputed series)
// -----------------------------------------------------------------------------
// Build the series we need on the daily timeframe BEFORE passing to request.security.
// In Pine v6 the cleanest pattern is: compute on the chart, then call security with
// a simple expression. For prior close we use close[1] inside the expression.
priorCloseDaily = request.security(syminfo.tickerid, "D", close[1], lookahead = barmerge.lookahead_off)
todayOpenDaily  = request.security(syminfo.tickerid, "D", open,     lookahead = barmerge.lookahead_off)
dailyATR        = request.security(syminfo.tickerid, "D", ta.atr(atrLen), lookahead = barmerge.lookahead_off)

// -----------------------------------------------------------------------------
// SESSION / TIME LOGIC
// -----------------------------------------------------------------------------
inSession  = not useSession or not na(time(timeframe.period, sessionTime))
inEntryWin = not na(time(timeframe.period, cutoffTime))
isNewDay   = ta.change(time("D")) != 0

// First bar of the session
sessionStart = ta.change(time(timeframe.period, sessionTime)) != 0 and inSession
var int sessionStartTime = na
if sessionStart
    sessionStartTime := time

// How many minutes since session start
minutesFromOpen = (time - nz(sessionStartTime)) / 60000
inORWindow      = inSession and minutesFromOpen < orMinutes

// -----------------------------------------------------------------------------
// STATE
// -----------------------------------------------------------------------------
var float rLine        = na
var float sLine        = na
var float rangeWidth   = na
var bool  setupValid   = false
var bool  tradedToday  = false
var int   armedTime    = na
var float gapPct       = na
var bool  gapSkipToday = false
var float runHi        = na
var float runLo        = na

// Trailing stop state
var float tsStopLong  = na
var float tsStopShort = na

// Equity pause state
var bool  paused        = false
var int   pauseUntil    = na
var float pauseBaseline = na

// -----------------------------------------------------------------------------
// DAILY RESETS
// -----------------------------------------------------------------------------
if isNewDay
    rLine            := na
    sLine            := na
    rangeWidth       := na
    setupValid       := false
    tradedToday      := false
    armedTime        := na
    tsStopLong       := na
    tsStopShort      := na
    sessionStartTime := na
    runHi            := na
    runLo            := na

    // Gap calculation from precomputed daily series
    gapPct := not na(priorCloseDaily) and priorCloseDaily != 0 ? math.abs(todayOpenDaily - priorCloseDaily) / priorCloseDaily * 100 : 0.0
    gapSkipToday := useGapSkip and gapPct > maxGapPct

// -----------------------------------------------------------------------------
// BUILD OPENING RANGE
// -----------------------------------------------------------------------------
if inSession and sessionStart
    runHi := high
    runLo := low
else if inORWindow
    runHi := na(runHi) ? high : math.max(runHi, high)
    runLo := na(runLo) ? low  : math.min(runLo, low)

// First bar after the OR window closes -> finalize the setup
orFinalize = inSession and not inORWindow and na(rLine) and not na(runHi) and not na(runLo) and not tradedToday
if orFinalize and not gapSkipToday
    rLine      := runHi
    sLine      := runLo
    rangeWidth := rLine - sLine
    float maxAllowed = useATRMaxRange and not na(dailyATR) ? dailyATR * atrMaxMult : rangeWidth + 1
    setupValid := rangeWidth > 0 and rangeWidth <= maxAllowed
    armedTime  := time

// Expire setup if pending orders haven't fired in expireHrs
setupExpired = setupValid and not na(armedTime) and (time - armedTime) > expireHrs * 3600000
if setupExpired
    setupValid := false
    strategy.cancel("BuyStop")
    strategy.cancel("SellStop")

// -----------------------------------------------------------------------------
// EQUITY PAUSE
// -----------------------------------------------------------------------------
if usePause
    if na(pauseBaseline)
        pauseBaseline := strategy.equity
    if paused and not na(pauseUntil) and time >= pauseUntil
        paused        := false
        pauseUntil    := na
        pauseBaseline := strategy.equity
    if not paused
        bool ddHit   = (pauseBaseline - strategy.equity) / pauseBaseline * 100 > ddPausePct
        bool profHit = (strategy.equity - pauseBaseline) / pauseBaseline * 100 > profitPausePct
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
        float riskMoney = strategy.equity * riskPct / 100.0
        float perShare  = math.abs(entryPrice - stopPrice)
        qty := perShare > 0 ? math.floor(riskMoney / perShare) : fixedQty
    math.max(1, qty)

// -----------------------------------------------------------------------------
// ENTRIES
// -----------------------------------------------------------------------------
useStop   = entryMode == "Stop"   or entryMode == "Both"
useMarket = entryMode == "Market" or entryMode == "Both"

// Stop orders — submit once when setup arms
if tradingAllowed and useStop and strategy.position_size == 0
    float buyEntry  = rLine + bufferPts
    float sellEntry = sLine - bufferPts
    float buySL     = rLine - rangeWidth * slMult
    float buyTP     = rLine + rangeWidth * tpMult
    float sellSL    = sLine + rangeWidth * slMult
    float sellTP    = sLine - rangeWidth * tpMult

    qtyBuy  = calcQty(buyEntry,  buySL)
    qtySell = calcQty(sellEntry, sellSL)

    strategy.entry("BuyStop",  strategy.long,  qty = qtyBuy,  stop = buyEntry,  comment = "Stop L")
    strategy.exit ("XL-Stop",  from_entry = "BuyStop",  stop = buySL,  limit = buyTP)

    strategy.entry("SellStop", strategy.short, qty = qtySell, stop = sellEntry, comment = "Stop S")
    strategy.exit ("XS-Stop",  from_entry = "SellStop", stop = sellSL, limit = sellTP)

// Market breakout entries — evaluated each bar
brokeUp   = useMarket and tradingAllowed and strategy.position_size == 0 and close > rLine + bufferPts
brokeDown = useMarket and tradingAllowed and strategy.position_size == 0 and close < sLine - bufferPts

if brokeUp
    float buySL = rLine - rangeWidth * slMult
    float buyTP = rLine + rangeWidth * tpMult
    strategy.entry("BuyMkt", strategy.long, qty = calcQty(close, buySL), comment = "Mkt L")
    strategy.exit ("XL-Mkt", from_entry = "BuyMkt", stop = buySL, limit = buyTP)
    tradedToday := true
    if useStop
        strategy.cancel("SellStop")

if brokeDown
    float sellSL = sLine + rangeWidth * slMult
    float sellTP = sLine - rangeWidth * tpMult
    strategy.entry("SellMkt", strategy.short, qty = calcQty(close, sellSL), comment = "Mkt S")
    strategy.exit ("XS-Mkt",  from_entry = "SellMkt", stop = sellSL, limit = sellTP)
    tradedToday := true
    if useStop
        strategy.cancel("BuyStop")

// Detect stop-order fill: position appears where there was none
if strategy.position_size != 0 and strategy.position_size[1] == 0
    tradedToday := true
    if strategy.position_size > 0
        strategy.cancel("SellStop")
    else
        strategy.cancel("BuyStop")

// -----------------------------------------------------------------------------
// TRAILING STOP
// -----------------------------------------------------------------------------
if useTS and not na(rangeWidth) and rangeWidth > 0
    if strategy.position_size > 0
        float avg        = strategy.position_avg_price
        float activateAt = avg + rangeWidth * tsStart
        if high >= activateAt
            float candidate = close - rangeWidth * tsDist
            tsStopLong := na(tsStopLong) ? candidate : math.max(tsStopLong, candidate)
        if not na(tsStopLong)
            strategy.exit("TSL", stop = tsStopLong)
    else
        tsStopLong := na

    if strategy.position_size < 0
        float avg        = strategy.position_avg_price
        float activateAt = avg - rangeWidth * tsStart
        if low <= activateAt
            float candidate = close + rangeWidth * tsDist
            tsStopShort := na(tsStopShort) ? candidate : math.min(tsStopShort, candidate)
        if not na(tsStopShort)
            strategy.exit("TSS", stop = tsStopShort)
    else
        tsStopShort := na

// -----------------------------------------------------------------------------
// END-OF-SESSION FLATTEN
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

var box orBox = na
if orFinalize and setupValid and showBox
    if not na(orBox)
        box.delete(orBox)
    orBox := box.new(left = nz(sessionStartTime), top = rLine, right = time, bottom = sLine,
                     border_color = color.new(color.red, 40), bgcolor = boxColor,
                     xloc = xloc.bar_time, extend = extend.right)
if not na(orBox) and setupValid
    box.set_right(orBox, time)

plotshape(showEntries and strategy.position_size > 0 and strategy.position_size[1] == 0,
          title = "Long Entry", style = shape.triangleup, location = location.belowbar,
          color = color.new(color.green, 0), size = size.small)
plotshape(showEntries and strategy.position_size < 0 and strategy.position_size[1] == 0,
          title = "Short Entry", style = shape.triangledown, location = location.abovebar,
          color = color.new(color.red, 0), size = size.small)

// Status table
var table statusTbl = table.new(position = position.top_right, columns = 2, rows = 7,
                                 bgcolor = color.new(color.black, 80), border_width = 1)
if barstate.islast
    table.cell(statusTbl, 0, 0, "Status", text_color = color.white, bgcolor = color.new(color.blue, 60))
    table.cell(statusTbl, 1, 0, paused ? "PAUSED" : gapSkipToday ? "GAP-SKIP" : setupValid ? "ARMED" : "WAITING",
               text_color = paused ? color.red : setupValid ? color.lime : color.yellow)
    table.cell(statusTbl, 0, 1, "R", text_color = color.white)
    table.cell(statusTbl, 1, 1, na(rLine) ? "—" : str.tostring(rLine, format.mintick), text_color = rColor)
    table.cell(statusTbl, 0, 2, "S", text_color = color.white)
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
