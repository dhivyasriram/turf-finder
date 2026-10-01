---
name: turf-finder
description: Finds football turf/ground slots on Playo (playo.co) in Bangalore. Takes three optional inputs in any order or phrasing: a date (defaults to the next upcoming Sunday), one or more specific venue names (defaults to searching South and Central Bangalore for the top 5 options), and a time range for the start time, e.g. "9am to 11am" (defaults to 7-9am on Saturdays/Sundays; on a weekday date with no time given, the skill asks for one before searching). When venues are named, only those venues are checked. Looks for continuous 90-minute slots (or a duration the user states) starting within the time range. Outputs a table (Venue Name, Field Size, Available Time Slots, Cost, Location) in chat and saves it as a Google Sheet in the user's own Google Drive (default name "Turf Finder results", configurable). Trigger this whenever the user asks to find a turf, football ground, or playing slot in Bangalore, mentions booking football on Playo, or asks "/turf-finder" (with or without a date and/or venue names, e.g. "/turf-finder Tackle and Games Period on Oct 11" or "/turf-finder Oct 7 6pm to 8pm"), even if they don't spell out every filter each time — reuse the defaults below.
---

# Turf Finder

Finds bookable football turfs on Playo for a target date and start-time window (by default the
upcoming Sunday, 7-9am), either across South/Central Bangalore or at specific venues the user
names, and reports back as a table in chat plus a Google Sheet on Drive.

> **Recommended model: Opus (or Fast mode on Opus for quicker output).** This can't be enforced
> from skill frontmatter — a skill runs on whatever model the session uses — so it's a manual
> choice when invoking `/turf-finder`. Opus is worth it here: this is long-horizon browser
> automation over a flaky React SPA, and — more importantly — judging real 90-minute availability
> is a reasoning trap. An earlier version inferred slots from the coarse Start Time list and
> produced false positives (see §3); getting it right needs careful reasoning about what each
> signal actually proves, which is exactly where a weaker model takes the plausible-looking
> shortcut. Wrong answers here cost a real wasted trip, so don't trade intelligence for speed.

## 0. Parse the inputs

The skill takes three **optional** inputs. The user may give any combination of them, or none, in
any order and any phrasing — work out which is which yourself rather than expecting a fixed format.

| Input | Examples of how it may arrive | If absent |
|---|---|---|
| **Date** | "2026-10-11", "Oct 11", "11th", "this Saturday", "next Sunday", "tomorrow" | Upcoming Sunday (step 1) |
| **Venue(s)** | "Tackle Jayanagar", "tackle and games period", "Depot18, Verve, One 8", "the Tiger 5 at Dairy Circle" | Area search: top 5 in South/Central (step 2A) |
| **Time range** | "9am to 11am", "9-11am", "between 6 and 8pm", "after 7pm", "7:30am" | Sat/Sun: 7-9am. Mon-Fri: **ask** (step 1B) |
| **Duration** (rare) | "for 2 hours", "1 hr game", "2hrs" | 90 minutes |

