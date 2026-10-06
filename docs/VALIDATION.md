# Validation

Everything below was measured on the development machine (Windows, Ryzen 5 2600,
GCC 15.2) with fastchess. No numbers are estimates unless marked as such.

## Test conditions

| Setting | Value |
|---|---|
| Time control | 10s + 0.1s per move (20s per move for the clamp sweep in §4) |
| Hash | 64 MB |
| Threads | 1 |
| Opening book | 25 standard openings, 6 plies, both colours |
| Rules | draw by 40 moves / 8 moves within 10 cp, max 250 moves |
| Seed | fixed (1234567890987654321) |

## 1. Functional test suite

54 automated tests pass on the current build: UCI handshake and options,
`position`/`go` handling, perft, mate scores, insufficient material, threefold
repetition, fifty-move rule, incremental NNUE accumulator verification against a
full recomputation, and Syzygy probing (tablebase load, DTZ root probe, probe
counting in `tbhits`).

## 2. Network integrity

| Check | Result |
|---|---|
| engine integer math vs independent Python reference | 0 mismatches / 400 positions |
| trainer features → file → engine round trip | 0 mismatches / 200 positions |
| effect of int16 quantisation on validation loss | none measurable (< 0.01 %) |

## 3. SPRT: 1.5 versus 1.3

Sequential probability ratio test, elo0 = 0, elo1 = +5, α = β = 0.05,
normalised model.

| | |
|---|---|
| Games | **563** |
| Score | **63.5 %** (224 W / 72 L / 267 D) |
| Elo | **+96 ± 21** (nElo +139) |
| LLR | **+2.96**, bound ±2.94 → **H1 accepted** |
| Draw ratio | 47.3 % |
| Duration | 58 min |

An independent 100-game confirmation run using only the engine's default
options gave 61.5 % (+81 ± 48, LOS 99.97 %).

## 4. Dependence on the clamp (how far the network may move the evaluation)

100 games per configuration, 10s + 0.1s:

| `NNUE Clamp` | Score | Draw ratio |
|---|---|---|
| 25 | 51.5 % | 63 % |
| **50** | **60.5 %** | 52 % |
| 100 | 52.0 % | 36 % |
| 300 | 44.0 % | 17 % |

Repeated at 20s per move (100 games each) the optimum did not move:
R=50 → 62.0 %, R=150 → 61.0 %, R=300 → 43.5 %.

The bound is a safety mechanism, not a handicap: without it the network's
errors on positions unlike its training data get amplified by the search and
the engine resigns hopeless positions.

## 5. External reference (Houdini 1.5a)

| Engine | Games | Score | Elo |
|---|---|---|---|
| Houdini 1.5a vs 1.3 (no network) | 100 | 77.5 % | +215 ± 68 |
| Houdini 1.5a vs **1.5** (network) | 100 | 66.0 % | +115 ± 60 |

Enabling the network is worth **+100 Elo** measured against an engine that has
nothing to do with this project. Both engines had tablebases in this test.

## 6. Time control sensitivity

| Time control | 1.5 vs Houdini 1.5a |
|---|---|
| 10s + 0.1s | −115 Elo (100 games, fastchess) |
| 300s per move | +97 Elo (11 games, Arena) |

## 7. The network is tied to the evaluation it was trained against

The network is a correction on top of a specific hand written evaluation — the
one in 1.5 and 1.3, which are identical in that respect. Applying it to a
version with a different evaluation was measured and is a clear loss:

| Match | Games | Score | Elo |
|---|---|---|---|
| 1.4 + network vs 1.4 plain | 100 | 34.0 % | −115 ± 51 |

This is why the download contains the network only in `binaries/1.5-nnue/`.

## 8. What the network does and does not do

Measured on 3,195 unique positions taken from real games, comparing each
evaluation with the search score of the opponent engine:

| Position class | classical eval | 1.5 (network) |
|---|---|---|
| clearly winning (classical ≥ +800) | 84.9 % | 84.9 % |
| moderate advantage (+300…+800) | 82.5 % | 82.5 % |
| roughly equal (\|eval\| ≤ 50) | 56.0 % | 53.2 % |
| moderate disadvantage (−800…−300) | 64.7 % | 64.7 % |

The network never changes the sign of a decided position; it refines
evaluations around equality, which is where the hand written evaluation is
weakest.

For reference, three generations of *absolute* networks (replacing the
classical evaluation entirely, the newest with a 768-384 squared/mg-eg
architecture) were trained and are published in `nets/`. They score well on the
distribution they were trained on (74 % sign agreement) but fall to 25–38 % on
sparse endgame positions, where most search nodes live. In a 100-game match the
best of them scored 1 % against 1.3. They are included for research, not for
play.

Two data problems were identified on the way and are worth recording:

* the Lichess evaluation database is a *calibration* set dominated by decided
  positions — a network trained on it fits it well (74 % sign agreement) and
  then collapses on real game positions (50 %);
* harvesting positions with fewer than ten pieces (the great majority of search
  nodes) is mandatory: without them a network reaches only 25 % agreement in
  endgames, i.e. worse than a coin flip.
