# VVV (Virtual Visit Vitae) - The Technical Bible

![VVV Engine Header](https://via.placeholder.com/1200x300.png?text=VVV+Neural+Swarm+WebGL2+Engine)

> **"A living, breathing particle presentation engine where data flows like a neural swarm."**

---

## 1. Project Overview and Authorship

**VVV (Virtual Visit Vitae)** is a state-of-the-art neural-swarm WebGL2 particle presentation engine. It transforms static slide images into mesmerizing 20,000-particle WebGL2 swarm visualizations featuring fluid, physics-based transitions. 

- **Concept, Architecture & Mathematical Vision**: Valery Kourbanov (kourbanov.com)
- **Code Implementation**: Antigravity AI (Google Gemini Pro & Flash) - 100% generated in code.

The engine elegantly deconstructs visual information into a highly optimized binary particle buffer and recomposes it dynamically in the browser, creating an organic, 'alive' portfolio experience.

---

## 2. Complete Project History and Evolution

The evolution of VVV is a testament to iterative refinement, deep reverse engineering, and AI-assisted architectural discovery.

### Genesis
Valery Kourbanov had a bold vision for an 'alive' portfolio and presentation site. Instead of standard image transitions, the presentation would be a continuous stream of particles that form images and organically flow between slides like a neural swarm, emphasizing the interconnectedness of ideas.

### Early Iterations (v1-v3)
The initial versions established the basic particle systems and primitive image encoders. They were functional but lacked the fluidity, performance optimizations, and exact mathematical precision required for a truly seamless experience.

### The Master Benchmark (v7)
The breakthrough occurred with the discovery of `VVV_ideal.html` as the gold standard. This master template contained 20 pre-baked slides from `kourbanov.com`, processed with deep learning. It served as the ultimate benchmark for how the particle data should be structured and rendered.

### The Build System Problem & Reverse Engineering
The previous build system (`VVV.py`) was compiling particles into the wrong binary format—specifically, a `Float32` array taking up 40 bytes per particle. This was bloated and caused rendering anomalies.

By rigorously inspecting the `VVV_ideal.html` JavaScript and shader code, the Antigravity AI team successfully reverse-engineered the *true* underlying format:
1. **Particle Buffer Structure**: A hyper-optimized 10 bytes per particle, consisting of `UNSIGNED_SHORT` for coordinates and `UNSIGNED_BYTE` for attributes.
2. **Latent/Heatmap**: The engine expects an actual PNG image of the slide (not a JPEG, not a synthetic semantic mask).
3. **Coordinate System**: The Y-axis is stored top-down (0 at the top, 65535 at the bottom), with the inversion to WebGL's Normalized Device Coordinates (NDC) handled entirely in the shader.
4. **Semantic Classes**: 
   - `0` = base (sampled from `heat.r`)
   - `127` = halo (sampled from `heat.g`)
   - `255` = text (sampled from `heat.b`)
5. **Text Enforcement**: The fragment shader explicitly forces the text class (255) to render as pure black.

### The Color Fix
An issue arose where the semantic heatmap PNG generated a solid green background. This was resolved by injecting the actual slide image as `latent_b64`, ensuring the `u_heatmap` texture sampled the correct visual data for both particles and the background quad.

### Current Stable State
The VVV engine can now ingest any PNG/JPG slides, execute importance sampling, compile them into the hyper-optimized 10-byte binary format, and synthesize a fluid standalone HTML presentation.

---

## 3. Full Mathematical Specification

The motion of the swarm is governed by a set of continuous equations evaluated per-vertex on the GPU.

### Minimum Jerk Easing (Smooth Transitions)
To ensure organic acceleration and deceleration, the transition parameter $t$ is mapped through a minimum jerk easing function:
```math
t' = t^3 \cdot (t \cdot (t \cdot 6.0 - 15.0) + 10.0)
```

### Curl Kernel Field (Fluid Turbulence)
The "swarm" behavior is generated using a pseudo-random hash to seed a curl kernel field, creating localized vortices:
```math
H(\mathbf{p}) = \text{fract}(\sin(\mathbf{p} \cdot \langle 12.9898, 78.233 \rangle) \times 43758.5453)
```
```math
\theta(\mathbf{p}, t) = H(\mathbf{p} \cdot \text{seed}) \cdot 2\pi + \phi_{\text{offset}} + t \cdot \pi
```
```math
\mathbf{V}_{\text{curl}} = \langle \cos(\theta), \sin(\theta) \rangle \cdot S_{\text{curl}}
```

### Cubic Bézier Interpolation (Trajectory)
Particles do not move in straight lines. They follow curved trajectories governed by cubic Bézier curves, expanding outwards before converging on the target:
```math
\Delta = \mathbf{p}_{\text{target}} - \mathbf{p}_{\text{source}}
```
```math
\mathbf{CP}_1 = \mathbf{p}_{\text{source}} + \Delta \cdot C_1
```
```math
\mathbf{CP}_2 = \mathbf{p}_{\text{target}} - \Delta \cdot C_2
```
```math
\mathbf{B}(t') = (1-t')^3\mathbf{p}_{\text{source}} + 3(1-t')^2t'\mathbf{CP}_1 + 3(1-t')t'^2\mathbf{CP}_2 + t'^3\mathbf{p}_{\text{target}}
```

### Final Position Vector
The final position combines the Bézier trajectory with an envelope-scaled curl field (ensuring maximum turbulence at the midpoint $t=0.5$):
```math
\mathbf{P}_{\text{final}} = \mathbf{B}(t') + \sin(\pi \cdot t') \cdot \mathbf{V}_{\text{curl}}
```

---

## 4. Shader Architecture

### Vertex Shader (`VVV_ideal.html` spec)
The vertex shader is responsible for unpacking the 10-byte attributes, evaluating the mathematical trajectories, and projecting the coordinates into NDC.

```glsl
// Minimum jerk easing
float ease(float t) {
    return t * t * t * (t * (t * 6.0 - 15.0) + 10.0);
}

// Pseudo-random hash
float hash(vec2 p) {
    return fract(sin(dot(p, vec2(12.9898, 78.233))) * 43758.5453);
}

// Curl kernel field
vec2 kernelField(vec2 pos, float t, float seed) {
    float h = hash(pos * seed);
    float angle = h * 6.28318 + u_phase_offset + t * 3.14159;
    return vec2(cos(angle), sin(angle)) * u_curl_strength;
}

// Cubic Bezier interpolation
vec2 cubicBezier(vec2 p0, vec2 p1, vec2 p2, vec2 p3, float t) {
    float u = 1.0 - t;
    return u*u*u*p0 + 3.0*u*u*t*p1 + 3.0*u*t*t*p2 + t*t*t*p3;
}

void main() {
    // Main trajectory
    vec2 delta = a_target - a_source;
    vec2 cp1 = a_source + delta * u_cp1_coeff;
    vec2 cp2 = a_target - delta * u_cp2_coeff;
    float t = ease(t_raw);
    float envelope = sin(PI * t);
    vec2 pos = cubicBezier(a_source, cp1, cp2, a_target, t)
             + envelope * kernelField(a_source, t, u_seed);

    // NDC conversion (shader inverts Y)
    pos.x = pos.x * 2.0 - 1.0;
    pos.y = -(pos.y * 2.0 - 1.0);
    
    gl_Position = vec4(pos, 0.0, 1.0);
}
```

### Fragment Shader Key Logic
The fragment shader handles the semantic coloring and aura rendering.
- **Class 255 (Text)**: `coreColor = vec3(0,0,0)` (Always pure black), `coreAlpha=1.0` (Sharp edges).
- **Class 0 (Base)**: `coreColor = mix(vec3(0.4), vec3(0.1), v_weight)` (Grayscale gradients).
- **Phantom Particles**: Renders 3 orbiting dots per core particle for a complex aura effect.
- **Lines**: Utilizes `GL_LINES` mode to draw connections between particles, creating a sweeping energetic grid.

---

## 5. Binary Format Specification

The engine achieves its extreme performance by packing 20,000 particles into a precise 10-byte-per-particle buffer. 

| Byte Offset | Attribute | Type | Range / Description |
|---|---|---|---|
| 0-1 | `a_source.x` | `uint16` | 0..65535 (Normalized 0..1, Left to Right) |
| 2-3 | `a_source.y` | `uint16` | 0..65535 (Normalized 0..1, Top to Bottom). *Inverted in shader.* |
| 4 | `a_sourceWeight` | `uint8` | 0..255 (Normalized 0..1 for rendering strength) |
| 5 | `a_sourceRadius` | `uint8` | Raw values, typically 2..5 pixels in size |
| 6 | Unused | `uint8` | Padding / Reserved |
| 7 | Unused | `uint8` | Padding / Reserved |
| 8 | `a_isStructure` | `uint8` | 0 or 1 (Boolean for structural binding) |
| 9 | `a_semanticClass` | `uint8` | 0=Base, 127=Halo, 255=Text |

---

## 6. Importance Sampling Algorithm

To accurately represent an image with only 20,000 points, VVV uses an intelligent Importance Sampling algorithm during compilation. Dark pixels (like text) and high-contrast edges attract a higher density of particles.

```python
# 1. Invert luminance so dark pixels have higher weights
inv_lum = 255 - grayscale

# 2. Extract edge topology
edge = detect_edges(grayscale)

# 3. Construct probability map (heavily favoring darks and edges)
prob_map = clip(inv_lum - 5, 0, 255) * 4.0 + edge * 2.0 + 0.5

# 4. Normalize probability distribution
prob_dist = prob_map / np.sum(prob_map)

# 5. Sample 20,000 indices based on distribution
indices = np.random.choice(w * h, n_particles, p=prob_dist)
```

---

## 7. WebGL Quad Dissolve & Heatmap Textures

### Latent Heatmap Decoding
The slide images are embedded as Base64 strings. At runtime:
1. Base64 is converted to a `Blob(type='image/png')`.
2. Passed to `createImageBitmap()`.
3. Uploaded as a WebGL Texture.
4. Sampled in the vertex shader as `u_heatmap` to determine the underlying color of the particle.

### Quad Dissolve Transition
Beneath the particles, a full-screen quad transitions the background slide image using a noise-driven dissolve.
```glsl
float noise = fract(sin(dot(uv, vec2(12.9898, 78.233))) * 43758.5453) * 0.1;

// Phase 1 (0.0-0.2): Old slide disassembles via noise
float oldDissolve = 1.0 - smoothstep(0.0, 0.2, u_progress + noise);

// Phase 2 (0.2-0.8): Particle swarm flows in space

// Phase 3 (0.8-1.0): New slide assembles
float newDissolve = smoothstep(0.8, 1.0, u_progress - noise);
```

---

## 8. Aura Swarm Dynamics

When particles are at rest or flowing in space, they exhibit localized dynamic behavior to feel "alive".

```js
// Calculated per-frame
wanderRadius = mix(0.12, 0.03, density)
flow = fract(u_time * 0.5 + pHash)

// Complex harmonic jitter
jitterDir = [
    sin(u_time * 2.0 + pHash * 10.0), 
    cos(u_time * 1.5 + pHash * 8.0)
]

// Apply jitter attenuated by the flow state
pos += jitterDir * wanderRadius * (1.0 - flow)

// Simulated depth pulsing
depth = 0.5 + 0.5 * sin(u_time * 3.0 + pHash * 10.0)
```

---

## 9. 5-Stage Compilation Pipeline & Package Architecture

### Package Architecture
- `VVV.html`: The final standalone WebGL presentation output.
- `VVV.py`: The CLI compiler tool.
- `VVV.md`: This comprehensive technical specification.
- `VVV_ideal.html`: The master gold-standard template.

### 5-Stage Compilation Pipeline
1. **INIT**: Probes the environment, sets up directories, verifies dependencies.
2. **INGEST**: Locates and loads PNG/JPG slide images from the `slides/` directory.
3. **ENCODE**: Executes the Importance Sampling algorithm, extracting 20,000 particles per slide, generating the 10-byte binary buffers, and encoding the PNG heatmaps.
4. **STREAM_OPT**: Progressive chunking. Slide #1 is marked for instant boot, while subsequent slides are deferred for asynchronous loading.
5. **SYNTHESIS**: Injects the compiled JSON manifest, binary buffers, and latent images into the `VVV_ideal.html` master template, writing out a final, standalone `index.html`.

### CLI Reference

**Interactive Mode** (Prompts for inputs):
```bash
python VVV.py
```

**Automated CLI Mode**:
```bash
python VVV.py --slides ./slides --outdir ./build --out index.html
```

#### Flags
- `--slides <path>`: Directory containing source slide images (e.g. `1.png`, `2.png`). Default: `./slides`.
- `--outdir <path>`: Output directory for the build. Default: `./build`.
- `--out <filename>`: Name of the output HTML file. Default: `index.html`.
- `--particles <int>`: (Optional) Override particle count. Default: `20000`.

---

## 10. Test Environment Setup

To validate the engine, a controlled environment is used.
Path: `test_env_fresh/`
Contents:
- `slides/1.png` through `5.png` (Test slide sequence)
- `VVV.py` (The compiler)
- `VVV_ideal.html` (The template)
- `run_build.bat` (A double-click convenience script)

**Execution**: Double-click `run_build.bat`. It will execute the CLI command, compile the slides, and automatically open the resulting `index.html` in the default web browser.

---

## 11. Troubleshooting Guide

| Issue | Cause | Solution |
|---|---|---|
| **Particles appear as a solid block of color** | Incorrect binary format or stride alignment in WebGL. | Ensure the `stride` in `vertexAttribPointer` is exactly 10 bytes and offsets are correctly calculated (0, 2, 4, 5, 8, 9). |
| **Background is solid green/weird color** | `latent_b64` is populated with a synthetic semantic mask instead of the real image. | Ensure the compiler base64 encodes the original visual slide PNG. |
| **Image is rendered upside down** | WebGL NDC Y-axis mismatch. | VVV stores Y top-down (0-65535). Do not invert in python; the vertex shader handles `pos.y = -(pos.y * 2.0 - 1.0);`. |
| **Text class (255) rendering white instead of black** | Fragment shader logic bypass. | Check the `u_heatmap` sampling. The fragment shader must explicitly clamp class 255 to `vec3(0,0,0)`. |
| **Slide transition is instant / No swarm** | `t_raw` or easing function broken. | Verify the `u_progress` uniform is smoothly interpolating between 0.0 and 1.0 in the Javascript render loop. |

---

## 12. Lessons Learned / Developer Notes

- **Binary Efficiency is King**: Moving from a 40-byte Float32 structure to a 10-byte mixed-type structure resulted in a 4x reduction in memory bandwidth, which is crucial when animating 20,000 particles at 60fps.
- **Trust the Shader**: Replicating transformations in Python (like Y-axis inversion) that the shader already expects leads to double-inversions. The source of truth for coordinate spaces must be the shader code itself.
- **Heatmaps as Visuals**: The realization that the `latent_b64` string drives *both* the particle coloring and the background quad dissolve was a critical breakthrough in restoring visual fidelity.
- **Importance Sampling Tuning**: Edge detection is just as important as luminance. Text requires high density to be readable; thus, the probability map must aggressively bias towards sharp contrasts.

> *End of Document. Engine Status: Nominal. Swarm Active.*