How to tell them apart:
- Anything that reads as a calendar day (a date, a weekday name, a relative day like "tomorrow" or
  "this weekend") is the **date**. Everything else that names a place, club, turf or brand is a
  **venue**. Locality words attached to a venue name ("Tackle *Jayanagar*", "Tiger 5 *Dairy
  Circle*") are part of that venue's name, not a request to search that area.
- Anything that reads as a clock time or time range ("9am to 11am", "6-8pm", "morning 7 to 9") is
  the **time range**; a length of play ("2 hours", "for an hour") is the **duration**. Don't
  confuse a date number ("the 11th") with a time ("11am").
- Split multiple venues on commas, "and", "&", "+", or new lines. Keep informal or partial names
  as given (e.g. "one8", "tackle") — step 2B matches them fuzzily.
- Relative days resolve against today's date, computed (see step 1). "This weekend" with no
  day means Sunday. A weekday name alone ("Saturday") means the next occurrence, including today.
- If an input is genuinely ambiguous in a way that changes the result (e.g. "11/10" could be 11 Oct
  or 10 Nov, or a word could be either a venue or a locality), ask one short question before
  searching instead of guessing. Don't ask about things you can resolve yourself.

Before searching, restate what you understood in one line, e.g.
"Checking **Tackle Jayanagar** and **Games Period** for **Sunday, Oct 11, 2026**, 90-min slots
starting **9:00-11:00 AM**." or
"Searching South/Central Bangalore for **Saturday, Oct 10, 2026**, 90-min slots starting
**7:00-9:00 AM** (default weekend window)." Always include the start window and duration.

## 1. Resolve the target date

- If the user gave a date, use it **as given, even if it isn't a Sunday** — an explicit date always
  wins. Compute its weekday (below), state it back, and use that weekday's pricing band in step 4.
- Otherwise, use the **upcoming Sunday**: if today is a Sunday, that's today; otherwise the next
  Sunday after today.
- **Compute the weekday — do not guess it.** The date usually arrives as a bare ISO string (e.g.
  "2026-09-15") with no day-of-week. Never attach a day name by feel; actually derive it. The
  reliable way is to shell out rather than do mental math: `date -d 2026-09-15 +%A` (GNU) or
  `date -jf %Y-%m-%d 2026-09-15 +%A` (BSD/macOS) returns the weekday, and
  `date -d "2026-09-15 +$(( (7 - $(date -d 2026-09-15 +%u) % 7) % 7 )) days" +%Y-%m-%d` gives the
  upcoming Sunday directly. If no shell is available, compute it explicitly (e.g. Zeller's
  congruence or day-of-year mod 7 from a known anchor) and show your working — don't assert.
- **When defaulting (no date given), verify the resolved target date actually IS a Sunday before
  searching.** This is a real check, not a formality: run the weekday computation on the
  *resolved* date and confirm it returns Sunday. An earlier run asserted "today is Monday" without computing it, landed on a date that was
  really a Monday (not the Sunday intended), and scraped the whole thing for the wrong day —
  including weekday instead of weekend pricing. The restate-the-date step only catches errors if
  the date was independently derived; echoing an unchecked assumption catches nothing.
- **When the user gave a date, still compute its weekday** — never attach a day name by feel. Also
  sanity-check that the date is today or later; if it's in the past (e.g. "Oct 4" when today is
  Oct 6), assume the next year's date only if that's clearly meant, otherwise ask.
- State the resolved date *with its computed weekday* back to the user before searching (e.g.
  "Looking at Sunday, Sept 20, 2026"). In default mode, if the verification shows it is *not* a
  Sunday, stop and recompute rather than proceeding.

## 1B. Resolve the time window and duration

The **time range is a range of allowed start times**, not a window the whole game must fit into.
"9am to 11am" means the game may start at any bookable time from 9:00 up to and including 11:00
(so with 90 minutes it can end as late as 12:30). This matches the default: 7-9am means starts at
7:00, 7:30, 8:00, 8:30 or 9:00, ending 8:30-10:30.

- **Time range given** → use it, on any day (weekday or weekend).
- **No time range, and the target date is a Saturday or Sunday** (including the default upcoming
  Sunday) → use the default start window **7:00-9:00 AM**.
- **No time range, and the target date is Monday-Friday** → **stop and ask the user for a time
  range. Do not start searching without one.** Public holidays that fall on a weekday count as
  weekdays here — don't try to detect holidays. Ask one short question, e.g. "Oct 7 is a Wednesday
  — what start-time range should I check (e.g. '6pm to 8pm')?" Then wait for the answer.
- **Parsing the range:**
  - "9-11am" / "9 to 11 am" → the am/pm marker applies to both ends: 9:00-11:00 AM.
  - A single time ("7:30am") → that one start time only.
  - Open-ended ("after 7pm", "before 8am") → from that time to the venue's closing time / from
    opening time to that time; state the concrete range you'll use.
  - If am/pm can't be inferred ("7 to 9" with no marker and no context like "morning"/"evening"),
    or the end is before the start, ask rather than guess.
  - Candidate start times are every half-hour mark from start to end inclusive (e.g. 9:00-11:00 →
    9:00, 9:30, 10:00, 10:30, 11:00). Snap odd values inward to the half-hour (9:15 → 9:30).
- **Duration:** 90 minutes unless the user explicitly states a different length ("2 hours",
  "1 hr"). The time range never sets the duration by itself — "9am to 11am" is still a 90-minute
  game. If a duration is given, use it in step 3 in place of 90 minutes everywhere.

Throughout the rest of this skill, **"the start window"** means the resolved range and **"the
duration"** means the resolved length. Where the text says "7:00-9:00" or "90 minutes", it is
describing the default — substitute the resolved values.

## 2. Decide which venues to check

