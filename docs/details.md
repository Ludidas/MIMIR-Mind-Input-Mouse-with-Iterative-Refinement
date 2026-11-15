# MIMIR: details

Reference notes behind the [main README](../README.md): full dataset results,
what has and hasn't been tested, design trade-offs and tuning.

## Scoring the decoder

From `Unicorn-quadclick/`:

```
python evaluate_dataset.py      # prints the tables below, saves results.png
python tests/test_core.py
```

## Dataset results

g.tec SSVEP dataset: 2 subjects × 2 runs × 20 trials, four targets (9, 10, 12,
15 Hz), 8 electrodes at the back of the head. The idle baseline for each run is
taken from the subject's *other* run.

Live decision rule replayed over every trial (40 trials per row):

| Channels | Subject | Right | Wrong | No answer | Time to answer |
|---|---|---|---|---|---|
| All 8 | 1 | 40 | 0 | 0 | 2.4 s |
| All 8 | 2 | 36 | 2 | 2 | 3.9 s |
| 3 the Unicorn also has (PO7, Oz, PO8) | 1 | 40 | 0 | 0 | 2.3 s |
| 3 the Unicorn also has | 2 | 22 | 1 | 17 | 4.4 s |

- False selections while the subject rested: 0–2 per ~140 s.
- Fixed 3 s window, all 8 channels: plain CCA 85% → idle-corrected 95%
  (subject 2: 73% → 90%). g.tec's own reference result for subject 2 is about
  85% with a trained classifier.
- Simulated full click: 3 selections (≈64 buttons, element mode) take about
  10 s for subject 1 and 15 s for subject 2. A pixel grid on a full-HD screen
  needs 5 selections, 17 s and 24 s.

Some people give a strong signal and some a weak one. With only three
back-of-head electrodes the weak subject mostly gets "no answer" rather than
wrong answers. Calibrate each person and demo with the strongest.

Caveats: two subjects and 80 trials is a small sample, and the two thresholds
were picked after looking at this same data, so expect live numbers to be a
bit worse.

## What has and hasn't been tested

Tested (headless Linux):

- Decoder on the dataset (numbers above).
- Whole loop: dataset → engine → UDP → overlay → clicks on demo buttons.
  Subject 1: 10/10 targets hit, 3 selections each, no undo needed.
- The g.Pype pipeline with g.Pype's generator, and with dataset data fed
  through `gp.CsvReader` into `DecoderNode` (picked 12, 9, 15, 10 Hz correctly).
- Jaw-clench detector on synthetic bursts; zero false clenches across all four
  real recordings.

Not tested yet, check these first:

- **The Unicorn itself.** `gp.HybridBlack` builds and connects, but there was
  no headset. Confirm the data is in microvolts and the channel order is
  Fz C3 Cz C4 Pz PO7 Oz PO8.
- **Real jaw clenches.** The dataset has none. Watch the scope, clench, and
  adjust `CLENCH["k"]` in `bci/config.py` if needed.
- **Windows-only parts:** real-button detection (UI Automation), the screen
  capture used for enlarging, double and right click, pop-up menu detection,
  screen scaling (125%/150%), and `0_stop_all.bat`. The see-through layer,
  hotkeys and left click were reported working on one PC. The logic around all
  of them was tested with the Windows calls swapped out.
- **Typing** into a real Windows program. Windows only reports classic text
  fields as text boxes; for anything else (such as a document's main editing
  area) use `T` to open the keyboard for whatever has the cursor.
- **Screen flicker vs. LEDs.** The dataset used LEDs. A monitor gives a weaker
  signal, and small regions deep in the split weaker still. If deep levels are
  unreliable, raise `--fill`, or make each level zoom to full screen.

## Two choices or four

`B` (or `--choices 2`) switches to two boxes per step, flickering at 9 and
15 Hz. The boxes are bigger and there is less to tell apart, but a click takes
more picks: with 40 buttons, about 8 against 4 (both including the click-type
question). With two choices the keyboard takes 5 to 6 picks per letter.

On the dataset every pair of speeds did about as well as four-way picking for
the strong subject; for the weak subject, pairs gave 0–1 wrong picks in 20
trials against 2 in 40 with four choices. That's too few trials to call a
difference. Try both on each person.

## Box sizes

By default every box at a level is the same size (`--split equal`). A button
belongs to the box that holds its middle, and boxes with no buttons don't
flicker. `--split balanced` cuts between buttons instead: about half a pick
fewer per click, but the boxes come out uneven and smaller.

| Buttons on screen | Equal: picks to reach one button | Balanced |
|---|---|---|
| 20 | 2.8 | 2.4 |
| 40 | 3.4 | 3.0 |
| 64 | 3.8 | 3.0 |

(Splitting only, measured before the enlarged last step and the click-type
question were added.)

Something sitting on the line between two boxes can be picked through either
box. A first-step box where nothing was detected carries on as a plain pixel
grid.

## Flicker styles

`C` cycles, or pass `--stim`:

- `checker` (default): a checkered circle in the middle of each region, light
  and dark squares swapping places so brightness never changes.
- `circle`: a plain flickering circle.
- `full`: the whole region flickers.

At the last step, where each box sits around a single icon, the whole small
box flickers regardless, so you look straight at the icon.

## Tuning

Settings live in `Unicorn-quadclick/bci/config.py`.

- More wrong picks than you'd like → raise `thr_score` / `thr_margin`.
- Too slow or too many "no answer" → lower them, or lengthen calibration.
- US mains is 60 Hz (`MAINS_HZ`); the dataset replay uses 50 Hz by itself.
