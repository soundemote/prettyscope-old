# Prettyscope Phosphor Rendering Technique

This is the working explanation of how Prettyscope draws oscilloscope light. It
is intentionally written in the spirit of m1el's woscope article:

<https://m1el.github.io/woscope-how/>

woscope showed that a convincing oscilloscope trace should be treated as an
electron beam depositing energy, not as an ordinary polyline. Prettyscope keeps
that idea and adds a separate phosphor screen model: the beam writes energy into
a persistent buffer, the buffer decays like a screen surface, and the final image
is colorized from accumulated light.

## The Short Version

Each frame has three jobs:

1. Decay the previous phosphor buffer into a destination buffer.
2. Add the current beam/dot energy into that destination buffer.
3. Composite the destination buffer to the screen.

The renderer uses ping-pong framebuffers:

```text
source phosphor texture
  -> decay shader
  -> destination phosphor texture
  -> additive beam or dot deposits
  -> visible composite
  -> swap source/destination next frame
```

The important separation is:

- **beam deposit**: new signal energy for this frame
- **phosphor buffer**: memory of previous energy
- **decay shader**: screen chemistry / burn behavior
- **composite shader**: final visible color

If those jobs collapse into "draw a line with alpha," the result looks like a
graph. If the buffer is low precision or the decay is too linear, the result
looks like stepped pixel decay.

## Signal To Beam

The signal first becomes visual positions.

For 1D scope mode:

```text
x = sweep position
y = signal amplitude * gain + offset
```

For XY/vectorscope mode:

```text
x = left or X signal * gain
y = right or Y signal * gain
```

XY mode must preserve square geometry. Window shape should not turn a circle
into an oval. The renderer should fit XY drawing into a centered square inside
the scope pane.

1D mode has one special rule: do not connect the trace across sweep wrap. When
the sweep resets from the right side to the left side, that is a discontinuity,
not a beam segment.

## Beam Segment Mode

The native Prettyscope standalone renderer uses the woscope-style analytic beam
segment method for the golden trace.

Each signal segment becomes a quad. In the fragment shader, each pixel evaluates
how much energy a moving Gaussian beam would have deposited while traveling from
the segment start to the segment end.

In the segment's local frame:

```text
start = (0, 0)
end   = (length, 0)
pixel = (x, y)
```

The beam has Gaussian falloff perpendicular to its path. The shader integrates
the Gaussian over the segment length, which produces a soft beam without brittle
joins.

The practical shader shape is:

```glsl
float sigma = max(beamSize * 0.25, 0.001);

float alpha = erfApprox(x / (sqrt(2.0) * sigma))
            - erfApprox((x - len) / (sqrt(2.0) * sigma));

alpha *= exp(-(y * y) / (2.0 * sigma * sigma))
       * beamSize / (2.0 * len);

alpha = clamp(alpha * beamIntensity, 0.0, 1.0);
```

This is adapted from the woscope beam approach, not from PrettyScope Graveyard.
The source relationship is documented in `docs/SALVAGE.md`.

## Dot Deposit Mode

For small browser/module panes, dot deposit mode can work better than connected
segments. Instead of drawing a continuous strip, each sampled position deposits a
round phosphor impact into the persistent buffer.

The vertex shader places point sprites:

```glsl
gl_Position = vec4(clipPosition, 0.0, 1.0);
gl_PointSize = dotSize;
```

The fragment shader turns each point sprite into a round light deposit:

```glsl
vec2 centered = gl_PointCoord * 2.0 - 1.0;
float r2 = dot(centered, centered);
if (r2 > 1.0) discard;

float gaussian = exp(-r2 * 3.6);
float core = smoothstep(1.0, 0.0, r2);
float alpha = (gaussian * 0.82 + core * 0.18) * intensity;
```

This is still a phosphor model. The dots are not meant to read as dotted UI
lines. They should be small energy deposits that accumulate into a living trace.

Avoid these dot-mode mistakes:

- duplicate start/end dots on every sweep range
- regular point spacing that reads as a dotted polyline
- over-bright endpoints
- too few points for the pane size
- connecting dots with an accidental line pass
- clearing the buffer when a control changes

If endpoints look too bright, skip duplicated range boundaries or reduce the
age/intensity boost at the start and end of each range.

## Additive Deposit

Beam energy is added to the phosphor buffer. Use additive blending:

```cpp
glBlendFunc(GL_SRC_ALPHA, GL_ONE);
```

or, when the shader writes premultiplied energy:

```cpp
glBlendFunc(GL_ONE, GL_ONE);
```

Prettyscope generally uses two additive deposits:

- wide, low-intensity glow/halo
- narrow, higher-intensity core

The halo gives the beam a soft optical footprint. The core gives it focus.

## Phosphor Decay