There are two modes, chosen by whether the user named venues in step 0.

### 2A. Area search (no venues named) — build a candidate list, ordered by rating

Go to `https://playo.co/venues/bangalore/sports/football` (this is the direct URL for Bangalore
football venues — no need to click through the city picker). This lists venues sorted by distance
with their locality name under each card (e.g. "Jayamahal Palace Road", "Ashok Nagar").

Playo has no built-in "South/Central Bangalore" filter, so classify each venue's locality
yourself:

- **Central Bangalore**: MG Road, Brigade Road, Ashok Nagar, Shivajinagar, Vasanth Nagar,
  Richmond Town, Museum Road, Cunningham Road, Sampangi Rama Nagar, Palace Road, Gandhi Nagar,
  Majestic, Cubbon Park, Infantry Road, Commercial Street, Jayamahal, RBANM's Ground, Malleshwaram.
- **South Bangalore**: Jayanagar, JP Nagar, Basavanagudi, BTM Layout, Banashankari, Lalbagh area,
  Padmanabhanagar, Kumaraswamy Layout, Girinagar, Wilson Garden, Bannerghatta Road (northern
  stretch near Dairy Circle).
- Treat Indiranagar, Frazer Town, Cooke Town, HSR Layout, Whitefield, Koramangala, etc. as
  East/other Bangalore — **exclude** these.

If a locality is ambiguous, a quick web search of "<locality> Bangalore zone" resolves it — don't
guess silently.

From the venues that fall in South or Central Bangalore and are marked **Bookable**, build a
candidate list ordered by rating (Featured / higher-rated first). **The goal is 5 venues that
actually have a qualifying slot (the duration, starting in the start window) on the target date — not just the first 5 on the
list.** In testing, Sunday-morning demand is high enough that most venues come up empty (4 of the
first 5 checked had zero availability), so expect to work through more than 5 candidates. Check
them in rating order, keep every venue that has at least one qualifying window (step 3), and stop
once you have 5 such venues — or once you've checked ~15 candidates without reaching 5, whichever
comes first (if you hit the 15 cap short of 5, report however many you found rather than padding
the table with empty rows).

### 2B. Named venues — check only those venues

When the user names one or more venues, **check only those venues**. Do not add other venues, do
not apply the South/Central zone filter, and the 5-venue target and ~15-candidate cap don't apply.

1. **Find each venue's Playo page.** Match names fuzzily and case-insensitively ("tackle" →
   "Tackle Jayanagar", "one8" → "One 8 Sports Club", "depot 18" → "Depot18 - Sports"). Ways to
   find it, fastest first:
   - Check the venue list at `https://playo.co/venues/bangalore/sports/football` with `find` for
     the name (click **Show More** to load more cards if it isn't in the first batch — venues are
     sorted by distance, so farther ones only appear after a few Show Mores).
   - If still not found, web search `playo <venue name> bangalore` — the result URL is the venue
     page directly (`https://playo.co/venues/bengaluru/<slug>`), which you can navigate to.
   - Once on the venue detail page, confirm the name and locality match what the user meant.
2. **Several listings match one name** (e.g. "Tiger 5 Dairy Circle" → Facility A and Facility B):
   check **all** clearly matching listings, give each its own row, and say which listings the name
   matched. Don't check loosely related venues that merely share a word.
3. **No listing matches**: don't guess a substitute. Add a row with the user's venue name and
   "Not found on Playo" in Available Time Slots (other cells "—"), and mention it in the summary.
4. **Venue outside South/Central** (e.g. Indiranagar): check it anyway — the user named it. The
   Location column still shows its real zone, e.g. "Indiranagar (East)".
5. **Not bookable / no football**: if the venue exists but isn't marked Bookable, or doesn't list
   Football, add a row saying so ("Not bookable on Playo" / "No football on Playo") instead of
   checking slots.
6. For every named venue that can be checked, run step 3 fully (all offered start times in the
   start window) and step 4 for field size and cost. Since the user cares about these specific
   venues, list **every** qualifying window you find, not just the first.

## 3. Check each venue for a continuous slot of the chosen duration in the start window

**The aggregate "Start Time" list is not trustworthy — do not use it to infer availability.**
Earlier versions of this skill inferred a 90-minute window from that list alone (treating any two
consecutive available marks as proof of a free 90-minute block) and this produced false positives:
verified against the actual Court dropdown, venues the list said were open at 7:00 and 7:30 AM
turned out to have **zero** bookable courts for either hour, let alone a continuous 90 minutes.
The list appears to reflect something looser than real per-court booking state (possibly venue
operating windows plus partial cache), and cannot be relied on. **The Court dropdown, with a real
duration set, is the only signal that reflects actual bookability** — always confirm through it.

