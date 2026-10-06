# AGENTS.md — picking up development here

*For an AI agent, or a person arriving cold. Named `AGENTS.md` because that is
the filename the common coding agents look for without being told.*

**Read §1 before editing anything.** It is the one rule in this file that will
cost you work if you skip it.

---

## 1. The module file is VENDORED. Do not edit it here.

`music_module.html` is a **copy**. The original lives in the ButterflyDreaming
working tree at `M_Fractal/music_module.html`, and the copy here is refreshed by:

    ./sync_from_bd.sh

That is a **copy-down, not a merge**. It overwrites `music_module.html` and
discards anything you changed in it. `MODULE_SOURCE.txt` records which BD commit
the current copy came from.

So the workflow for a module change is:

1. edit `M_Fractal/music_module.html` in the BD repo,
2. commit and push there,
3. `./sync_from_bd.sh` here,
4. commit the refreshed copy here.

**What you MAY edit freely here:** `index.html` (the host page), `README.md`,
this file, `sync_from_bd.sh`. Those are this repository's own.

This is not bureaucracy. Two copies of an earlier module diverged in exactly
this way and polish landed in only one of them; the project has paid for the
lesson twice.

---

## 2. What this repo is

    index.html          the host page — ~145 lines of plain JS (88 code, 46 comment)
    music_module.html   the module, vendored (see §1)
    bass-recorder/      the sampler's four notes — part of the MODULE (see §8)
    README.md           what the module does and how the figure is made
    sync_from_bd.sh     refresh the vendored copy from BD
    MODULE_SOURCE.txt   which BD commit that copy came from

No build step, no bundler, no server, no dependencies to install. **Open
`index.html` in a browser.** If you want a server for cache reasons,
`python3 -m http.server` in this directory is enough.

Published at <https://wrcstewart.github.io/ButterflyDreaming-Standalone-Fractal/> from `main`, root path.
Pushing to `main` redeploys; allow a minute or so.

---

## 3. The contract, and where it lives in the code

A ButterflyDreaming media module is **an iframe that speaks a handful of
`postMessage`s**. That is the whole integration surface — there is no SDK and
nothing to import.

| direction | message | meaning |
|---|---|---|
| module → host | `BD_READY` | loaded; send me a script |
| host → module | `bd_script_update` `{ script }` | the text to render |
| module → host | `bd_av_state` `{ text, fromDrift }` | its live script, on **every** render |
| module → host | `bd_module_log` `{ level, line }` | its console, forwarded out of the iframe |
| host → module | `bd_ui_config` `{ hideControls, hostChrome }` | optional; see §4 |

Two details in `index.html` are load-bearing and easy to undo by accident:

- **The message listener is attached BEFORE `frame.src` is set.** The module
  announces `BD_READY` the moment it loads; a listener added afterwards misses
  it and the page sits there empty. The `frame.src = ...` line is deliberately
  the last statement in the file.
- **`fromDrift` marks a frame a timer produced, not a person.** The host keeps
  a `live` variable that follows every frame, but only writes the textarea on
  non-drift frames — otherwise it fights anyone typing and destroys the undo
  history. **Copy reads `live`, not the textarea**, so it is never stale.

---

## 4. `bd_ui_config` — two flags that are NOT the same flag

    hideControls   the HOST supplies the stepper column; hide mine
    hostChrome     the host draws things AROUND this iframe (default TRUE)

BD reserves layout for furniture only BD draws — it stamps its arrows and Extension panel into this module's two dock slots, for which the panel grid holds a whole area each. A host
that stamps nothing into that reserve gets it as lost picture.

These were one flag until 2026-09-30 and had to be split, because the two cases
pull apart exactly here:

| | hideControls | hostChrome |
|---|---|---|
| BD itself | false | true |
| an Ancillary Viewer | **true** | false |
| **this page** | **false** | **false** |

A standalone wants the chrome released and the steppers **kept**. That
combination was unreachable before the split. `hostChrome` defaults to `true`
so a host that says nothing keeps BD's behaviour; `hideControls` implies it.

---

## 5. The script format, and the two rules that bind a renderer

Lines beginning `%%bd_` are directives; everything else is prose. A block
directive opens with `[` and closes on a line that is exactly `%%bd_]`.

**`_p_` asks for a control.** `%%bd_p_angle` means *give this one a stepper*;
`%%bd_angle` means *use this value and offer no control*. The mark is
**presentation only** — it never changes what a directive means, and a module
strips it before looking the value up.

