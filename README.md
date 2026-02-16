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

5. Quantize $D_t$ to the grid above.

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



### (3) Groove pattern mode (16-step rhythms)

* A 16-step hit/rest pattern (clave/tresillo/etc) decides whether each step is a hit.
* Step duration depends on Helicity (scaled) and Rate.
* If pattern says rest → it forces note-off and advances time.
* If hit → plays a note (pitch uses a walk generator in this mode).



### (4) Custom mode (your 8-step pattern)

* There are 8 toggles to decide hit/rest.
* Step duration is the C STEP (same as STEP) times RateFactor (probably need rething this).
* Rest/hit behavior same as Groove mode.

---

## Performance controls

### Gravity

Gravity applies a contextual pull back toward tonic, stronger when you drift high.

Weight multiplier:
$w \leftarrow w \cdot \frac{1}{1 + g_\text{eff} d^2}$

where

* $d = |n - {\rm root}|/12$ (octave distance)
* $g_\text{eff}$ grows with register, so high notes pull back more

Intuition is that the **melody can roam, but it “comes home.”**

### Contour

Contour biases upward vs downward steps.

* If candidate pitch > current pitch → multiply by `contourUp`
* If candidate pitch < current pitch → multiply by `contourDown`

So we get a “steering wheel” feel such that if we turn Contour up → more rising lines
and if we turn it down → more descending lines

### Phrase Length

Phrase forces a cadence to tonic every $L$ notes.

* Phrase OFF → no cadence
* Phrase = 8 → every 8 events, next note forced to tonic

This is a way to enforce more “musical sentences.” Not always effective, especially in the AR mode I feel.



