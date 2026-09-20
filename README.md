# Session-Candle-Breakout-Zones-UTC-TP-FIX

<img width="1471" height="846" alt="image" src="https://github.com/user-attachments/assets/cdafb715-0feb-4c1e-a868-657eb3ba41f3" />


//@version=5
// =====================================================================
//  SRJ UTC NEW RANGE BO ALL ZONES VVIMP  —  v2.1 TP FIX  (timeframe-adaptive)
//  Implements the "Session Candle Breakout Strategy" spec.
//
//  v2 CHANGE — the ZONE now uses the CHART's own timeframe:
//     5M chart  -> 5-minute reference candle / zone
//     15M chart -> 15-minute reference candle / zone
//     1H chart  -> 1-hour reference candle / zone
//     4H chart  -> 4-hour reference candle / zone
//     D  chart  -> daily reference candle / zone
//  The reference candle = the chart-timeframe candle that CONTAINS the
//  selected UTC session time. Change the chart TF and the zones are
//  recalculated and redrawn automatically for that TF (old TF logic is
//  dropped — state resets on every timeframe change).
//
//  Rules kept from spec: zone = full range (body+wicks); delete old zone
//  on new zone; entry only on a CONFIRMED BODY breakout (wick-only = no
//  trade); retest needs price to RETURN then another confirmed body
//  breakout; multi-retest until next zone; SL beyond confirmation candle;
//  TP1..TP4 ladder. Evaluated on closed bars -> non-repainting.
// =====================================================================
indicator("Session Candle Breakout Zones [UTC] — TP FIX", "Session Breakout TP FIX",
     overlay=true, max_boxes_count=500, max_labels_count=500, max_lines_count=500)

// ---------------------------- INPUTS ----------------------------
grpS  = "Sessions (UTC) — reference candle = chart-TF candle containing this time"
useS1 = input.bool(true, "Session 1", inline="s1", group=grpS)
h1v   = input.int(0,  "H", 0, 23, inline="s1", group=grpS)
m1v   = input.int(0,  ":", 0, 59, inline="s1", group=grpS)
useS2 = input.bool(true, "Session 2", inline="s2", group=grpS)
h2v   = input.int(7,  "H", 0, 23, inline="s2", group=grpS)
m2v   = input.int(0,  ":", 0, 59, inline="s2", group=grpS)
useS3 = input.bool(true, "Session 3", inline="s3", group=grpS)
h3v   = input.int(12, "H", 0, 23, inline="s3", group=grpS)
m3v   = input.int(0,  ":", 0, 59, inline="s3", group=grpS)
useS4 = input.bool(true, "Session 4", inline="s4", group=grpS)
h4v   = input.int(13, "H", 0, 23, inline="s4", group=grpS)
m4v   = input.int(30, ":", 0, 59, inline="s4", group=grpS)

grpE   = "Entry rules"
minBodyPct  = input.float(25, "Min body % of candle range (anti-wick)", 0, 100, group=grpE, tooltip="Unclear / small body = NO TRADE. 0 = accept any body.")
strictNext  = input.bool(false, "First entry ONLY on the immediate next candle", group=grpE, tooltip="ON = strict spec reading: only the candle right after the reference candle can give the first entry. OFF = first confirmed body breakout at any time (until the next zone).")
allowRetest = input.bool(true,  "Show retest entries", group=grpE)
allowMulti  = input.bool(true,  "Allow MULTIPLE retest entries", group=grpE)
allowOpp    = input.bool(true,  "Allow opposite-side body breakout as new entry", group=grpE)

grpR    = "Risk display (pips)"
showSL  = input.bool(true, "Show SL", group=grpR)
slPips  = input.float(50, "SL distance beyond zone formation candle", step=1, group=grpR)
showTP  = input.bool(true, "Show TPs", group=grpR)
tp1     = input.float(50,  "TP1", step=1, group=grpR)
tp2     = input.float(100, "TP2", step=1, group=grpR)
tp3     = input.float(150, "TP3", step=1, group=grpR)
tp4     = input.float(200, "TP4", step=1, group=grpR)
pipOver = input.float(0, "Pip size override (price units, 0 = auto)", step=0.0001, group=grpR, tooltip="AUTO: XAU/gold uses 0.10 per pip, so 50 pips = 5.00 in price. Other symbols use 10 x mintick. Use override if your broker uses a different convention.")