- **RULE 1 — the writer reconstructs the form it read.** A script that arrived
  carrying `%%bd_p_angle` must be written back carrying `%%bd_p_angle`. Get
  this wrong and merely moving a stepper silently changes which controls exist.
- **RULE 2 — a script with no mark anywhere keeps every control.** Marking is
  opt-in, so nothing written before the convention changes behaviour.
- **RULE 9 — a module's default script carries no directive it cannot act on.**
  A directive in a default script is a promise the module keeps.

The full set (RULES 1–9) is in BD's `CollagePlanStarted_2026-09-22.md`.

---

## 6. Preparing something for ButterflyDreaming

If you build a new module, or change this one in a way BD must see, this is the
checklist. **A new module needs FOUR registries and nobody has ever remembered
all four unprompted.**

### 6.1 The four registries (all in the BD repo)

| what | where |
|---|---|
| `MODULES` — embedded + standalone URLs | `viewer.js` (~line 2783) |
| `AV_VIEWER_MODULES` — which modules have a viewer page | `viewer.js` (~line 13297) |
| `AV_RENDERERS` — module id → renderer path | `AV/kolam.html` (~line 149) |
| `express.static` route — `/bd_X/` → the source dir | `server.js` (~lines 163–205) |

Note the route name and the directory name differ on purpose: `/bd_M_ABC/`
serves `./M_Music/`. Do not "fix" that.

### 6.2 The database side

Memgraph, reached through `node bd_tool.js` at the BD repo root.

    node bd_tool.js cypher "MATCH (n) WHERE n.name STARTS WITH 'bd_V_Kolam3D' RETURN n.name, n.seq, n.hasModuleScript"

A module occupies three kinds of node: a **Cluster**, a **gateway** TextNode
carrying `seq = -1`, and one or more **content** TextNodes carrying
`hasModuleScript = '<module id>'` and `seq >= 1`. The content node's text IS the
default script.

**Every DB-mutating `bd_tool.js` subcommand takes a pre-flight backup by
default. Do not pass `--no-backup`.**

**Editing stored text does not reach a running BD.** Node text loads with the
graph at boot, so re-tapping a node shows the stale copy. After a DB edit, say
so: *reload BD*.

### 6.3 Cache-busters and canaries

- `AV/kolam.html` loads renderers as `AV_RENDERERS[m] + '?v=NN'`. **Bump `NN`
  on any change to a module.** iOS Safari caches an iframe `src` hard, and a
  stale renderer means layout fixes silently never arrive.
- BD, the AV and `sr_editor` each host their **own** canary — a visible border
  rotating red → green → blue on every change to a cached file. They are
  independent; one says nothing about another. **Media modules have no canary
  host**, so a module-only change rotates nothing — but it still needs the
  `?v=` bump above.

### 6.4 Two-phase deploy

**Receivers before writers.** A renderer must be able to READ a new form before
anything WRITES one. A new module ships as a receiver from day one.

---

## 7. BDX / AVX / RX — the next stage

There is a second, independent harness, and it is where this repo's ideas grow
up. Repo <https://github.com/wrcstewart/bdx-demo>, local checkout
`~/bdx_demo`, pages <https://wrcstewart.github.io/bdx-demo/>.

| | what it is | where it runs |
|---|---|---|
| **BDX** | the controller — script panel, steppers, renderer | GitHub Pages (static) |
| **AVX** | the viewer — renders what it is told, no controls | GitHub Pages (static) |
| **RX** | the relay — a rendezvous for two browsers on different devices | a Node host |

RX is **live** at <https://rx.virtualfictions.uk/health>, on the existing
Discourse VPS behind a Cloudflare tunnel.

**The claim it makes:** the module architecture needs nothing of BD — no
Memgraph, no corpus, no graph, no pairing, no curation, no speech. And the
dependency ladder is worth stating precisely, because it is easy to overclaim
in either direction:

1. **Same machine → no server at all.** If the controller *opened* the viewer it
   holds a window handle, and `postMessage` reaches it, cross-origin included.
   Measured at **~1 ms, against ~30 ms through a socket**.
2. **Cross-device → a rendezvous is unavoidable.** Two browsers on two devices
   cannot reach each other; no window handle exists. That is the *only* reason
   the socket path exists in BD at all.
