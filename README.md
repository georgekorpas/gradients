# gradients
A midi generative sequencer for the Electra One MKII controller based on a variety of ideas.



## Big picture

This sequencer has **two independent parts**:

1. **When** to trigger events (rhythm / duration / rests)
2. **What pitch** to play on a hit event (pitch generation + scale/mask + weighting)

There are 4 modes controlling the *rhythm driver*:

* **AutoRegressive** (AR): duration depends on previous pitch; pitch depends on previous duration (this is massively inspired by the beautiful AR sequencer of SDKC Instruments Hellical) 
* **Fixed step**: constant step from STEP selector (very simple constant 16th note sequence)
* **Groove pattern**: 16-step hit/rest pattern (inspired partially by Noise Engineering's Zularic Repetitor)
* **Custom**: your 8-step hit/rest pattern (you still do not decide the pitch)

On top of any mode there is an optional **RESTS** layer that can skip hits probabilistically. I implemented this because of my failure to always produce musical sequences. 
I would love to know how SDKC has done it but for the moment it is what it is.

Finally, the pitch generator is always scale-aware and mask-aware, and in AR mode it’s coupled to durations.

---

## Timebase and quantization

Everything runs in MIDI clock ticks at 24 PPQN with either of:

* internal clock: `timer.setBpm(tempo * 24)`
* external clock: `transport.onClock()` is called once per incoming MIDI clock tick

Therefore:

* `24 ticks = 1 quarter note`
* `12 ticks = 1/8`
* `6 ticks = 1/16`
* `3 ticks = 1/32`

### AR rhythmic grid (important)

AR mode never outputs arbitrary tick lengths. It snaps to a fixed **grid**.

In the current version, the AR grid is:

`{ 6, 8, 9, 12, 16, 18, 24, 36, 48 }`

Meaning the fastest AR value is 6 ticks = 1/16 (tried with less ticks but I was getting these "surprise" 1/32 bursts which were not musical to my ears).

---

## The `MODES`

### (1) AutoRegressive mode

The idea is that of a two-way coupling:

* previous **pitch → next duration**
* previous **duration → next pitch**

For the duration update (pitch → duration) we have:

1. Compute a continuous duration target (ticks):

* extract degree index in the selected scale:
  `deg = degreeIndexFromPitch(prevPitch)` (1..N)
* extract register above root:
  `reg = floor((prevPitch - rootNote)/12)` clipped

2. Build a metric:
   $\text{metric} = \sqrt{\text{deg}} + 0.9\cdot \text{reg}$

3. Apply Helicity and Rate:
   $\tilde{D}_t = 4.0\cdot \text{helicScale}(h)\cdot \text{metric}\cdot \text{RateFactor}$

Helicity mapping is chosent to be exponential:
$\text{helicScale}(h) = 0.4\cdot 7^{h/127}$

4. Add mild autoregressive “duration feedback” (so it doesn’t lock to constant 16ths):
   $\tilde{D}_t \leftarrow \tilde{D}*t\cdot \Big( 0.90 + 0.25\cdot \mathrm{clip}(D*{t-1}/24,0,1.5) \Big)$

5. Quantize:
   $D_t = \text{QuantizeToGrid}(\tilde{D}_t)$
   with the grid above.

So AR timing is quantized and truly depends on the previous pitch.

For the pitch update (duration → pitch) we have:

The pitch is sampled from the allowed note set (scale + OCT CUT + SPREAD + mask), using weighted roulette.

Weights include:

* proximity to the “target index”: implied by the previous duration (duration→pitch coupling)
* jump penalty: discouraging huge leaps
* recent-note penalty: discouraging short loops
* gravity: pull toward tonic
* contour: prefer rising/falling motion

Then a note is emitted unless the RESTS layer cancels it.


### (2) Fixed step mode

* Step duration is taken from STEP selector (`1/4, 1/8, 1/16, …`)
* Duration is multiplied by `RateFactor`
* On every step, either:

  * play a note (pitch selected by the pitch generator), or
  * rest (RESTS layer)

No AR coupling in timing (only pitch selection uses previous duration if you keep it; but the main “autoregressive feel” is in AR mode).

---

### 3) Groove pattern mode (16-step rhythms)

* A 16-step hit/rest pattern (clave/tresillo/etc) decides whether each step is a hit.
* Step duration depends on Helicity (scaled) and Rate.
* If pattern says rest → it forces note-off and advances time.
* If hit → plays a note (pitch uses a walk generator in this mode).

---

### 4) Custom mode (your 8-step pattern)

* Your 8 toggles decide hit/rest.
* Step duration is your C STEP (same as STEP) times RateFactor.
* Rest/hit behavior same as Groove mode.

---

## The new performance controls

### Gravity

Gravity applies a contextual pull back toward tonic, stronger when you drift high.

Weight multiplier:
[
w \leftarrow w \cdot \frac{1}{1 + g_\text{eff} d^2}
]
where

* (d = |n - root|/12) (octave distance)
* (g_\text{eff}) grows with register, so high notes pull back more

Intuition: **melody can roam, but it “comes home.”**

### Contour

Contour biases upward vs downward steps.

* If candidate pitch > current pitch → multiply by `contourUp`
* If candidate pitch < current pitch → multiply by `contourDown`

So you get a “steering wheel” feel:

* turn Contour up → more rising lines
* turn it down → more descending lines

### Phrase Length

Phrase forces a cadence to tonic every L notes.

* Phrase OFF → no cadence
* Phrase = 8 → every 8 events, next note forced to tonic

This is very effective for “musical sentences.”

---

## How to test Gravity / Contour / Phrase (fast + reliable)

### Before starting (common setup)

Use settings that make behavior obvious:

* MODE: **AutoRegressive**
* RESTS: **Off**
* ROOT EMPH: **Off** (temporarily) — so Gravity’s effect is not masked by tonic weighting
* OCT CUT: 3–4 oct
* SPREAD: medium/high
* Rate: 1 (neutral)
* Mask: enable the scale normally (don’t remove many notes)

#### Optional: make it measurable

In your DAW, record MIDI for ~30 seconds and inspect the piano roll.

---

### A) Test Gravity

**Goal:** with high Gravity, pitches should spend more time near tonic, and return faster after climbing.

1. Set **Gravity = 0** (full left)

2. Let it run 20–30 seconds

   * Note the register drift: it should wander more widely.

3. Set **Gravity = max** (full right)

4. Let it run again
   Expected:

   * fewer sustained high-register runs
   * more frequent returns toward tonic region
   * smaller “average pitch” over time

**Quick quantitative check (in piano roll):**

* Compare mean MIDI note number (average pitch) across the two recordings.
  Gravity high should reduce the mean and the variance.

---

### B) Test Contour

**Goal:** Contour should bias direction of motion.

1. Set **Contour = center** (neutral) → baseline

2. Set **Contour high** (full right)
   Expected:

   * more upward steps than downward steps
   * rising phrases more common

3. Set **Contour low** (full left)
   Expected:

   * more downward steps than upward steps
   * falling phrases more common

**Quick quantitative check (in MIDI):**
Count how many note-to-note intervals are positive vs negative.

* Contour high → %positive > %negative
* Contour low → %negative > %positive

---

### C) Test Phrase Length

**Goal:** phrase cadence should be unmistakable.

1. Set Phrase = **Off**

2. Listen: no periodic “home punctuation”

3. Set Phrase = **8**

4. Listen: you should hear an obvious “return to tonic” punctuation roughly every 8 events.

**Very obvious test:**

* Temporarily set mask so only **tonic + 5th + 2nd** are enabled.
* Phrase = 8 will sound like a clear cadence back to tonic.

If you don’t hear it, it means the cadence trigger isn’t firing — but in v59_FIXED it should.

---

## If something feels subtle

Gravity and Root Emph both pull “home”, but they do it differently:

* **Root Emph** = “tonic is always more likely, regardless of context”
* **Gravity** = “the further you drift, the stronger the pull back”

So if Root Emph is high, Gravity can feel less dramatic. For testing: set Root Emph OFF.

---

If you want, I can add a tiny “debug” overlay in the info bar (only when you hold a button) to show live values like current reg, chosen dur ticks, phrase counter. That makes verifying behavior even easier without touching MIDI routing.