grpD     = "Display"
showZone   = input.bool(true, "Show zone", group=grpD)
showBody   = input.bool(true, "Shade reference-candle body inside zone", group=grpD)
deleteOld  = input.bool(true, "Delete old zone when new zone forms", group=grpD)
showBuy    = input.bool(true, "Show BUY entries", group=grpD)
showSell   = input.bool(true, "Show SELL entries", group=grpD)
showTable  = input.bool(true, "Show status table", group=grpD)
colZone    = input.color(color.new(#2962FF, 0), "Zone color", group=grpD)
colBull    = input.color(#26a69a, "Bull color", group=grpD)
colBear    = input.color(#ef5350, "Bear color", group=grpD)

// Pip conversion
// IMPORTANT: On XAUUSD this indicator treats 1 pip = 0.10 price units.
// Therefore: 50 pips = 5.00, 100 pips = 10.00, 150 pips = 15.00, 200 pips = 20.00.
bool isGold = str.contains(str.upper(syminfo.ticker), "XAU") or str.contains(str.upper(syminfo.description), "GOLD")
float autoPip = isGold ? 0.10 : syminfo.mintick * 10
float pip = pipOver > 0 ? pipOver : autoPip

// -------- reference-candle detection on the CHART timeframe --------
// A bar is a reference candle if an enabled session timestamp falls
// inside this bar's time span [bar_open, bar_close).  This works
// uniformly for 5M / 15M / 30M / 1H / 4H / Daily, so the selected chart
// timeframe alone controls which candle becomes the zone.
yr = year(time,  "UTC")
mo = month(time, "UTC")
dy = dayofmonth(time, "UTC")

contains(_use, _h, _m) =>
    ts = timestamp("UTC", yr, mo, dy, _h, _m)
    _use and time <= ts and ts < time_close

isRef1 = contains(useS1, h1v, m1v)
isRef2 = contains(useS2, h2v, m2v)
isRef3 = contains(useS3, h3v, m3v)
isRef4 = contains(useS4, h4v, m4v)
isRef  = isRef1 or isRef2 or isRef3 or isRef4

p1 = str.tostring(h1v) + ":" + (m1v<10?"0":"") + str.tostring(m1v)
p2 = str.tostring(h2v) + ":" + (m2v<10?"0":"") + str.tostring(m2v)
p3 = str.tostring(h3v) + ":" + (m3v<10?"0":"") + str.tostring(m3v)
p4 = str.tostring(h4v) + ":" + (m4v<10?"0":"") + str.tostring(m4v)
sName = isRef1 ? "S1 " + p1 : isRef2 ? "S2 " + p2 : isRef3 ? "S3 " + p3 : isRef4 ? "S4 " + p4 : ""
tfTag = " (" + timeframe.period + " zone)"

// --------------------------- STATE ---------------------------
var float zHi   = na
var float zLo   = na
var float zBHi  = na
var float zBLo  = na
var string zName = na
var int   dir     = 0      // 0 none, 1 bullish, -1 bearish
var bool  armed   = false  // price returned to zone since last entry
var int   retests = 0
var int   barsInZone = 0   // chart bars since zone creation
var box   zBox    = na
var box   zBodyBox = na
var label zLab    = na
var line  slLine  = na
var label slLab   = na
var array<line>  tpLines = array.new<line>()
var array<label> tpLabs  = array.new<label>()

// reset everything if the chart timeframe changes (drop old-TF logic)
var string lastTF = timeframe.period
if timeframe.period != lastTF
    lastTF := timeframe.period
    box.delete(zBox)
    box.delete(zBodyBox)
    label.delete(zLab)
    line.delete(slLine)
    label.delete(slLab)
    if array.size(tpLines) > 0
        for ln in tpLines
            line.delete(ln)
        array.clear(tpLines)
    if array.size(tpLabs) > 0
        for lb in tpLabs
            label.delete(lb)
        array.clear(tpLabs)
    zHi := na
    zLo := na
    zName := na
    dir := 0
    armed := false
    retests := 0
    barsInZone := 0

// ----------------------- SIGNAL LOGIC -----------------------
bool sigBuy = false, sigSell = false, sigRBuy = false, sigRSell = false

if barstate.isconfirmed and not isRef and not na(zHi)
    barsInZone += 1
    float rng    = high - low
    bool  bodyOK = rng > 0 and math.abs(close - open) / rng * 100.0 >= minBodyPct
    bool  bullBrk = high > zHi and close > zHi and close > open and bodyOK
    bool  bearBrk = low  < zLo and close < zLo and close < open and bodyOK
    bool  firstOK = not strictNext or barsInZone == 1

    if dir == 0
        if bullBrk and firstOK
            sigBuy := true
        else if bearBrk and firstOK
            sigSell := true
    else if dir == 1
        if armed and bullBrk and allowRetest and (allowMulti or retests < 1)
            sigRBuy := true
        else if bearBrk and allowOpp
            sigSell := true
    else
        if armed and bearBrk and allowRetest and (allowMulti or retests < 1)
            sigRSell := true
        else if bullBrk and allowOpp
            sigBuy := true

bool anySig = sigBuy or sigSell or sigRBuy or sigRSell

// clear old SL/TP drawings on a new signal or a new zone
if anySig or isRef
    line.delete(slLine)
    label.delete(slLab)
    if array.size(tpLines) > 0
        for ln in tpLines
            line.delete(ln)
        array.clear(tpLines)
    if array.size(tpLabs) > 0
        for lb in tpLabs
            label.delete(lb)
        array.clear(tpLabs)

// ------------------- APPLY SIGNAL / DRAW -------------------
if anySig
    int sDir = (sigBuy or sigRBuy) ? 1 : -1
    if sigBuy
        dir := 1
        armed := false
        retests := 0
    if sigSell
        dir := -1
        armed := false
        retests := 0
    if sigRBuy or sigRSell
        retests += 1
        armed := false

    bool showThis = sDir == 1 ? showBuy : showSell
    if showThis
        string txt = (sigRBuy or sigRSell ? "RETEST ENTRY – " : "") + (sDir == 1 ? "BUY" : "SELL") + "\n@ " + str.tostring(close, format.mintick)
        label.new(bar_index, sDir == 1 ? low : high, txt,
             style = sDir == 1 ? label.style_label_up : label.style_label_down,
             color = color.new(sDir == 1 ? colBull : colBear, sigRBuy or sigRSell ? 35 : 0),
             textcolor = color.white, size = size.small)

    float entry = close
    // FIXED SL: always use the ORIGINAL zone formation candle.
    float slP = sDir == 1 ? zLo - slPips * pip : zHi + slPips * pip
    int labelOffset = 8
    if showSL
        slLine := line.new(bar_index, slP, bar_index + 1, slP, extend = extend.right, color = colBear, style = line.style_dashed, width = 2)
        slLab  := label.new(bar_index + labelOffset, slP, "SL • " + str.tostring(slPips) + " pips • " + str.tostring(slP, format.mintick), style = label.style_label_left, color = color.new(colBear, 15), textcolor = color.white, size = size.small)
    if showTP
        float tp1Price = sDir == 1 ? entry + tp1 * pip : entry - tp1 * pip
        float tp2Price = sDir == 1 ? entry + tp2 * pip : entry - tp2 * pip
        float tp3Price = sDir == 1 ? entry + tp3 * pip : entry - tp3 * pip
        float tp4Price = sDir == 1 ? entry + tp4 * pip : entry - tp4 * pip

        array.push(tpLines, line.new(bar_index, tp1Price, bar_index + 1, tp1Price, extend = extend.right, color = colBull, style = line.style_dashed, width = 1))
        array.push(tpLabs, label.new(bar_index + labelOffset, tp1Price, "TP1 • " + str.tostring(tp1) + " pips • " + str.tostring(tp1Price, format.mintick), style = label.style_label_left, color = color.new(colBull, 15), textcolor = color.white, size = size.small))

        array.push(tpLines, line.new(bar_index, tp2Price, bar_index + 1, tp2Price, extend = extend.right, color = colBull, style = line.style_dashed, width = 1))
        array.push(tpLabs, label.new(bar_index + labelOffset, tp2Price, "TP2 • " + str.tostring(tp2) + " pips • " + str.tostring(tp2Price, format.mintick), style = label.style_label_left, color = color.new(colBull, 15), textcolor = color.white, size = size.small))

        array.push(tpLines, line.new(bar_index, tp3Price, bar_index + 1, tp3Price, extend = extend.right, color = colBull, style = line.style_dashed, width = 1))
        array.push(tpLabs, label.new(bar_index + labelOffset, tp3Price, "TP3 • " + str.tostring(tp3) + " pips • " + str.tostring(tp3Price, format.mintick), style = label.style_label_left, color = color.new(colBull, 15), textcolor = color.white, size = size.small))

        array.push(tpLines, line.new(bar_index, tp4Price, bar_index + 1, tp4Price, extend = extend.right, color = colBull, style = line.style_dashed, width = 1))
        array.push(tpLabs, label.new(bar_index + labelOffset, tp4Price, "TP4 • " + str.tostring(tp4) + " pips • " + str.tostring(tp4Price, format.mintick), style = label.style_label_left, color = color.new(colBull, 15), textcolor = color.white, size = size.small))

// arming: price RETURNS to the zone after an entry (touch alone = no entry)
if barstate.isconfirmed and not anySig and not isRef and dir != 0 and not na(zHi)
    if dir == 1 and low  <= zHi
        armed := true
    if dir == -1 and high >= zLo
        armed := true

// -------------------- NEW ZONE CREATION --------------------
if barstate.isconfirmed and isRef
    if deleteOld
        box.delete(zBox)
        box.delete(zBodyBox)
        label.delete(zLab)
    else
        if not na(zBox)
            box.set_right(zBox, time)
        if not na(zBodyBox)
            box.set_right(zBodyBox, time)
    zHi := high
    zLo := low
    zBHi := math.max(open, close)
    zBLo := math.min(open, close)
    zName := sName + tfTag
    dir := 0
    armed := false
    retests := 0
    barsInZone := 0
    if showZone
        zBox := box.new(left = time, top = high, right = time, bottom = low, xloc = xloc.bar_time, border_color = colZone, border_width = 1, bgcolor = color.new(colZone, 88))
        if showBody
            zBodyBox := box.new(left = time, top = zBHi, right = time, bottom = zBLo, xloc = xloc.bar_time, border_color = na, bgcolor = color.new(colZone, 72))
        zLab := label.new(x = time, y = high, text = zName, xloc = xloc.bar_time, style = label.style_label_down, color = color.new(colZone, 25), textcolor = color.white, size = size.small)

// keep the active zone extended to the right; tint by direction
if not na(zBox)
    box.set_right(zBox, time)
    box.set_border_color(zBox, dir == 1 ? colBull : dir == -1 ? colBear : colZone)
if not na(zBodyBox)
    box.set_right(zBodyBox, time)

// ------------------------ STATUS TABLE ------------------------
var table t = table.new(position.top_right, 2, 6, bgcolor = color.new(color.black, 80), border_width = 1, border_color = color.new(color.gray, 60))
if barstate.islast and showTable
    string st = na(zHi) ? "waiting first session…" : dir == 0 ? (strictNext and barsInZone >= 1 ? "next-candle window passed" : "waiting first body breakout") : (dir == 1 ? "BULLISH" : "BEARISH") + (armed ? " • retest ARMED" : " • waiting return to zone")
    table.cell(t, 0, 0, "Chart TF", text_color = color.gray, text_size = size.tiny)
    table.cell(t, 1, 0, timeframe.period + " zone", text_color = color.yellow, text_size = size.tiny)
    table.cell(t, 0, 1, "Zone",  text_color = color.gray, text_size = size.tiny)
    table.cell(t, 1, 1, na(zName) ? "—" : zName, text_color = color.white, text_size = size.tiny)
    table.cell(t, 0, 2, "High",  text_color = color.gray, text_size = size.tiny)
    table.cell(t, 1, 2, na(zHi) ? "—" : str.tostring(zHi, format.mintick), text_color = color.white, text_size = size.tiny)
    table.cell(t, 0, 3, "Low",   text_color = color.gray, text_size = size.tiny)
    table.cell(t, 1, 3, na(zLo) ? "—" : str.tostring(zLo, format.mintick), text_color = color.white, text_size = size.tiny)
    table.cell(t, 0, 4, "State", text_color = color.gray, text_size = size.tiny)
    table.cell(t, 1, 4, st, text_color = color.white, text_size = size.tiny)
    table.cell(t, 0, 5, "Retests", text_color = color.gray, text_size = size.tiny)
    table.cell(t, 1, 5, str.tostring(retests), text_color = color.white, text_size = size.tiny)

// -------------------------- ALERTS --------------------------
alertcondition(sigBuy,   "BUY Entry",          "Session Breakout: BUY entry (confirmed body breakout above zone)")
alertcondition(sigSell,  "SELL Entry",         "Session Breakout: SELL entry (confirmed body breakout below zone)")
alertcondition(sigRBuy,  "RETEST ENTRY – BUY", "Session Breakout: RETEST ENTRY BUY")
alertcondition(sigRSell, "RETEST ENTRY – SELL","Session Breakout: RETEST ENTRY SELL")
alertcondition(sigBuy or sigSell or sigRBuy or sigRSell, "Any Entry", "Session Breakout: new entry signal")
alertcondition(isRef,    "New Zone Created",   "Session Breakout: new reference zone created")
// Updated on 20-09-2026 by Melbin George