3. **Whose rendezvous is a free choice.** Run RX yourself (~10 lines around
   `bd_relay.js`, no account), or point at ours.

**How this page relates to it.** This repo is a BDX with the relay left out:
script panel, steppers, renderer, on a URL, no corpus. The next step for any of
these four pages is the same one — **add a View button** that opens an AVX and
drives it. Same machine needs no relay at all (tier 1), which makes it a genuinely
small change: claim the window *inside the click* (see §8), then `postMessage`
the script to it on every `bd_av_state`.

`AV/bd_av_client.js` in the BD repo (414 lines) is the viewer shim and **is the
third-party contract** — the artifact someone else would use.

---

## 8. Traps already paid for

Each of these cost a debugging round. They are not hypothetical.

- **A gesture does not survive an `await`.** `window.open` and clipboard writes
  are both refused after one. Claim the resource *inside* the click — open
  `about:blank` first and navigate later; hand the clipboard the *promise*.
  Safari refuses what Chrome allows. `index.html`'s Copy button has a
  select-the-text fallback for exactly this.
- **A missing asset fails silently.** The sampler loads four `.mp3`s from
  `./bass-recorder/` relative to the module. Vendoring copied the `.html` and
  not the directory, all four 404'd, and **nothing raised an error**:
  `Tone.loaded()` simply never resolved, `samplerReady` stayed false, Play and
  Stop never left their disabled state — while Save midi and Copy .abc, which
  need only the script text, lit up normally. It reads as a broken audio
  library and was a missing directory. `sync_from_bd.sh` now copies the samples
  too. **Some buttons live means the script arrived and the audio chain did
  not.**
- **Copying a file copies its claims.** Three of these four hosts were spliced
  from the fourth, and shipped six comments true only of the origin — one of
  which was a layout bug wearing a comment's clothes. After generating siblings
  from a template, read each against *the thing it now describes*. Grep the
  copies for the origin's proper nouns.
- **`node --check` parses as CommonJS** and silently passes ES-module errors.
  For an HTML file, extract the `<script>` body and check that.
- **A check whose filter excludes the failing pattern proves nothing.** Said
  after a "clean" grep reported four broken call sites as fine.
- **Test the path, not a stage of it.** A regex bug survived a test that ran on
  raw text and so never reached the normaliser that caused it.
- **iOS inputs below 16px auto-zoom on focus.** Every focusable input needs
  `font-size: 16px` or larger.

---

## 9. Developments worth trying

Ordered roughly by ratio of interest to effort. None is started.

### Separate grammars for pitch and rhythm  *(the one to do first)*

**Today one walk produces both, and they cannot be told apart.** Read
`turtleWalk` and the code below it: a single 2D turtle emits segments, each
tagged `isHorizontal`. Then **horizontal runs become notes** (duration = run
length) and **vertical movement sets pitch**. Melody and rhythm are two
projections of one walk, so you cannot vary one without the other. Every
rhythmic idea you have is also a melodic idea, whether you wanted it or not.

The module is already **two-layered** and that is the hook: `X` and `Y` are
rewriting-only — they shape the string and never reach the turtle, which sees
only `F`, `+` and `-`. Adding a third class of symbol that the *duration* reader
consumes and the *pitch* reader ignores (or the reverse) would give two grammars
running in one string — the same move the Kolam3D repo's `AGENTS.md` proposes
for pitch and angle, where `+` likewise drives two things at once. The plumbing already exists; what is missing is a second consumer.

### The reflection wall is the interesting object — expose it

When the walk would run past the top or bottom of the range it **reflects**
rather than clipping. That is what keeps a long walk singing instead of drifting
out of audibility, and it is the single most musically consequential rule in the
file. It is currently implicit. Directives for the bounds — and for whether the
reflection is elastic or hard — would turn an internal safety mechanism into an
instrument.

### Smoothly altering the grammar

The steppers move continuously; **the grammar does not move at all**. Editing a
rule is a discrete jump, and `iterations` is worse — growth is exponential per
pass, which is why `MAX_SYMBOLS_TRAVERSED` exists. Between iteration 5 and 6
there is nothing.

