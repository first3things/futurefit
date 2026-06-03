# 3v3 Matchday Planner

A free, single-page tool for organising **FA FutureFit U7 3v3 football** matchdays.

Enter how many players each team brings and the planner instantly works out:

1. **How many pitches you need**, and
2. **Exactly how every game lines up** — which players are on each side, paired Home vs Away wherever possible.

It is built around the FutureFit principle that **everyone plays, with no substitutes**.

🔗 **Live tool:** https://first3things.github.io/futurefit/.
📐 **More on the format:** [A closer look at 3v3](https://futurefit.englandfootball.com/futurefit/a-closer-look-at-3v3/index.html)

---

## What it does

Grassroots organisers rarely get perfect numbers on a matchday. This tool takes the players who actually turn up for each team and arranges them into the best possible set of 3v3 games, following FA FutureFit guidance:

- **3 players per side maximum.**
- **No substitutes** — every player is on a pitch.
- **Home vs Away** fixtures wherever the numbers allow.
- Smaller or uneven games (2v2, 3v2, 2v1) are used only when needed, and flagged as "smaller games."
- **Superstar teams** (mixing both clubs) are created only when necessary to keep everyone playing.

It also shows the FutureFit **3v3 essentials**: pitch size, goal size, ball size, match length and recommended playing time.

## How the allocation works

The logic runs in two stages:

1. **Choose the games** — pick the set of game sizes (3v3, 3v2, 2v2, …) that uses every player, preferring full 3v3s, avoiding lone players (a team of 1 is only ever needed at totals of 2, 3 or 7), and avoiding lopsided games.
2. **Assign the clubs** — a small dynamic-programming pass colours each side Home or Away to **maximise genuine Home-vs-Away fixtures**, only falling back to same-club games or mixed "superstar" teams when the numbers force it.

The full rules and reference algorithm are documented in [`docs/pitch-allocation-spec.md`](./pitch-allocation-spec.md).

## Tech

- **Single self-contained HTML file** — no build step, no framework, no dependencies.
- Vanilla HTML, CSS and JavaScript.
- Loads Barlow / Barlow Condensed from Google Fonts.
- Accessible: associated labels, `aria-live` results, keyboard focus styles, WCAG AA contrast, `prefers-reduced-motion` support, and a print stylesheet.
- Styled to match the England Football FutureFit brand.

## Files

| File | Purpose |
|------|---------|
| `index.html` | The complete tool (rename of `3v3-matchday-planner.html`). |
| `pitch-allocation-spec.md` | Full build specification: rules, algorithm, test cases. |
| `README.md` | This file. |

## Accessibility & branding notes

- Home is shown in **red**, Away in **blue**, distinguished by both colour **and** a letter (H / A) so it never relies on colour alone.
- Player counts are capped at 200 to keep results instant.

## Credits

Built by [First Three Things](https://www.ftt.ai).
Based on [FA FutureFit](https://futurefit.englandfootball.com/futurefit/) principles for U7 football. Not an official FA product.

## License

**Copyright © 2026 First Three Things. All rights reserved.**

This project is source-available, not open source. The code is public for
transparency, but no permission is granted to use, copy, modify, or redistribute
it without prior written permission. See [`LICENSE`](./LICENSE) for the full
terms. For licensing enquiries, get in touch via [ftt.ai](https://www.ftt.ai).
