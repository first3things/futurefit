# Matchday Planner — Build Specification

A specification for an online tool that takes the number of players from two
teams (Home and Away) and returns (1) how many pitches are needed and (2) the
exact line-up of every game.

Based on FA FutureFit principles for U7 football: **3v3 maximum, no substitutes,
everyone plays all the time.**

> This document describes the current, game-based model with Home-vs-Away
> pairing. It supersedes all earlier versions of the spec.

---

## 1. Inputs

| Field | Type | Constraints |
|-------|------|-------------|
| `home` | integer | 0–200 |
| `away` | integer | 0–200 |

Derived: `total = home + away`.

A meaningful result requires `total >= 2`. If `total < 2`, show a friendly
"add at least 2 players" prompt.

Values should be clamped on input: no negatives, no leading zeros, capped at a
sensible maximum (200) to keep the search instant.

---

## 2. Fixed parameters

| Constant | Value | Meaning |
|----------|-------|---------|
| `MAX_PER_SIDE` | 3 | A team has at most 3 players. |
| `MIN_PER_SIDE` | 1 | A team may be as small as 1, but only when unavoidable (see §5). |
| `TEAMS_PER_PITCH` | 2 | One pitch hosts one game of two teams. |

No substitutes: every player is assigned to a team, and every team is on a
pitch. Nobody sits out.

---

## 3. Core concepts

The model is built around **games**, not individual teams. A **game** occupies
one pitch and is defined by the sizes of its two sides.

**Allowed game types** (by side sizes), with the number of players each uses:

| Game | Players | Notes |
|------|---------|-------|
| 3v3 | 6 | Full game — the ideal. |
| 3v2 | 5 | Uneven. |
| 2v2 | 4 | Even but not full. |
| 3v1 | 4 | Uneven; lopsided — avoided by the imbalance rule. |
| 2v1 | 3 | Contains a team of 1; only when unavoidable (see §5). |
| 1v1 | 2 | Only permitted when it is the entire plan (see §5). |

A **team** within a game is 1, 2 or 3 players. Each team is coloured by club:

- **Pure team** — all Home, or all Away.
- **Superstar team** — contains players from both clubs. With only two clubs,
  a superstar team is always a 2+1 mix (never 1+1+1).

A **game type** by club pairing:

- **Home-vs-Away fixture** — one side is entirely Home and the other entirely
  Away. This is the desired outcome for every pitch.
- **Same-club game** — both sides are pure but from the same club (Home vs Home,
  or Away vs Away). Allowed only when forced (see §6).
- A game containing a superstar team is neither a clean fixture nor a same-club
  game; it is a fixture-with-a-mix and is preferred over a same-club game.

Number of pitches = number of games. Because every game has exactly two teams,
the total number of teams is always even and every team always has an opponent
(no byes during rotation).

---

## 4. Hard rules (never violated)

1. **Everyone plays.** All `total` players are placed; nobody sits out.
2. **Max 3 per side**, min 1 per side.
3. **Every game has two teams** (so teams always pair into pitches with no byes).
4. **A 1v1 game is only allowed when `total == 2`.** For any larger total, a
   better arrangement must be found instead (see §5).

---

## 5. Stage A — choosing the games

The plan is built in two stages. **Stage A** decides the set of games (the side
sizes). **Stage B** (§6) assigns clubs to the sides.

Among all valid sets of games whose sizes sum to `total`, pick the best by the
following criteria, applied in strict priority order (criterion 0 is a hard
filter; the rest are lexicographic tie-breakers, each minimised).

**0. No stray 1v1 (hard filter).**
Discard any candidate that contains a 1v1 game unless `total == 2`.

**1. Fewest teams of size 1.**
Minimise the number of one-player teams across the whole plan. A lone player is
treated as worse than an uneven game, so 13 players becomes `3v2 + 2v2 + 2v2`
(no teams of 1) rather than `3v3 + 2v2 + 2v1`. A team of 1 is only unavoidable
at totals **2, 3 and 7** — these are the only totals that cannot be built purely
from sides of size 2 and 3.

**2. Fewest non-full games.**
Prefer plans with the most 3v3 games; minimise the count of games that aren't
3v3.

**3. Least lopsidedness (imbalance).**
Among the remaining uneven games, minimise the total gap between the two sides
(`sum of |sideA − sideB|`). This makes the tool prefer **2v2 over 3v1** for four
leftover players — both are "non-full," but 3v1 is lopsided.

**4. Least total deficiency.**
Minimise the total shortfall from full games (`sum of (6 − players_in_game)`),
nudging toward fuller games overall.