**And `iterations` is worse still than "discrete" — it is barely a control at
all.** Measured 2026-10-06, counting *distinct figures* across the stepper's
5–20 range: **4** under the Peano rules this module used to ship with, **3**
under the current ones. Iterations 6 and 8 are byte-identical under both,
because the curve is self-similar and the skip lands on a self-similar boundary
*by construction*. The fractal is **one endless string**; depth only ever
extends it, and the only audible axis is where you begin.

**There is already a mechanism here that knows where it is in the string**, and
it is the hook. `sharedOpening()` measures how much iteration N repeats of N-1,
and `expandAndWalk` traverses that prefix without storing it, keeping only what
follows. That is a positional cursor into the rewritten string — and since
2026-10-06 it is a cheap one, because the prefix costs time rather than memory.

**`start_at` is BUILT (2026-10-06)** and is the control to reach for. A
`%%bd_p_start_at` stepper giving the offset into the rewritten string in
**thousands of symbols** — `2.5` means 2,500 — added to the shared opening so
`0` is the behaviour from before it existed, and clamped to 25% of the
iteration's real length with the clamp reported rather than silent. Measured:
**twelve settings from 0 to 177k give twelve distinct figures**, against three
for `iterations` across its whole 5–20 range.

**What is left of the idea**: `iterations` is now nearly redundant — it only
decides how long the string is, which is to say how far `start_at` may reach.
Deriving it (expand to whatever depth covers `start_at` + the kept window) and
taking it off the panel is the tidy ending, and is NOT done, because removing a
control from scripts people have already saved is not reversible. That is a
decision for the author, not a refactor to slip in.

Everything below now has somewhere to stand.

**Fractional iterations**, once `start_at` exists.
`%%bd_iterations 5.4` = iteration 5 with 40% of its
symbols rewritten one more pass. Deterministic, reproducible from the script,
no new directive vocabulary, and it makes `iterations` a stepper you can hold
down. Two selection orders are worth building and comparing, because they will
not *sound* alike: **prefix** (the first 40% of symbols) elaborates the opening
and leaves the ending plain — a melody that grows more intricate as it starts
over; **interleaved** (every 5th of every 2) thickens the whole piece evenly.
For music the prefix order may well be the interesting one, which is the
opposite of what it would be for a drawing.

**Weighted productions** are the obvious alternative and carry a real cost:
`lindenmayer` supports stochastic successors, so a slider could move probability
mass from rule A to rule B — but **the script is the source of truth**, and the
same script must give the same piece on two devices. That needs a seed
directive, and a seed is a thing a user has to understand. Do the fractional
version first.

### A grammar editor rather than a textarea

The rules are free text with **no feedback until it plays**. A malformed rule is
indistinguishable from a boring one, and here the failure is especially quiet —
a grammar that blows the emission cap gets silently truncated and the status
line is the only sign. An inspector beside the box — non-terminal count, string
length at each iteration, where the cap will bite — would turn blind editing
into informed editing, and it needs no audio at all.

### What already works, and is worth knowing before you build it

**Fractal already emits ABC.** `midiToAbc` builds a real score, and the
**Copy abc** button produces a fully-formed `bd_M_ABC` script. So the pipeline
*generate here → hand-edit in the ABC module → play* exists **today**, across
the two standalone pages, via the clipboard. Do not rebuild it. Do consider
making it visible: neither page tells you the other one is there.


---

## 10. Where the rest of the knowledge is

The BD repo is the source of truth for everything above.

| | |
|---|---|
| `DOCS_INDEX.md` | what each of ~40 docs *is* |
| `PLANNING_REGISTER.md` | how far each design is *built*, evidence-based, ending in every unbuilt item in one table |
| `CHANGELOG.md` | newest-first narrative log; the friendly read |
| `CollagePlanStarted_2026-09-22.md` | the `%%bd_p_` convention and RULES 1–9 |
| `BDX_DEMO_PLAN.md` | §7 above, in full |
| `AV/README.md` | the Ancillary Viewer, and the 1 ms vs 30 ms measurement |

**Start with the two indexes.** They exist so that an arriving agent does not
have to grep.

### Known inconsistency, not yet fixed

BD's `MODULES` table still points `standalone` at the **old, retired** repos
(`bd_V_Kolam`, `bd_M_ABC`, `bd_M_Fractal` — the `preview.html` pages), not at
the four `ButterflyDreaming-Standalone-*` repos these files live in. Kolam3D has
no `standalone` entry at all. Updating that table is a small, safe change that
nobody has made yet.

---

*Licence CC0. Do what you like with it.*
