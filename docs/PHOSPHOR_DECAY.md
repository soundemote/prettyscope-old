# Phosphor Decay Notes

This describes the current Prettyscope phosphor look for future ports, including
browser/WebGL work. The current decay model is not copied from PrettyScope
Graveyard. It is the standalone renderer's current golden-look approximation,
combined with the Woscope-style Gaussian beam rendering noted in `SALVAGE.md`.

For the fuller woscope-style walkthrough of the complete beam/deposit/decay
pipeline, see `PHOSPHOR_RENDERING_TECHNIQUE.md`.

## Render Loop

Persistent rendering uses two color textures and framebuffers:

1. Decay the previous texture into the destination framebuffer.
2. Draw the new beam additively into that destination.
3. Present the destination texture to the screen.
4. Swap source and destination next frame.

The beam is drawn in two additive passes:

```cpp
glBlendFunc(GL_SRC_ALPHA, GL_ONE);
beamRenderer_.draw(vertexCount, params.glowColor, params.glowWidth, params.glowStrength, width, height);
beamRenderer_.draw(vertexCount, params.traceColor, params.traceWidth, 1.15f, width, height);
```

The wide pass gives the glow, and the narrow pass gives the bright core.

## Beam Shape

Signal samples are converted into segment quads. The fragment shader evaluates
an integrated Gaussian over each segment:

```glsl
float sigma = max(beamSize * 0.25, 0.001);

alpha = erfApprox(beamFrame.x / (SQRT2 * sigma))
      - erfApprox((beamFrame.x - len) / (SQRT2 * sigma));

alpha *= exp(-beamFrame.y * beamFrame.y / (2.0 * sigma * sigma))
       * beamSize / (2.0 * len);

alpha = clamp(alpha * beamIntensity, 0.0, 1.0);
fragColor = vec4(beamColor, alpha);
```

This is why the trace feels like a beam instead of a simple polyline.

## Phosphor Decay

The decay pass removes the background color first, then operates only on signal
energy:

```glsl
vec3 src = texture(image, uv).rgb;
vec3 signal = max(src - backgroundColor, vec3(0.0));
```

Phosphor mode uses brightness-dependent retention:

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

fragColor = vec4(backgroundColor + signal, 1.0);
```

The important behavior is:

- Bright pixels drain faster through `brightDrain`.
- Dim pixels retain longer through `tailBoost`.
- `afterglow` raises the maximum retention toward `0.9975`.
- The small subtractive floor ensures the burn eventually disappears.
- The final power curve shapes the dim tail without making it permanent.

## Golden Defaults

The current standalone default phosphor values are:

```cpp
traceColor = {1.0f, 0.22f, 0.70f};
glowColor = {0.18f, 0.80f, 1.0f};
traceWidth = 2.0f;
glowWidth = 7.0f;
glowStrength = 0.35f;
persistence = 0.98f;
fastDecay = 0.25f;
afterglow = 0.95f;
decayStyle = DecayStyle::Phosphor;
```

## Frame-Rate Stable Port

The native demo currently runs the decay once per frame. A WebGL port should make
decay frame-rate stable by scaling retention and subtractive decay by elapsed
time:

```glsl
float frameScale = dtSeconds * 60.0;
signal *= pow(keep, frameScale);
signal = max(signal - floorAmount * frameScale, vec3(0.0));
```

Without this, the trail length changes between 60 Hz, 120 Hz, and 144 Hz displays.