Open each shortlisted venue's page and click **Book Now** — this opens a booking widget with
Sport, Date, Start Time, Duration, and Court fields.

1. Set **Sport** to Football (or the closest equivalent — some venues list it as "Turf Football",
   5-a-side, etc.).
2. Set **Date** to the resolved target date using the date-picker calendar icon.
**Primary method — set Duration to 90 minutes and read the Court dropdown.** This asks Playo the
exact question you care about (which courts are free for a continuous 90-minute block starting at
T), so there's no inference. Order of operations matters:

3. For each candidate start time in the start window — by default **7:00, 7:30, 8:00, 8:30,
   9:00 AM**; otherwise every half-hour mark in the resolved range (step 1B) — (skip any not
   offered by the Start Time dropdown at all — if a time isn't listed as an option, it can't be booked, full stop):
   a. Select that start time, then press `Escape` to close the Start Time listbox (see Court-
      dropdown notes below — the listbox overlaps other controls if left open).
   b. **Bump Duration from "1 Hr" to "1 Hr 30 Mins"** by clicking the Duration `+` button once —
      *before* selecting any court. Verify with `get_page_text` that it reads "1 Hr 30 Mins".
      (If the user set a different duration, click `+`/`−` until it reads that duration instead,
      e.g. "2 Hr". The first `+` click sometimes doesn't register — always verify the text. Some
      venues step in whole hours only: if the exact duration can't be set, use the next longer
      one — e.g. 2 Hr for a 90-minute ask — and note in the output that it has to be booked that
      way.)
      Doing this while no court is selected is what avoids the cart-add glitch (see reliability
      notes): with nothing selected, `+` simply increments the duration; if you select a court
      first and then click `+`, some venues instead treat it as "add this 1-hour slot to cart and
      extend," which is the wrong path.
   c. Open the **Court** dropdown and read which named courts appear (e.g. "6a side Turf 2",
      "5 a side Court 1"). **Every court listed here is bookable for the full 90 minutes T to
      T+90** — that's the whole point of setting the duration first. Record the exact court names
      and the price shown (which is now the price for the whole duration; divide by 1.5 — or the duration in hours — for the hourly rate if
      you want to report per-hour).
   d. A venue can have zero, one, or several qualifying windows for the date — record each as
      "start–end" (e.g. "7:00 AM - 8:30 AM").
4. **Check EVERY offered start time in the start window before rejecting a venue — do not stop
   at the first one.** With the default window a slot may start at 7:00, 7:30, 8:00, 8:30, OR 9:00
   (start between 7-9am, ending 8:30-10:30am), so a venue with 7:00 fully booked can still qualify
   on a later start. The same applies to any user-given window. The
   only two reasons to stop early are:
   - You've **confirmed** a qualifying window (one is enough unless you want to list several), or
   - You've checked every start time in the start window that the Start Time dropdown actually
   offers. Concretely: look at which candidate times (e.g. 7:00/7:30/8:00/8:30/9:00) the dropdown lists (a time not listed
   can't be booked, so it's fair to skip *that specific time*), and verify the Court dropdown for
   each one that IS listed until you either find a slot or exhaust them. A common mistake is
   seeing "No Courts Available" at 7:00 and moving on while 8:00 or 8:30 was still an available,
   unchecked start — don't do that.

> **Known limitation (kept deliberately): area search reports the first window, not all windows.**
> In area-search mode (2A), a venue is marked as qualifying as soon as one start time is confirmed,
> and the remaining start times are *not* checked. So the Available Time Slots cell shows the
> earliest free window found, and there may be more free windows at that venue that weren't
> checked. (E.g. in the Oct 4, 2026 run, One 8, Verve and Games Period were free at 7:00 and
> never checked at 7:30-9:00.) This is a speed trade-off: checking every start time adds up to 4
> more booking-widget rounds per qualifying venue with the default window (more with a wider
> user-given window). Named-venue mode (2B) does check every start
> time. When presenting area-search results, don't imply the listed window is the only one.
> To change this, make area search follow the same "check every offered start time" rule as 2B.

