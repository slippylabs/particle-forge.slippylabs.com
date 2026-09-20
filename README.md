# Particle Forge

Design a particle emitter and watch it — bursts and streams, gravity, drag, colour and size over lifetime, additive blending — then export the config plus drop-in JavaScript or C#. Runs entirely in your browser.

**Live:** <https://particle-forge.slippylabs.com/>

## What it does

- Eight presets — sparks, explosion, smoke, fire, magic, confetti, rain, trail — and 28 dials.
- Burst or continuous emission, with a repeat interval, a duration and a particle cap.
- Gravity in both axes, linear drag, colour/size/alpha over lifetime with an adjustable curve, spin, four shapes and additive or normal blending.
- A scrub bar, because any moment in the effect is as cheap to compute as any other.
- Click the stage to move the emitter. Export JSON, JavaScript or C#.

## How it works

The usual way to write one of these is to keep an array of particles and nudge each of them every frame. It works, and it has two problems that only show up later: the result depends on the frame rate, and it cannot be rewound.

This one is built the other way round. Nothing is stored between frames. **`systemAt(config, t)` returns the particles alive at time t**, worked out from scratch, and the renderer just calls it with the current time. Particle *n* is emitted at a time that depends only on *n*, and each particle draws its randoms from its **own stream keyed on its index** — a single shared stream consumed in frame order is precisely what makes an effect look different at 144 Hz.

The motion is a closed form. With linear drag the velocity obeys `dv/dt = g − k·v`, so

    v(t) = (v₀ − g/k)e^(−kt) + g/k
    p(t) = p₀ + (v₀ − g/k)(1 − e^(−kt))/k + (g/k)t

and at k = 0 it degenerates to the parabola. Both branches are written out, because the k → 0 limit of the first is 0/0 and evaluating it numerically at small k loses every significant figure.

The cost is that particles cannot interact with each other or the world. For sparks, smoke and confetti that is a good trade.

## Verification

- **Motion**: 75 (v₀, gravity, drag) combinations × 41 samples against **scipy integrating the same ODE** (DOP853, rtol 1e-12). Worst relative error **1.3e-12** in position, 6.2e-13 in velocity. The drag branch converges on the parabola as k → 0.
- **Frame-rate independence**: the 57 particles alive at t = 2.0 s are **bit-identical** whether the clock arrived in 1/30, 1/60 or 1/144 s steps.
- **Emission**: 21 (rate, time) pairs — exactly `floor(t·rate)+1` for a stream, a whole number of waves for a burst, and a duration genuinely stops emission.
- **Lifetimes**: 20,000 draws all inside the configured range and uniform across it (Kolmogorov–Smirnov p = 0.78).
- **The cap** keeps the *newest* particles, which is what a real pool does — dropping the newest makes a burst vanish at its peak.
- **Curves**: size, alpha, colour and rotation follow their stated interpolation exactly.
- A **control**: the same particle with drag 0 and drag 2 lands 133 px apart after a second, so "matches the closed form" is a claim about something.

**260 checks.**
