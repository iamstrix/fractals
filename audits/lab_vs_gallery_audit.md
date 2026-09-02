# Lab vs Gallery: Configuration Coverage Audit

## Lab's Shader Architecture

The Laboratory generates shaders using **one fixed template**:

```
for each iteration:
  1. Space folding operator (one of 7 types)
  2. Rotation:  rot(fi * rot_base + t * rot_time)
  3. Wave warp:  uv.x += amp * sin(uv.y * freq + t)
  4. Distance:   d = length(uv) * exp(-length(uv0) * decay)
  5. Sine mod:   d = sin(d * freq + t * rate)
  6. Glow:       d = pow(numerator / d, exponent)
  7. Color:      IQ cosine palette accumulation
```

This covers **25 knobs** across 5 cards.

---

## Gallery Shaders That FIT the Lab Template

These shaders can be faithfully reproduced by the Lab's 25 knobs:

| Shader | Notes |
|--------|-------|
| **GLITCH GEOM** | Circle inversion + rot + warp + glow. Almost 1:1, except it divides the sine result by a constant (see below). |

Only **1 out of 16** Gallery shaders maps cleanly to the Lab template.

---

## What the Gallery Uses That the Lab DOESN'T Have

### 1. Sine Divisor (amplitude scaling)
**Used by:** GLITCH GEOM, VOID DRIVE, PARASPASM, PARACORE, GLITCH SPASM, CYBERMORPH 57, XENOCORE, ECHO PULSE

Gallery shaders write: `d = sin(d * freq + t * rate) / divisor;`
Lab generates: `d = sin(d * freq + t * rate);` (no divisor)

The divisor controls the amplitude of the sine wave and dramatically affects glow intensity. **This is the easiest missing knob to add** — it's a single float.

---

### 2. Radial Pulse / UV Breathing
**Used by:** VOID DRIVE, PARACORE, XENOCORE

```glsl
uv *= 1.0 + amplitude * sin(t * timeFreq + length(uv) * spatialFreq);
```

This makes the UV space "breathe" per iteration — a pulsing radial zoom effect. The Lab has no equivalent. Would need **3 knobs**: pulse amplitude, pulse time frequency, pulse spatial frequency.

---

### 3. Distance-Dependent Rotation
**Used by:** PARASPASM, PARACORE, GLITCH SPASM, ECHO PULSE

Gallery shaders modulate rotation by distance from origin:

```glsl
// PARASPASM:
uv *= rot(t * 0.678 + length(uv0) * 6.572);

// PARACORE:
uv *= rot(sin(t + length(uv) * 4.21) + length(uv0) * 4.74);

// GLITCH SPASM:
uv *= rot(t * 0.566 + sin(t + length(uv) * 4.30));
```

Lab's rotation is: `rot(fi * rot_base + t * rot_time)` — no distance dependency at all. This is a fundamentally different rotation model that creates spiraling, vortex-like structures.

---

### 4. Simplex Noise Perturbation
**Used by:** FLESH, PARAFLESH

```glsl
z += 0.15 * vec2(noise(z*2.0 + t), noise(z*2.0 - t));
uv *= rot(noise(uv * 2.0 + t) * 0.4);
```

These shaders include a full simplex noise function (`hash` + `noise`) and use it to add organic, non-deterministic perturbation to the fractal orbit. The Lab has no noise function at all. This creates the "biological/fleshy" look that's impossible to replicate with the Lab's clean mathematical transforms.

---

### 5. Smooth Iteration Count Coloring
**Used by:** FLESH, PARAFLESH

```glsl
float smooth_iter = iter - log2(max(1.0, log2(dot(z,z)))) + 4.0;
```

Classic Mandelbrot/Julia smooth coloring based on escape iteration count. The Lab uses distance-based glow accumulation instead, which produces a fundamentally different visual — flowing neon lines vs. smooth gradient bands.

---

### 6. Orbit Trap Shading
**Used by:** ORBIT TRAP

```glsl
trapCross = min(trapCross, min(abs(uv.x), abs(uv.y)));
trapRing  = min(trapRing, abs(length(uv) - 0.9));
trapPoint = min(trapPoint, length(uv - movingPoint));
// Color from exp(-trap * falloff)
```

