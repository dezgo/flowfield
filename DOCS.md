# Flow Field — Technical Documentation

A single-file, dependency-free generative-art toy. ~1,400 particles drift through
an invisible noise field leaving glowing trails, react to your mouse, detonate in
"big bangs", and respond live to music. Everything lives in `index.html` — HTML,
CSS, and vanilla JS, no build step.

- **Live:** https://flowfield.appfoundry.cc
- **Repo:** https://github.com/dezgo/flowfield

---

## Controls

| Input        | Effect                                                              |
|--------------|--------------------------------------------------------------------|
| Move mouse   | Steer particles — they spiral toward the cursor (vortex, not just pull) |
| Click        | Local particle burst at the cursor                                 |
| **B**        | Big bang — collapse to centre, white flash, shake, shockwave rings |
| **M**        | Mic mode — react to the room / an external speaker                 |
| **S**        | Capture this computer's audio directly (tab / system audio)        |
| **Space**    | Shift the colour palette                                           |

Left untouched it runs an automatic **breathing cycle**: drift → gravity collapse
→ big bang → repeat.

---

## How it works

### Rendering loop
A full-window `<canvas>` driven by `requestAnimationFrame`. Each frame:
1. Apply a decaying **screen-shake** offset to the canvas transform.
2. Advance the breathing-cycle state machine.
3. Paint a near-transparent dark rectangle over everything — this is what creates
   the **glowing trails** (old frames fade out gradually instead of being cleared).
4. Switch to `'lighter'` (additive) blending and draw every particle as a short
   line segment from its previous to current position.
5. Draw blooms (radial-gradient glows) and shockwave rings.

`DPR` **supersamples** at 2× the device pixel ratio (capped at 3): the canvas
renders more pixels than the screen has and the browser downscales, anti-aliasing
the thin filaments. (If it ever feels heavy, drop the multiplier toward 1.5.)

### The flow field
A small hand-rolled **value-noise** function (seeded, deterministic permutation
table) samples a smooth 2D field. Each particle reads the noise at its position,
turns that into an angle (`noise * 3π`), and accelerates along it. A slow time
term (`t * 0.0009`) drifts the field so the flow keeps evolving. Field "zoom" is
the `SCALE` constant.

### Particles
A fixed pool of `NUM` (1,400) particles, each with position, previous position,
velocity, and a life counter. Velocity is integrated with heavy damping
(`v = v*0.86 + a*0.14`) for smooth, flowing motion. When a particle dies or drifts
off-screen it respawns at a random spot. Colour is HSL, derived from speed +
x-position + the global palette `hue`, so faster particles glow brighter.

### Mouse interaction
Within a radius of the cursor, particles get a force that's part **attraction**
and part **tangential** (perpendicular) — the tangential component is what makes
them spiral into a vortex rather than just clumping.

### Big bang & breathing cycle
`bigBang()` slams every particle to centre with an outward radial velocity, flashes
the canvas white, sets `shake` and `bang` energy to 1, and spawns three staggered
shockwave rings. The `bang` value decays each frame; while it's high, the flow
field's influence is scaled down by `(1 - bang)` so raw explosion momentum
dominates first and the field smoothly reclaims control — no hard cutover.

The breathing cycle is a two-phase state machine (`'flow'` → `'collapse'`):
- **flow** for `FLOW_FRAMES` (~10s), then
- **collapse** — inward gravity (plus a swirl term) drags particles home, then
  auto-triggers a big bang, resetting to flow.

### Booms (the audio-reactive pops)
`boom(x, y, intensity)` fires at an **arbitrary location** and draws a glow
(`bloom`) plus a set of expanding shockwave rings, with the ring count and
wobbliness varying by a random style (clean rings vs. heavily-wobbled jagged
shards, etc.) and bigger hits adding an extra bold shock ring.

Rings aren't perfect circles — `makeRing()` precomputes a per-vertex **wobble**
array so each is drawn as a slightly organic closed path.

