# ButterflyDreaming — Fractal (standalone)

One media module from [ButterflyDreaming](https://butterflydreaming.org),
running on its own. Open `index.html` — there is no build step, no bundler and
no server.

**[butterflydreaming.org](https://butterflydreaming.org)** · CC0

---

## What it plays

**The same machinery as the Kolam module, sonified.** An L-system is rewritten
and walked by a turtle — but instead of drawing, the turtle's height becomes
pitch. Where the kolam makes a figure, this makes a melody from the same kind of
string.

    %%bd_axiom X
    %%bd_rule X: XFYFX+F+YFXFY-F-XFYFX
    %%bd_rule Y: YFXFY-F-XFYFX+F+YFXFY
    %%bd_iterations 5
    %%bd_angle 90

**The rewriting.** `X` and `Y` are rewritten `iterations` times; only `F`, `+`
and `-` reach the turtle, so `X` and `Y` shape the string without ever being
heard.

**The walking, and the wall.** The turtle moves in two dimensions as before. Its
vertical position is read as pitch, quantised onto `scale` from `root`. When it
would run past the top or bottom of the range it **reflects** rather than
clipping — a pitch-reflection bounce, which is what keeps a long walk inside a
singing register instead of drifting away into inaudibility.

`angle` does here what it does in the kolam: it decides whether the walk lies on
a lattice. A lattice angle gives a melody that returns to its own notes; an
awkward one wanders.

Played through Tone.js with bass-recorder samples. **Press play before anything
else** — browsers, and iOS in particular, will not start audio without a
deliberate gesture.

---

## For a developer: what a module has to do

A ButterflyDreaming media module is **an iframe that speaks four messages**.
That is the entire contract.

| direction | message | meaning |
|---|---|---|
| module → host | `BD_READY` | loaded; send me a script |
| host → module | `bd_script_update` `{ script }` | the text to render |
| module → host | `bd_av_state` `{ text, fromDrift }` | its live script, on **every** render |
| module → host | `bd_module_log` `{ level, line }` | its console, so a host can see inside the iframe |

There is a fifth, optional in both directions: `bd_ui_config`
`{ hideControls, hostChrome }`, which lets a host say *I supply the controls
myself* and *I draw nothing around this iframe*. Both default to BD's own
behaviour, so a module that ignores the message still works everywhere — but
this page sends `hostChrome: false`, because BD reserves layout for furniture
that only BD stamps in, and a standalone that kept the reserve would be giving
away picture for nothing.

Answer `bd_script_update`, announce `bd_av_state`, and **any** ButterflyDreaming
host can drive your module — this page, or BD itself, or a viewer on another
device. Nothing else is required.

`index.html` is a complete host in about 145 lines of plain JavaScript —
88 of code and 46 of comment — written to be read. Two details in it are worth stealing:

- **Attach the message listener before setting the iframe's `src`.** The module
  announces `BD_READY` the moment it loads, and a listener added afterwards
  misses it.
- **`fromDrift` marks a frame the module's own timer caused**, not a person. A
  host that writes every frame into a text box will fight anyone typing in it.
  This page tracks those frames in a variable so **Copy** is never stale, while
  the box itself only updates on a human change.

### Deliberately absent: deep links

Earlier standalones packed the whole script into a URL. That meant compression,
a wire table of abbreviated keys, and a length ceiling to measure against — a
great deal of apparatus standing between a reader and how a module actually
works. It is gone. Copy the script and paste it wherever you like.

---

## The script format

Lines beginning `%%bd_` are directives; everything else is prose. A block
directive opens with `[` and closes on a line that is exactly `%%bd_]`.

    %%bd_module bd_M_Fractal
    %%bd_axiom X
    %%bd_rule X: XFYFX+F+YFXFY-F-XFYFX
    %%bd_p_iterations 5
    %%bd_angle 90
    %%bd_p_scale min_pentatonic

**`_p_` asks for a control.** `%%bd_p_iterations` means "give this one a
stepper"; `%%bd_iterations` means "use this value and offer no control" — which
is exactly what `%%bd_angle` above is doing. The mark is
presentation only — it never changes what a directive *means*, and a module
strips it before looking the value up.

Two rules a module must honour:

1. **The writer reconstructs the form it read.** A script carrying
   `%%bd_p_angle` must come back carrying `%%bd_p_angle`, or a value update
   would quietly change which controls appear.
2. **A script with no mark anywhere keeps every control.** Marking is opt-in, so
   nothing written before the convention existed changes behaviour.

---

## Keeping this copy honest

`music_module.html` is **vendored** — a copy of the module as it stands in
ButterflyDreaming's own repository. That is deliberate: a developer should be
able to open it, read it and break it without a server.

The cost of vendoring is drift, and it has bitten this project before — two
copies of an earlier module diverged and polish landed in only one of them. So
the copy is refreshed by one deliberate command rather than by hand:

    ./sync_from_bd.sh

It overwrites `music_module.html` from the BD working tree and records which
commit it came from in `MODULE_SOURCE.txt`. **If you have changed the module
here, that command will discard your changes** — it is a copy-down, not a merge.

---

## Picking up development

`AGENTS.md` in this repository is the working guide: what may and may not be
edited here, how to prepare a change for ButterflyDreaming, the BDX/AVX/RX
harness, and a list of developments worth trying. Start there.

---

## Licence

CC0. Do what you like with it.