**5. Fewest superstar (mixed) teams** *(as estimated for Stage A feasibility)*.
A small DP checks the minimum mixed teams needed to realise the candidate with
exactly `home` Home players; candidates that can't reach `home` are infeasible.
(The definitive club assignment, including same-club vs fixture trade-offs, is
made in Stage B.)

**6. Fewest pitches.**
Final compactness tie-breaker.

---

## 6. Stage B — pairing teams onto pitches (Home vs Away)

Once the set of games is chosen, assign club colours to each side so the
matchday plays as real fixtures.

**Goal, in priority order:**
1. **Fewest same-club games.** A Home-vs-Away fixture is always preferred to a
   Home-vs-Home or Away-vs-Away game. Same-club games should occur only when the
   numbers force them — for example when only one club is present, or a club's
   surplus genuinely cannot be paired against the other club.
2. **Fewest superstar (mixed) teams.** Among assignments with the same number of
   same-club games, use the fewest mixed teams. Note the ordering: a clean
   fixture that requires one extra superstar team is preferred over introducing a
   same-club game. (So, e.g., 21 Home / 14 Away may use two superstar teams to
   keep every pitch a Home-vs-Away fixture, rather than one superstar team plus a
   Home-vs-Home game.)
3. **Most fixtures.** Final tie-breaker, maximising clean Home-vs-Away pitches.

**Why a simple "flip whole sides" approach is wrong.**
An earlier implementation tentatively set each game as Home-vs-Away, then
rebalanced the Home total by flipping entire sides between clubs. That cannot
express mixed splits such as `3 Home v 3 Away` + `2 Home v 3 Away`, so it
collapsed into same-club games (the 5 Home / 6 Away bug). The correct approach
assigns each game's two sides **independently**.

**Procedure (reference approach): dynamic programming over games.**
For each game with side sizes `(s0, s1)`, enumerate every filling
`side0 = (h0 Home, a0 Away)`, `side1 = (h1 Home, a1 Away)` where
`h0 + a0 = s0` and `h1 + a1 = s1`. For each filling record:
- `homeUsed = h0 + h1`
- `mixedTeams` = number of sides with both clubs present (0, 1 or 2)
- `isFixture` = one side pure-Home and the other pure-Away
- `sameClub` = 1 if `mixedTeams == 0` and not a fixture (both sides pure, same
  club), else 0

Then run a DP over the games, state = total Home players used so far, choosing
one filling per game, minimising the cost vector
`[sameClubGames, mixedTeams, −fixtures]` lexicographically, and requiring the
final Home total to equal `home` exactly.

**Display normalisation.** For each game, show the more-Home side (higher
`h − a`) on the left, so Home is consistently on the left.

---

## 7. Outputs

1. **Pitches needed** = number of games.
2. **Per-pitch line-up** — for each pitch: the format (e.g. "3v3"), and the two
   sides with their player make-up (how many Home, how many Away).
3. **Summary counts** — total players, number of superstar teams, and how many
   games are not full 3v3.
4. **"Smaller game" flag** — any pitch that is a 1v1 or 2v1 is marked as
   playable-but-not-ideal (visually distinct, plus a text label — not colour
   alone).

---

## 8. Reference algorithm (summary)

```
function solve(home, away):            # Stage A — choose games
    total = home + away
    if total < 2: return null
    candidates = all multisets of game-types
                 {3v3:6, 3v2:5, 2v2:4, 3v1:4, 2v1:3, 1v1:2}
                 whose player-sizes sum to total
    best = null
    for each candidate:
        if candidate has a 1v1 AND total > 2: skip          # §5.0

        oneTeams   = number of teams of size 1
        uneven     = games that aren't 3v3
        imbalance  = sum over games of |sideA - sideB|
        deficiency = sum over games of (6 - players_in_game)
        mixedMin   = minMixedTeams(team sizes, home)         # DP; skip if infeasible
        npitches   = number of games

        score = [oneTeams, uneven, imbalance, deficiency, mixedMin, npitches]
        keep candidate with lexicographically smallest score
    return best


function buildPitches(home, away):     # Stage B — assign clubs to sides
    best  = solve(home, away)
    games = expand best into a list of [largerSide, smallerSide]

    for each game (s0, s1):
        enumerate all fillings (h0,a0,h1,a1) with h0+a0=s0, h1+a1=s1
        annotate each with homeUsed, mixedTeams, isFixture, sameClub

    DP over games, state = Home players used so far:
        choose one filling per game,
        minimise [sameClubGames, mixedTeams, -fixtures] (lexicographic),
        require final Home total == home

    if no exact-home assignment exists:
        fall back to side0=all-Home / side1=all-Away per game   # guard; rare

    for each game: put the more-Home side (higher h - a) on the left
    return games with per-side {home count, away count, size}, plus superstar count
```