Completely different rendering paradigm. Instead of accumulating glow per iteration, it tracks the minimum distance the orbit ever passed to geometric shapes (cross axes, rings, points) and colors based on proximity. Would require a whole new shading mode.

---

### 7. SDF Hard-Edge Rendering
**Used by:** SIERPIN

```glsl
float fill = smoothstep(px, -px, d);
float edge = smoothstep(px * 2.2, 0.0, abs(d) - px * 0.7);
vec3 col = mix(ground, body, fill);
col = mix(col, edgeColor, edge * 0.9);
```

Uses signed distance field rendering with crisp fill + contour outline. No additive bloom, no IQ palette. The shape is rendered as a solid white form with an orange edge on a dark background.

---

### 8. Dark-on-Light / Inverted Tone
**Used by:** BONE

```glsl
vec3 paper = vec3(0.925, 0.912, 0.878);
vec3 col = mix(paper, vec3(0.075, 0.085, 0.105), ink);
```

Draws dark geometric lines on a light paper-toned background. The Lab always renders bright-on-black. An "invert" toggle or background color control would enable this.

---

### 9. Density Counting + Posterization
**Used by:** DENSITY

```glsl
vec2 g = mod(z + 0.5, 1.0) - 0.5;
hits += step(length(g), 0.24);
// ...
col = floor(col * 6.0) / 6.0;  // posterize
col += 0.10 * step(0.5, fract(dens * 6.0));  // banding
```

Counts how many iterations land inside a grid cell (density accumulation), then posterizes the output into flat color bands. Both the accumulation method and the posterization step are absent from the Lab.

---

### 10. Per-Level LOD Fading
**Used by:** BONE

```glsl
float lod = 1.0 - smoothstep(0.030, 0.070, w);
```

Fades out fractal levels whose stroke width exceeds a threshold — keeps deep recursion readable at small preview sizes. The Lab has no resolution-aware LOD.

---

## Summary Table

| Technique | Lab Has It? | Gallery Shaders Using It |
|-----------|:-----------:|--------------------------|
| 7 fold operators | Yes | Most |
| Iteration count | Yes | Most |
| Time speed | Yes | All |
| Global rotation | Yes | 6 shaders |
| Zoom | Yes | Partial |
| Step rotation (`fi * base + t * rate`) | Yes | GLITCH GEOM only |
| Warp X/Y | Yes | 7 shaders |
| Distance decay | Yes | 10 shaders |
| Sine frequency/time | Yes | 10 shaders |
| Glow numerator/exponent | Yes | 10 shaders |
| IQ cosine palette (a,b,c,d) | Yes | 10 shaders |
| Palette speed/spread | Yes | 10 shaders |
| **Sine divisor** | **No** | 8 shaders |
| **Radial pulse / UV breathing** | **No** | 3 shaders |
| **Distance-dependent rotation** | **No** | 4 shaders |
| **Simplex noise perturbation** | **No** | 2 shaders |
| **Orbit trap coloring** | **No** | 1 shader |
| **SDF hard-edge rendering** | **No** | 1 shader |
| **Dark-on-light inversion** | **No** | 1 shader |
| **Density counting** | **No** | 1 shader |
| **Posterization / banding** | **No** | 1 shader |
| **Smooth iteration coloring** | **No** | 2 shaders |
| **Per-level LOD** | **No** | 1 shader |

---

## Verdict

The Lab's 25 knobs cover **one specific shader architecture** (iterative fold + rotation + warp + glow accumulation + IQ palette). This architecture accounts for roughly **10 of the 16 Gallery shaders** in terms of the core loop structure, but even those 10 use a **sine divisor** that the Lab omits.

The remaining **6 Gallery shaders** (FLESH, PARAFLESH, ORBIT TRAP, SIERPIN, BONE, DENSITY) use fundamentally different rendering approaches that can't be reproduced by adjusting any combination of the current 25 knobs.

### Easiest wins to close the gap:
1. **Add sine divisor knob** (1 knob) — instantly improves fidelity for 8 Gallery shaders
2. **Add radial pulse knobs** (3 knobs) — covers VOID DRIVE, PARACORE, XENOCORE
3. **Add distance-rotation factor** (1-2 knobs) — covers PARASPASM, GLITCH SPASM, ECHO PULSE
