# Flow Field

An interactive generative-art toy: ~1,400 particles drift through an invisible,
slowly-evolving noise field, leaving glowing additive trails. It's a single
self-contained HTML file — no build step, no dependencies.

## Run it

Open `index.html` in any modern browser. That's it.

For the mic feature, some browsers block microphone access on `file://` URLs.
If so, serve it locally:

```
python -m http.server 8000
# then open http://localhost:8000
```

## Controls

| Input        | Effect                                            |
|--------------|---------------------------------------------------|
| Move mouse   | Steer the particles — they swirl toward the cursor |
| Click        | Local burst of particles                          |
| **B**        | Big bang — collapse, flash, shake, shockwave rings |
| **M**        | Music mode — react to ambient audio via the mic   |
| **Space**    | Shift the colour palette                          |

Left to itself it runs an automatic "breathing" cycle: drift → gravity collapse
→ big bang → repeat. In music mode the beats drive it instead — bass drops
trigger bangs and overall loudness makes the whole field surge.

Press **S** to react to this computer's audio directly (tab / system audio) —
cleaner than the mic and works with headphones.

## More

See [DOCS.md](DOCS.md) for how it all works — the flow field, particle system,
big-bang breathing cycle, audio/beat detection, tuning knobs, and the
deployment / CI setup.
