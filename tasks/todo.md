# Stocks app for AWTRIX NG (TC002)

- [x] Read the Berry API reference and skill (docs/examples/berry-app-system-prompt.md)
- [x] Pick a keyless quote source: Yahoo chart endpoint (needs any User-Agent; 429 without)
- [x] Write Stocks.ax: 4 symbols, change %/amount, up/down colours, knob paging, stored last quotes
- [x] Install via PUT /api/v1/apps/script/Stocks, error null
- [x] Verify on panel: AAPL green up, MSFT and GOOG red down, paging and dots work
- [x] v1.1 after feedback: drop the percentage, ticker in large font, chevron only, Parqet logo on the left
- [x] v1.3 after feedback: transparent logos (nvstly/icons) with FMP fallback on 404; Parqet dropped (white squares)
- [x] v2.0: ten stocks in one comma-separated field; Display = pages | ticker; ticker shows % change, coloured
- [x] v2.1: logos inline in the ticker (own scroller, scroll_text cannot carry images), gap setting, continuous loop
- [x] v2.2: market banner (closed / opening in N mins / closing in N mins) as first page and ticker lead
- [x] v2.3: big-move card (fireworks / rain), UPDATED time while open, no refetch after close, earnings countdown with BMO/AMC
- [x] v3.0: layout D locked in from docs/layout-options.html (logo | ticker + 1-dp change / price in large font coloured by direction); ticker scrolls the same block
- [x] v3.1: market glyphs drawn on the clock (moon, sun, sunrise, sunset) on the banner in both modes
- [x] v3.2: every ticker block pauses; fireworks/rain behind a frame, optional and off by default
- [x] v3.3: knob steps between stocks in ticker mode too (jump + hold); knob press toggles percent / amount, stored
- [x] v3.4: change text fitted to the room beside the ticker (proportional font); logo gap 1 px
- [x] v3.5: change without sign (always fits with %), direction triangle lower-right (pages) or after the price (ticker)
- [x] v3.7: ticker in the large font (rows 0-6) on the same baseline; change stays small; widths unchanged
- [x] v4.0: 20 stocks, speed default 200 %, pause default 2 s, logo source remembered per symbol, symbol whitelist, NO SUCH TICKER page, rewrite for clarity
- [ ] Optional: tune colours or page time to taste in the web UI

## Review 2026-10-16

Installed on the clock, AWTRIX NG 1.2.2. Compiled first try; all three
default stocks rendered correctly within seconds. Design choice: one stock per page
because a 52 px row fits about 12 small-font characters, not ticker + price + change.

## v1.1 2026-10-16

The owner found v1.0 too busy (ticker + percent + price). Now: logo 16x16 left, ticker and
price in the large font, a 7x4 two-pixel chevron after the price, no percent. Logos from
`https://assets.parqet.com/logos/symbol/<SYM>?format=png&size=64` (keyless; clearbit,
synthfinance and logo.dev did not work without a key). Verified on the panel: AAPL green ^,
MSFT and GOOG red v, logos rendered with transparent background shown as black.

## v1.3 2026-10-16

The owner disliked the white squares behind Parqet logos. Compared on the panel: nvstly/icons
(transparent, full-bleed, but 404 for BRK-A/B, SPY, QQQ) vs financialmodelingprep image-stock
(complete, but AAPL is a black apple in a white circle). Chosen: nvstly first, FMP on 404.
`http.request("HEAD", ...)` returned status 0 on the TC002, so the probe is `http.get` with
`cap: 16`. Verified: AAPL/MSFT/GOOG transparent; BRK-B and SPY via fallback.

## v2.0 2026-10-16

Ten stocks would need 14 settings with one field each, over the 12-field cap, so the
symbols became one comma-separated `stocks` text field (parsed with re.matchall, max 10).
Ticker mode uses the column form of scroll_text() with [text, colour] pieces, large font,
baseline 11 so the line sits in the middle of the 16 rows, {"repeat": 1} so the rotation
waits for one full run, and duration() returns 0 in that mode. Pieces are rebuilt once per
fetch callback, never in draw(). Verified with a 10-symbol test app: scroll, colours, and
pages mode unchanged.

## v2.1 2026-10-16

The owner wanted logos in the ticker and more space between stocks. scroll_text() only
moves text, so the ticker is now drawn by hand: x = width - elapsed*pxps/1000 mod (L+width),
segment = 16 px logo + 2 + text_width(pieces) + gap. Widths are measured in measure(),
called once after each rebuild (dirty flag) from draw() or duration(), with font("large")
set first; never per frame. Only segments intersecting the panel are drawn, so at most
two or three icon() calls a frame (limit is 4). The clock has autoTransition off, so a
run-once ticker would end on a blank panel; the line loops instead, and duration() still
returns one pass for clocks that rotate. Gap is a `number` setting, default 24 px.

## v2.2 2026-10-16