> **Important design rule — booms are pure overlay graphics.** A boom is *only*
> rings + glow. It must **never** touch the particles, shake the canvas, or shift
> the global `hue` — all of those make the swirly field jolt/recolour on the beat
> (the canvas-shake one is especially sneaky: it displaces trails mid-draw so they
> look like they shatter, while particle data stays clean). The field stays a calm,
> constant-speed twirl at all times; **booms float over it as the only thing that
> reacts to music.** See the audio section.

### Audio reactivity
Two capture sources feed one analyser pipeline (`startAnalyser`):
- **Mic (M):** `getUserMedia` with echo-cancellation / noise-suppression /
  auto-gain **off**, so the raw signal reaches the analyser.
- **System (S):** `getDisplayMedia({video, audio})` — a clean digital feed of a
  browser tab or the whole system. Works with headphones, no room noise.

Beat detection uses **spectral flux**: each frame it sums the *positive* changes
across a broad band (~47 Hz–10 kHz) vs the previous frame. This catches percussive
onsets from any source — even tinny phone speakers that produce almost no bass.
(An earlier bass-only version missed phone-speaker music for exactly that reason.)

Two gates stop false triggers and tune the feel:
- **Loudness gate:** an adaptively-tracked `ambient` floor (falls fast toward
  quiet, rises slowly) — a beat only counts if the sound is clearly above it, so
  a silent room stays calm.
- **Contrast threshold:** flux must spike to ≥ 1.6× its recent baseline, and a
  cooldown caps the rate (~4 booms/sec) so it accents strong beats instead of
  firing on every transient.

**Boom size scales with contrast.** `intensity = (flux / baseline)` shaped by a
power curve (capped at 3.5): a minor tick → a small boom (smaller glow, fewer
rings), a big hit → a much larger one (bigger glow, an extra bold shock ring).
Contrast scales the *boom*, never the field.

**The field is fully decoupled from the music — by design.** Every musical beat
calls only `boom()`; nothing in the audio path writes `shake`, `hue`, `bang`,
particle velocity, line width, or flow speed. `bigBang()` (the full particle-
flinging explosion) is reachable **only** from the **B** key and the idle breathing
cycle — never from a beat. So the swirly field looks identical whether music is
playing or not; only the ring/glow booms come and go. (This took some doing — see
the design rule under *Booms*.)

---

## Tuning knobs

All in `index.html`, mostly near the top of the script or in `readAudio()`:

| Constant / expr        | What it controls                                  |
|------------------------|---------------------------------------------------|
| `NUM`                  | Particle count                                    |
| `SCALE`                | Flow-field zoom (smaller = larger, sweeping flow) |
| `SPEED`                | Base particle speed                               |
| `FLOW_FRAMES`          | Seconds of drift before the auto-collapse         |
| `flux > fluxAvg * 1.6` | Beat selectivity (higher = only the punchiest)    |
| `beatCooldown = 14`    | Min frames between booms (higher = calmer)        |
| `ambient + 0.025`      | Loudness gate margin (lower = more sensitive)     |
| `Math.min(3.5, …)`     | Max boom size (the contrast→size curve)           |

---

## Hosting & deployment

- **Server:** a DigitalOcean droplet, nginx serving the static file from
  `/var/www/flowfield/`.
- **TLS:** Let's Encrypt cert via certbot (`--nginx`), fronted by Cloudflare
  (proxied / orange-cloud). The origin A record points at the droplet; it was
  temporarily set to "DNS only" to let certbot's HTTP-01 challenge through, then
  flipped back to Proxied.
- **CI/CD:** `.github/workflows/deploy.yml` runs on every push to `main` —
  checks out the repo, loads an SSH deploy key from repo secrets, and `rsync`s the
  files to the droplet. Updates go live in ~10–15s. Cloudflare serves the HTML as
  `DYNAMIC` (uncached), so pushes appear immediately with no cache purge.

Secrets used by the workflow: `DEPLOY_HOST` (droplet IP — **not** the
Cloudflare-proxied hostname, which won't route SSH), `DEPLOY_USER`, `DEPLOY_SSH_KEY`.

---

## File layout

```
flowfield/
├── index.html                 # the entire app (HTML + CSS + JS)
├── README.md                  # quick start + controls
├── DOCS.md                    # this file
└── .github/workflows/deploy.yml   # CI: rsync to the droplet on push
```