If a venue's operating hours (shown on its detail page) don't cover any part of the start window,
skip the booking-widget check entirely and note it isn't open then (e.g. "Closed 7-9am").

**Fallback method — two single-hour checks (only if the Duration stepper misbehaves).** If a
venue's `+` button won't cooperate (e.g. it keeps cart-adding despite no court being selected, or
the control is unresponsive), you can confirm a 90-minute window another way: check the Court
dropdown at **1-hour** duration for both **T** and **T+30 min**, and if the *same named court*
appears in both, the window T to T+90 is genuinely free — a court with no conflict in [T, T+60)
and none in [T+30, T+90) can't have one anywhere in [T, T+90). Match by full court name (venues
often have Court 1, Court 2, ... of the same format). This needs both half-hour marks to exist as
bookable start times; for venues that only offer whole-hour starts (e.g. Games Period), check
**T** and **T+60 min** instead — same court free at both 1-hour slots means [T, T+120) is clear,
which contains the 90-minute window. Prefer the Duration method above; this is the backup.

**Interacting with the Court dropdown reliably:**
- It only populates on a genuine trusted click (via the `computer` tool), never a JS-dispatched
  one — same constraint as venue card navigation elsewhere in this skill.
- After changing Start Time, the Court dropdown can render empty on the very next open even when
  data exists — close it and reopen once more, or wait ~1s, before trusting an empty result as
  real. An empty result is only real if it persists after a retry.
- Before each click, re-verify field state with `get_page_text` (not just a screenshot) — the
  Start Time listbox sometimes stays visually open and overlaps the Court control, causing a click
  intended for one field to silently land on the other (e.g. it can reselect a different Start
  Time instead of opening Court). Confirm Start Time still shows what you expect before reading or
  clicking Court, and redo the step if it doesn't.
- **After selecting a start time, press `Escape` (the `computer` tool's `key` action) to close the
  Start Time listbox before touching Court.** This is far more reliable than trying to click a
  neutral area or the field header to dismiss it — those clicks keep landing on a time option and
  silently change the start time (the overlap is that aggressive). Escape closes the headless-ui
  listbox cleanly and leaves the selected time intact, so the Court button below is then a safe
  target. (Escape occasionally doesn't register on the first try — re-check with `get_page_text`
  and press it again if the time list is still open.) Only fall back to a `read_page`-based ref
  click on the Court button if Escape genuinely won't close the list.
- To read the Court options without any click ambiguity once it's open, `read_page` the Court
  `listbox` by its ref — its children give exact court names and prices, which is cleaner than
  parsing `get_page_text`.
- Clicking a court name sometimes *immediately* adds it to the cart (a single-click "quick add"
  some venues use) rather than just highlighting it for a duration change. That's harmless — no
  payment occurs — but a cart-add made at 1 Hr only proves the 1-hour slot; it doesn't extend to
  90 minutes by itself. This is why the primary Duration method sets the duration *before*
  opening Court and only *reads* the court list — never click a court name. If a venue still
  cart-adds or the stepper won't move to the needed duration, switch to the fallback
  two-single-hour-checks method above rather than trying to drive the stepper past a cart-add.

## 4. Gather Field Size and Cost together, from the price chart

Playo *does* expose real per-hour pricing and field size anonymously — it's just not on the
booking widget. It's in a **price chart modal** on the venue's own detail page (not the "Book Now"
booking widget page — go back to the plain venue page, e.g.
`https://playo.co/venues/bangalore/<venue-slug>`). Under "Sports Available" it says "(Click on
sports to view price chart)" — that's the way in.

1. On the venue detail page, find the sport tag element for Football: the leaf node whose text is
   exactly `Football` **and** whose closest ancestor has class `cursor-pointer` (there are several
   other "Football" text matches on the page — breadcrumbs, related-searches links, footer links —
   this scoped selector is what avoids clicking the wrong one). In JS:
   ```js
   const el = Array.from(document.querySelectorAll('*'))
     .find(e => e.children.length === 0 && e.textContent.trim() === 'Football' && e.closest('.cursor-pointer'));
   const card = el.closest('.cursor-pointer');
   ```
2. Click it with a proper synthetic pointer sequence — a plain `.click()` does not reliably reach
   this React handler:
   ```js
   function fireClick(el){
     const r = el.getBoundingClientRect();
     const cx = r.left + r.width/2, cy = r.top + r.height/2;
     for (const type of ['pointerdown','mousedown','pointerup','mouseup','click']) {
       el.dispatchEvent(new MouseEvent(type, {bubbles:true, cancelable:true, clientX:cx, clientY:cy, view:window}));
     }
   }
   fireClick(card);
   ```