Banner from the quote's `currentTradingPeriod.regular` (start, end, gmtoffset; key order
varies per symbol, so three searches inside the block). keep raised to 900 bytes. status()
runs from loop() and only touches members when the text changes (dirty -> remeasure).
Tested by sed-injecting synthetic `now` into test copies (ps-600, pe-300, real time) and
installing them one at a time: five apps fetching at once got 429s from Yahoo and showed
`...`, which first looked like a parsing bug. The owner had meanwhile switched the live app
to ticker mode with five stocks; settings untouched.

## v2.3 2026-10-16

Earnings from Nasdaq's per-symbol sentence ("...on  11/02/2026 after market close.");
Yahoo quoteSummary needs a crumb now, and the Nasdaq calendar-by-date lists the time before
the symbol, which find/keep cannot look back for. Days-since-epoch via Hinnant's
days_from_civil in integer Berry. Pages are now a slot list [stock, kind, ms] walked by
time, so cards and earnings pages can have their own dwell. Ticker pause: freeze `el`,
accumulate the paused time in `shift`, fire once per pass via a pass counter. First cut
froze the scrolled line, which hid the percentage (the owner, live on the clock); replaced
with a two-row card used by both modes. The patch script once aborted on its own
over-strict assert and the tests ran the old file: check `grep` before trusting a patch.
Closed-market skip verified with a 1-minute refresh: round at start, one more once the
session is known, then skipped.

## v3.x 2026-10-16

The owner found the card transition awkward and asked for options first. Four layouts were
rendered on the clock (scratch app Topt) and published as docs/layout-options.html; he
picked D. Glyph widths: every font is 4 px/char except matrix-*8x6 (6 px), so the only
"bigger" price is 8 rows tall and a six-character price does not fit beside a logo. D uses
the 7-row large font instead and keeps two decimals. The change sits right after the
ticker (advance + 1 px), which ends exactly at column 51 for four-letter tickers; five
letters drop the % sign. Ticker blocks measure the small row and the large row separately
(w, w2). Each block and the banner stop once per pass (fired[] / firedB by pass number).
Effects moved behind `fx` (default off) after he said every stock should pause instead.
The % glyph in the small font is a 5-wide stylised shape; a pixel dump showed it whole.

## v3.3 2026-10-16

Knob in ticker mode: find the current block (last one whose left edge has entered the
panel), compute the neighbour's offset, set t0 so `el` puts it at its stop, start a pause,
and mark earlier blocks as fired for this pass. The target `el` must round up
(ceil(pos*1000/pxps)) or the block lands 1-2 px right of its stop and the change text
clips at the edge. Tested by firing on_knob() from timer.after() in a copy, since the knob
cannot be pressed over the API.

## v3.4 2026-10-16

The owner saw % signs cut off and asked for a smaller logo. icon() fits any web image to a
display-height square, so the logo cannot shrink. A pixel dump showed the real cause: the
small font is proportional (M is 5 px), so text_width("AAPL") = 16 misled the layout. Fix:
measure each ticker's ink in rebuild() (font("small") first), and pick the longest change
string that fits in 35 - ticker - 2 columns. Verified top-row ink never passes column 50
for MSFT, META, GOOGL, AMZN in both percent and amount modes.

## v3.5 2026-10-16

Mixed % signs read as a bug to the owner. Measured ink: MSFT 17, AMZN 18, "-1.9%" 17, "%" 3;
the sign (4 px) and decimal (6 px) cost the space, not the sign. Chosen from seven options:
drop the sign, keep % and one decimal, add a 5x3 triangle in the change colour for
colour-blind readers (lower-right in pages, after the price in ticker blocks; w2 += 7).

## 3.6 tried and reverted 2026-10-16

Small-logo trial: 10x10 GIFs stored in the clock's /ICONS under the ticker name (POST
/api/v1/files?dir=/ICONS, GIF only; PNG is refused) are drawn at their own size, which
freed room for a signed change. The owner did not like the look; reverted to 3.5 the same
hour, trial icons deleted from the clock, helper script removed. Notes kept here in case
it comes up again: sips can do the resize and GIF conversion; icon() draws at most four
different icons per frame.

## v3.7 2026-10-16

Ticker now in the 7-row large font on baseline 6 while the change stays small on the same
baseline; large and small share character widths for four-letter tickers, so the fit is
unchanged. GOOGL is 23 px in large (19 small), so five-letter tickers lose the decimal
("2%"). sw[] is measured with font("large") in rebuild().

## v4.0 2026-10-16

Rewrite of the whole file (the patch-on-patch had drifted). Limits and defaults per
The owner: 20 symbols (store keys q/e/lg 0-19), speed 200 %, pause 2 s. Logo source stored
as lg<i> = [sym, url] so the 16-byte probe runs once per symbol, not per start; the image
cache is the clock's. Symbol filter tightened to [A-Za-z0-9.^-], 12 chars, ^ -> %5E.
Settings and stored values go through num()/isinstance checks. Yahoo answers 404 for an
unknown symbol: bad[i] -> "NO SUCH TICKER" under the name (small font, 50 px, fits
without a logo), earnings fetch skipped for it. The owner's live settings (speed 243, pause
3, 11 stocks) are his own and were left alone; defaults only apply to a fresh install.