`minMixedTeams` is a DP over the candidate's team sizes returning the minimum
number of mixed teams needed to use exactly `home` Home players; infeasible if
`home` is unreachable.

### Edge cases

| Situation | Behaviour |
|-----------|-----------|
| `total < 2` | Show "add at least 2 players." |
| `total == 2` | The only case where 1v1 is allowed. With one club (2/0) the single game is Home vs Home. |
| `total` in {3, 7} | A team of 1 is unavoidable (e.g. 2v1). |
| One club is 0 | All games are same-club (e.g. all Home vs Home); no fixtures and no superstar teams are possible. |
| Unequal clubs | The DP keeps every pitch a Home-vs-Away fixture where possible, using superstar teams rather than same-club games; same-club games appear only when truly forced. |

---

## 9. Worked examples (use as automated tests)

H = pure Home side, A = pure Away side, M = superstar (mixed) side. Format
notation like `H3vA2` means a 3-player Home side versus a 2-player Away side.

| Home | Away | Total | Pitches | Games |
|------|------|-------|---------|-------|
| 1 | 1 | 2 | 1 | H1vA1 |
| 2 | 0 | 2 | 1 | H1vH1 (single club) |
| 2 | 1 | 3 | 1 | H2vA1 (smaller game) |
| 3 | 1 | 4 | 1 | H2vM2 |
| 4 | 3 | 7 | 2 | H2vA2, H2vA1 |
| 5 | 4 | 9 | 2 | H3vA2, H2vA2 |
| 5 | 6 | 11 | 2 | H3vA3, H2vA3 |
| 6 | 5 | 11 | 2 | H3vA3, H3vA2 |
| 7 | 6 | 13 | 3 | H3vA2, H2vA2, H2vA2 |
| 9 | 9 | 18 | 3 | 3 × H3vA3 |
| 12 | 12 | 24 | 4 | 4 × H3vA3 |
| 16 | 14 | 30 | 5 | 4 × H3vA3 + 1 × H3vM3 |
| 18 | 14 | 32 | 6 | 3 × H3vA3, H3vM3, 2 × H2vA2 |
| 21 | 14 | 35 | 6 | 4 × H3vA3, H3vM3, H2vM3 |
| 8 | 0 | 8 | 2 | 2 × H2vH2 (single club) |
| 30 | 0 | 30 | 5 | 5 × H3vH3 (single club) |
| 100 | 97 | 197 | 33 | 32 × H3vA3 + 1 × H2vM3 |

Invariants any implementation must satisfy, checkable on every input:
- **Conservation:** sum of all Home counts == `home`; sum of all Away == `away`.
- **No-1v1 rule:** no 1v1 game unless `total == 2`.
- **Team-of-1 rule:** no team of size 1 unless `total` ∈ {2, 3, 7}.
- **Pairing:** the number of same-club games is the minimum the numbers allow;
  with two non-empty clubs of equal-ish size, every pitch is a Home-vs-Away
  fixture.
- **Display:** the more-Home side is always on the left.

---

## 10. UI / accessibility requirements

- Two number steppers (Home, Away). A clear heading above them
  ("How many players are here?") and each input labelled with its unit
  ("players"). Labels associated with inputs via `for`/`id`.
- Live recalculation on change; results in an `aria-live="polite"` region.
- Headline hero showing the pitch count, plus a "3v3 essentials" facts strip
  (pitch 15×10m up to 20×15m, goals 120×75cm / 4×2.5ft, size-3 ball, no
  goalkeepers, 6–10 min games on a carousel, 30–40 min playing time per player).
- Each pitch shown as a card: pitch number, format, and the two sides as player
  markers — Home and Away distinguished by colour **and** letter (H / A), not
  colour alone.
- Superstar teams clearly tagged.
- "Smaller game" (1v1/2v1) flagged with a text label and a distinct (non-colour-
  only) treatment.
- All text meets WCAG AA contrast (4.5:1 body; large display text may rely on
  the AA-large threshold — verify coloured header bars and red-on-light text).
- Respect `prefers-reduced-motion`.
- Print stylesheet: drop heavy backgrounds, keep team colours, prevent pitch
  cards breaking across pages.
- Brand: England red as the dominant colour (hero, section headings, top band),
  navy as the structural/support colour. Bold flat panels (solid borders, no
  drop shadows). Pitch motif concentrated in the hero, not across the page.
- Short standing reminder: *no substitutes — everyone plays; smaller and uneven
  games are fine.*