The decay pass is the heart of the screen-burn feel.

A cheap decay is:

```glsl
signal *= persistence;
```

That is not enough. It fades everything at the same rate, so the trail feels
flat and digital. Prettyscope uses brightness-dependent retention:

```glsl
float brightness = max(max(signal.r, signal.g), signal.b);
float dimTail = 1.0 - smoothstep(0.015, 0.34, brightness);
float softTail = 1.0 - smoothstep(0.18, 0.82, brightness);

float brightDrain = brightness * mix(0.035, 0.24, fastDecay);
float tailBoost = dimTail * mix(0.0, 0.055, afterglow)
                + softTail * afterglow * 0.012;

float keep = clamp(
    persistence + tailBoost - brightDrain,
    0.0,
    mix(0.982, 0.9975, afterglow)
);

signal *= keep;
signal = max(signal - vec3(mix(0.0009, 0.00018, afterglow)), vec3(0.0));
signal = pow(signal, vec3(mix(1.035, 1.012, afterglow)));
```

This gives the useful physical illusion:

- fresh bright energy cools quickly
- dim energy lingers
- the long tail still eventually disappears
- screen burn feels slow without becoming permanent

## Why Stepped Trails Happen

The ugly "stacked bars of light" artifact happens when decay is visible in
discrete pixel levels.

Common causes:

- `RGBA8` accumulation texture with a long decay tail
- subtracting too much each frame
- multiplying by nearly the same value every frame until 8-bit quantization is
obvious
- drawing a very bright dot/beam into a low-precision buffer
- no dithering or noise in the decay/composite path

Preferred fixes:

1. Use floating point or half-float accumulation where available.
2. Keep deposited intensity lower and let persistence build brightness.
3. Use nonlinear brightness-dependent decay.
4. Add a tiny floor drain so the tail dies smoothly.
5. In `RGBA8` fallback paths, add very subtle dither/noise before quantization.
6. Avoid huge screen-burn values with 8-bit buffers.

In WebGL2, prefer `RGBA16F` if renderable. In WebGL1, check for the relevant
floating point texture and color-buffer extensions. If only `RGBA8` is
available, expose a safer burn range or dither the tail.

## Color From Energy

Do not draw the same RGB color with lower alpha forever. That looks like a UI
stroke.

Instead, map accumulated energy to color:

```text
low energy  -> deep blue/teal edge
mid energy  -> saturated phosphor color
high energy -> pale hot core
```

The core can drift toward white because intense phosphor and bloom read as a hot
emission, not just a brighter flat color.

Hue controls should rotate the palette. Screen burn should not change hue.
Signal gain should not change camera zoom.

## Frame-Rate Stability

Decay should be stable across monitor refresh rates.

If the phosphor math was tuned for 60 Hz:

```glsl
float frameScale = dtSeconds * 60.0;
signal *= pow(keep, frameScale);
signal = max(signal - floorAmount * frameScale, vec3(0.0));
```

Without this, 144 Hz displays decay differently than 60 Hz displays.

## First-Pass Parameter Meanings

Useful controls:

```text
signal gain       = amplitude scale only
line thickness    = beam/dot footprint size
glow strength     = wide halo deposit intensity
screen burn       = long-tail phosphor memory
fast decay        = bright energy drain
afterglow         = dim tail retention
hue               = palette rotation
brightness        = emitted energy scale
```

Screen burn is not a raw persistence multiplier. It should map to a few internal
values:

```text
more burn -> slower dim-tail decay
more burn -> slightly softer bright decay
more burn -> lower subtractive floor
```

But burn must never mean "never clear." The tail should decay slowly but surely.

## Practical Browser Notes

For browser/WebGL module panes:

- keep the phosphor buffer per canvas or per atlas, not per DOM element if that
  is too expensive
- use scissor regions so each pane decays independently
- preserve XY aspect by rendering inside a square rect
- do not clear when changing gain, hue, or thickness
- clear only when the target texture is resized or the user asks
- use point/dot mode for tiny module panes if segment beams look too line-like
- use analytic segment mode for larger high-fidelity scope surfaces

For known oscillator modules:

- draw from oscillator phase when possible
- use stable phasors rather than wall-clock sample chunk edges
- trigger or phase-align unknown buffers
- avoid repeatedly over-depositing sweep endpoints

## Relationship To woscope

woscope's important lesson is that oscilloscope rendering benefits from modeling
the beam as a physical process. Prettyscope follows that philosophy.

The difference is scope:

- woscope explains a beautiful WebGL beam segment
- Prettyscope adds persistent phosphor memory
- Prettyscope has both segment and dot deposit modes
- Prettyscope treats decay, bloom, color, gain, and host/plugin state as separate
  systems

This document should be updated whenever the golden renderer changes.