3. Wait ~1-1.5s (`await new Promise(r=>setTimeout(r,1200))`) for the modal's price data to load —
   it fetches asynchronously and reads as an empty dialog if you check too early.
4. Read it: `document.querySelector('.fixed.inset-0.z-30, [role="dialog"]').innerText`.

The modal's text lists **every format the venue offers** (e.g. "Football 5 a side", "Football 7 a
side", "Football 9 a side", ...) each with its own day-banded hourly rate, e.g.:
```
Football 5 a side
Saturday - Sunday
INR 1500.0 / hour
06:00 AM - 10:00 PM
```
This gives you both columns at once — but **filter them by what step 3 found free**:
- **Field Size**:
  - **Venue has a qualifying slot** → list **only the formats that had a free court in the
    reported window(s)**, taken from the court names in the Court dropdown (e.g. courts "5 a side
    Pitch 3", "7 a side Pitch 2" → "5-a-side / 7-a-side"). Do **not** list formats the venue offers
    but that had no free court then — e.g. a venue whose price chart shows 5/7/9-a-side but whose
    free courts were only 5s and 7s is "5-a-side / 7-a-side", not "5/7/9-a-side". (An earlier run
    listed 9-a-side for Social Grid and 9/11-a-side for Depot18 when none were free — misleading.)
  - **Venue has no qualifying slot** (or wasn't checkable / not found / no football) → put "—" in
    **both** Field Size and Cost. Don't list the venue's formats or prices. Since nothing will be
    reported, you can skip the price chart entirely for such venues — do step 3 first, and only
    open the price chart for venues that have a slot.
- **Cost**: pick the rate band that covers the target date's day-of-week and the start window (if the window straddles two bands, report both, e.g. "₹1450/hr before 11am; ₹1600/hr after")
  (weekday bands are usually "Monday - Friday" or "Monday - Thursday"; a Sunday falls under
  "Saturday - Sunday" or "Friday - Sunday" depending on venue — match whatever weekday the
  resolved target date actually is, and use a "Holiday(s)" band only if you know the date is a
  public holiday). Report a rate for **each format listed in Field Size** — i.e. only the
  formats with a free court when the venue has a slot, all formats when it doesn't — e.g.
  "5-a-side: ₹1500/hr; 7-a-side: ₹3000/hr" rather than picking just one, since the user hasn't
  specified a squad size. Field Size and Cost must always cover the same formats — and both are
  "—" for a venue with no slot.

If the dialog reads empty after the wait, the click landed on the wrong element or the request is
still in flight — retry once with a fresh page load rather than reusing a stale page state (rapid
repeated interactions on the same loaded page have caused this to misfire in testing).

- **Location**: the locality shown on the venue card/detail page, presented as a **short,
  clickable Google Maps link** — not a long raw URL. Build the underlying search URL from the
  venue name + locality (`https://www.google.com/maps/search/<venue+name>+<locality>+Bangalore`),
  but display it behind short link text so the reader sees a place name, not a URL:
  - In the **chat table (step 5)**: markdown `[<locality (zone)>](<maps url>)`, e.g.
    `[Jayanagar (South)](https://www.google.com/maps/search/Games+Period+Jayanagar+Bangalore)`.
  - In the **Google Sheet (step 6)**: a `HYPERLINK` formula so the cell shows only the place name
    (see step 6 for the exact CSV escaping).

## 5. Output table

Present results in exactly this column order, matching the format the user expects:

| Venue Name | Field Size | Available Time Slots | Cost | Location |
|---|---|---|---|---|

- **Area search (2A):** one row per venue that has a qualifying slot — up to 5, fewer if the
  ~15-candidate cap was hit. Don't pad with empty rows; instead, list the checked-but-empty venues
  in a short line under the table so the user knows they were tried.
- **Named venues (2B):** one row for **every** named venue (and every matched listing), in the
  order the user named them (any row without a slot has "—" for Field Size and Cost) — including ones with no slot ("No 90-min slot starting 7-9am", using the actual duration and window),
  not found ("Not found on Playo"), or not bookable. Never drop a venue the user asked about.
- If a venue has multiple qualifying windows, list them comma-separated in one cell.

## 6. Save to a Google Sheet on Drive

Save the results as a Google Sheet in the Drive of **whoever is running the skill** (via their own
Drive connector), under a fixed name so repeat runs keep one current file.

**Settings.** Before this step, check for an optional local settings file next to this skill:
`.claude/skills/turf-finder/settings.local.md` (it is git-ignored, so it's personal to one install
and never published). It may contain:

```
sheet_name: <file title to use>
replace_without_asking: true | false
```

- `sheet_name` — the Drive file title. **Default if absent: `Turf Finder results`.**
- `replace_without_asking` — whether an existing file with that exact title may be replaced without
  a confirmation. **Default if absent: `false`** (ask first).

Below, **SHEET_NAME** means the resolved title.

This is a Google Drive connector's file tools (`create_file`, `search_files`, `trash_file`, etc.)
— they're deferred, so load them first with `ToolSearch({query: "select:<tool names>"})` if they
aren't already available in context.

1. **Look for an existing file with that exact title** via `search_files` with query
   `title = '<SHEET_NAME>'` (owned by the user: add `and owner = 'me'`). The name is fixed across
   runs (no date suffix), so a repeat run should replace the old one rather than pile up duplicates.
2. **If one is found, decide whether to replace it:**
   - `replace_without_asking: true` and exactly one match → trash it (`trash_file`) and continue.
   - Otherwise **ask the user first**, naming the file and linking it, e.g. "A file named 'Turf
     Finder results' already exists in your Drive — replace it (the old one goes to Drive trash),
     or save this as a new file?" If they say replace, trash it. If they say new file, use
     `SHEET_NAME (YYYY-MM-DD)` with the target date instead and don't trash anything.
   - More than one match → never trash any of them automatically; ask which (if any) to replace.
   Trash-and-recreate is needed because `update_file` only changes metadata (title/parent), it
   cannot replace a file's content. Trash is recoverable from Drive, but never trash a file the
   user hasn't agreed to replace (directly, or via the setting).
3. **Create the new file** with `create_file`: `title` = SHEET_NAME,
   `contentMimeType: "text/csv"`, and `textContent` built from the output table (step 5) as CSV —
   header row `Venue Name,Field Size,Available Time Slots,Cost,Location`, one data row per venue.
   Quote any cell containing a comma (the Cost column often has one, e.g.
   `"5-a-side: ₹1500/hr; 7-a-side: ₹3000/hr"` — use `;` instead of `,` to separate multiple rates
   within a cell so the CSV stays unambiguous). Leave `disableConversionToGoogleType` unset — the
   default behavior converts CSV to a native Google Sheet automatically, which is what's wanted.
   - **Location column = a short clickable map link, via a `HYPERLINK` formula** (not a raw URL).
     The cell content is `=HYPERLINK("<maps url>","<locality (zone)>")`. Because that contains both
     commas and double-quotes, wrap the whole cell in double-quotes and double every internal
     double-quote for CSV — the field becomes:
     `"=HYPERLINK(""https://www.google.com/maps/search/Games+Period+Jayanagar+Bangalore"",""Jayanagar (South)"")"`
     On CSV→Sheets conversion, cells beginning with `=` are evaluated as formulas, so the Location
     cell renders as the place name linked to the map. After creating, confirm it worked by reading
     the file back (`read_file_content`): the Location cells should show the locality text (e.g.
     "Jayanagar (South)"), NOT a literal `=HYPERLINK(...)` string — if you see the raw formula text,
     the conversion didn't evaluate it and you should investigate rather than leaving raw formulas.
4. Report back the `viewUrl` from the response as a clickable link so the user can open it directly.

Creating this file is a "regular" action (no explicit per-run confirmation needed): it's a
personal data file in the user's own Drive and isn't sent to anyone else. Replacing an existing
file is the only part that needs consent (step 2). If the connector isn't available at all in a given session, fall back to building a
local `.xlsx` (see the xlsx skill) and hand it to the user via file delivery instead, explaining
that direct Drive upload needs the connector.

## Notes on reliability

This relies on scraping Playo's live booking widget, which is a React SPA — clicking venue cards
requires a real pointer click via the browser tool's `computer`/`find` actions (a JS-dispatched
`.click()` does not trigger its router). If a venue's booking widget behaves unexpectedly (empty
listbox, page not loading), skip it, note it in the output, and move to the next candidate so one
flaky venue doesn't block the whole run.
