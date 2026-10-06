# Samodiva

A chess engine by **Nikolay Kralev**.

Binaries and network files only — the source code is proprietary and is not
part of this repository.

---

## Quick start

Download **`binaries/1.5-nnue/Samodiva-1.5-x64.zip`** (1.1 MB), unpack it
anywhere, and run `samodiva_tb.exe`. It contains everything needed:

```
samodiva_tb.exe     the engine (UCI), with Syzygy tablebase support
samodiva.nnue       the evaluation network — keep it in the same folder
READ ME FIRST.txt   short instructions
```

No configuration is needed: the engine finds `samodiva.nnue` next to itself and
loads it automatically. Without that file it falls back to the classical
evaluation and plays about 250 Elo weaker.

---

## What is in here

| Path | Contents |
|---|---|
| `binaries/1.5-nnue/Samodiva-1.5-x64.zip` | **ready-to-use pack, 1.1 MB** — engine + network + readme |
| `binaries/1.0` … `binaries/1.4` | Windows builds of every released version |
| `binaries/1.4/samodiva_tuned.exe` | 1.4 with the tuned evaluation (prebuilt) |
| `binaries/1.5-nnue/` | the current version, unpacked: engine, tablebase build and the network |
| `nets/` | the other networks, published for research |
| `docs/` | measured strength results |

`samodiva.exe` is the plain build, `samodiva_tb.exe` the build with Syzygy
tablebase support (it needs a tablebase path, see below).

### Runs on any machine

Every binary except `samodiva_tuned.exe` is compiled for the plain **x86-64
baseline** instead of the CPU of the build machine, so it also starts on older
processors and on CPUs without AVX2, where a `-march=native` build dies with an
illegal instruction. The price is a few percent of speed on a modern CPU.

### The network belongs to version 1.5 only

`samodiva.nnue` is a **correction to the evaluation of version 1.5** (and of
1.3, whose evaluation is identical). It was trained against exactly that
evaluation, therefore:

* **1.5** — keep `samodiva.nnue` next to the executable. It is already in the
  download and the engine loads it automatically; nothing to configure.
* **1.0 … 1.4** — **do not copy the network there.** Those versions have their
  own (older or tuned) evaluation and the corrections do not apply to them.
  Measured cost of forcing it onto 1.4: about 115 Elo.

---

## Current version: 1.5 (NNUE)

Version 1.5 adds a neural network on top of the hand written evaluation. The
network is loaded from the file named in the `NNUEFile` option and the engine
works out from the file itself what kind of network it is.

### Options added in 1.5

| Option | Type | Default | Meaning |
|---|---|---|---|
| `NNUEFile` | string | `samodiva.nnue` | network to load |
| `Use NNUE` | check | true | master switch |
| `NNUE Clamp` | spin | 50 | how far the network may pull the evaluation away from the classical one |
| `NNUE Residual` | check | true | the network is a correction to the classical evaluation |

`NNUE Residual` and `NNUE Clamp` are detected from the network file itself, so
loading an older absolute network switches the engine into plain
"network only" mode automatically.

### Tablebases (1.3 and later)

```
setoption name SyzygyPath value E:\path\to\syzygy
```

The support is compiled into `samodiva_tb.exe`; the plain `samodiva.exe` has no
Syzygy options at all. Tablebases are not part of the network file.

---

## Measured strength

Measured with fastchess, 10s + 0.1s per move, 64 MB hash, single thread,
opening book of 25 standard openings. Full details in
[docs/VALIDATION.md](docs/VALIDATION.md).

| Match | Games | Score | Elo |
|---|---|---|---|
| **1.5 vs 1.3 — SPRT, alpha = beta = 0.05, H1 accepted** | **563** | **63.5 %** | **+96 ± 21** |
| 1.5 vs 1.3 (confirmation run, default options) | 100 | 61.5 % | +81 ± 48 |
| 1.5 + tablebases vs 1.3 + tablebases | 100 | 58.5 % | +60 ± 48 |
| 1.3 vs Houdini 1.5a | 100 | 22.5 % | −215 ± 68 |
| **1.5 vs Houdini 1.5a** | 100 | **34.0 %** | **−115 ± 60** |

Against an external engine the network is worth **+100 Elo**
(Houdini 1.5a: 22.5 % → 34.0 % when the network is enabled).

The engine is strongly time-control dependent:

| Time control | 1.5 vs Houdini 1.5a |
|---|---|
| 10s + 0.1s | −115 Elo |
| 300s per move | +97 Elo (11 games) |

Its time management spends about 1/40 of the remaining clock per move, so a
300 second budget becomes roughly 7.5 seconds of thinking per move.

Version 1.5 passes a 54 test functional suite covering UCI compliance, perft,
mate handling, incremental accumulator verification and Syzygy probing.

---

## Networks

| File | Type | Notes |
|---|---|---|
| **`binaries/1.5-nnue/samodiva.nnue`** | **residual, 768-256** | **the shipped network, +96 Elo — use this one** |
| `samodiva_v3_rr50.nnue` | residual, 768-256 | identical backup of the above |
| `nets/samodiva_v1.nnue` | absolute, 768-256 | first generation, weaker |
| `nets/samodiva_v2.nnue` | absolute, 768-256 | second generation, weaker |
| `nets/samodiva_v6q.nnue` | absolute, 768-256 | trained with decisive positions kept |
| `nets/samodiva_v7.nnue` | absolute, 768-384 | squared activations, mg/eg heads |
| `nets/samodiva_v7b.nnue` | absolute, 768-384 | as above plus endgame positions |
| `nets/samodiva_v7r.nnue` | residual, 768-384 | as above, trained as a correction |

The absolute networks are published for research. None of them is stronger
than the shipped residual network: a network that replaces the classical
evaluation completely loses accuracy on endgame positions, which is where most
of the search tree lives (measured: 25–38 % sign agreement there versus 58 % for
the hand written evaluation). The residual design — a bounded correction on top
of the hand written evaluation — is what makes 1.5 strong.

Network files carry a four byte header that identifies their type, so the engine
never guesses wrong.

---

## Running

Samodiva speaks UCI.

```
cd binaries\1.5-nnue
samodiva_tb.exe
```

```
uci
isready
position startpos
go movetime 10000
```

Any UCI-compatible GUI (Arena, CuteChess, en-croissant, Banksia) works.

---

## License

See [LICENSE](LICENSE). The binaries and networks may be used and redistributed
under the MIT terms; the source code is proprietary and is not distributed here.
