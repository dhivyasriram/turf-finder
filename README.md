# Turf Finder

A [Claude Code](https://claude.com/claude-code) skill that finds bookable football turf slots in
Bangalore on [Playo](https://playo.co). It checks real court availability for your date and time,
then gives you a table of venues, field sizes, free time slots, prices and map links. It also saves
the table to a Google Sheet.

## Usage

Run `/turf-finder` in Claude Code. You can add any of three inputs, in any order and phrasing:

| Input | Examples | If you leave it out |
|---|---|---|
| Date | `Oct 11`, `6th october`, `this Saturday`, `tomorrow` | The upcoming Sunday |
| Venue(s) | `Tackle Jayanagar`, `tackle and games period`, `social grid` | Searches South/Central Bangalore for the top 5 venues |
| Start-time range | `9am to 11am`, `6-8pm`, `after 7pm` | 7-9am on Saturdays and Sundays. On a weekday, the skill asks you for one. |

Examples:

```
/turf-finder
/turf-finder 9am to 11am
/turf-finder 3rd october, tackle jayanagar and games period
/turf-finder 6th october, social grid, 8pm to 10pm
```

By default the skill looks for a continuous 90-minute game. To change that, add a length such as
`for 2 hours`. A time range means the window the game can *start* in. For example, `9am to 11am`
allows starts from 9:00 to 11:00.

## Output

| Venue Name | Field Size | Available Time Slots | Cost | Location |
|---|---|---|---|---|

- **Named venues:** every venue you named gets a row, even if it has no free slot.
- **Area search:** you get only venues with a free slot. The venues that were checked but had
  nothing free are listed under the table.
- **Field Size and Cost:** these cover only the field sizes that actually had a free court in that
  window. They show "—" when a venue has no slot.
- **Google Sheet:** the results are also saved to **your own** Google Drive, through your Drive
  connector, as a sheet named **"Turf Finder results"**. If a file with that name already exists,
  the skill asks before replacing it. Without a Drive connector, you get a local `.xlsx` instead.

### Optional settings

Create `.claude/skills/turf-finder/settings.local.md`. It's git-ignored, so your settings stay on
your machine:

```
sheet_name: My turf slots
replace_without_asking: true
```

- `sheet_name` — the Drive file name to use. Default: `Turf Finder results`.
- `replace_without_asking` — set to `true` to skip the "replace existing file?" question. Default:
  `false`.

## How it checks availability

Playo's start-time list isn't reliable on its own. The skill instead sets the real duration in
Playo's booking form and reads which courts the form offers for that whole block, which is the only
signal that matches actual bookability.

## Requirements

- Claude Code with a browser tool (the built-in browser or Claude in Chrome), for browsing Playo.
- Optional: a Google Drive connector, for saving the results sheet.
- Opus is recommended. Judging real 90-minute availability is easy to get wrong.

## Install

Copy the `.claude/skills/turf-finder` folder into one of these:

- your project's `.claude/skills/`, to use it in that project only
- `~/.claude/skills/`, to use it in every project

## Known limitations

- In area search, each venue shows only the **first** free window found, to keep runs fast.
  Named-venue searches check every start time.
- The skill depends on Playo's website layout, so changes to the site can break it.
