# Claude VGA Selfie — Face Tracking App

**Live demo:** [iuriikruglov.github.io/Claude_selfy](https://iuriikruglov.github.io/Claude_selfy/)

---

## What It Is

A fun single-page web application that renders an animated pixel-art portrait of Claude (the AI assistant) in classic VGA / DOS style — complete with blinking eyes, a talking mouth, PC speaker sound effects, and real-time face tracking via the device's front-facing camera.

The portrait reacts to the user in real time: Claude's eyes follow the user's face around the screen, the background bookshelf shifts with a parallax effect, and when the user smiles, Claude smiles back.

---

## Technology Stack

The entire application is written in **vanilla HTML, CSS, and JavaScript** — no frameworks, no build tools, no back-end. It is a single self-contained `.html` file that runs entirely in the browser.

| Layer | Technology |
|---|---|
| Rendering | HTML5 Canvas 2D API |
| Face tracking | MediaPipe Face Mesh (WebAssembly, loaded from CDN) |
| Audio | Web Audio API (`OscillatorNode` with `square` wave) |
| Styling | Pure CSS with `transform: scale()` for responsive layout |
| Hosting | GitHub Pages (static HTTPS) |

---

## How Face Tracking Works

Face tracking is powered by **[MediaPipe Face Mesh](https://google.github.io/mediapipe/solutions/face_mesh)** — a Google open-source library that runs a neural network entirely on-device inside the browser via WebAssembly. No video data is ever sent to a server.

### Pipeline

1. The browser requests access to the front-facing camera via `navigator.mediaDevices.getUserMedia()`.
2. Each camera frame is passed to `FaceMesh.send()`, which returns a set of **468 facial landmarks** — precise (x, y, z) coordinates mapped to the user's face in normalised 0–1 space.
3. Key landmarks are extracted each frame:
   - **Landmark 1** — nose tip: used as the face's horizontal and depth reference point.
   - **Landmarks 33 & 263** — outer eye corners: used to derive the vertical gaze reference.
   - **Landmarks 234 & 454** — left and right face edges: used to measure face width for normalisation.

### Eye Tracking

The nose tip's X coordinate is mapped to a horizontal pupil offset (inverted, because the camera feed is mirrored for display). The eye midpoint Y coordinate maps to a vertical offset. Both values are smoothed with a low-pass filter (`value += (target - value) * 0.25`) to produce fluid, natural-looking eye movement. The pupils are clamped to ±2 pixels horizontally and ±1 pixel vertically within the pixel-art eye area.

### Parallax Effect

The bookshelf background is drawn offset in the **opposite** direction to the face movement — amplified by a factor of 2× horizontally and 1.5× vertically. This creates a subtle depth illusion: as the user moves left, the background shifts right, as if Claude and the shelf exist on different depth planes.

---

## How Smile Detection Works

Smile detection uses the same 468 landmarks from MediaPipe, without any additional model.

Two geometric signals are measured each frame:

**1. Mouth width ratio**
The horizontal distance between the left mouth corner (landmark 61) and right mouth corner (landmark 291) is measured and divided by the overall face width (landmarks 234→454). At a neutral expression this ratio is approximately 0.35; a genuine smile stretches the corners and raises it toward 0.45+.

**2. Corner elevation**
The average Y position of the two mouth corners is compared to the Y position of the lip centre (midpoint of landmarks 13 and 14). When smiling, the corners rise above the lip centre, producing a positive `cornerLift` value.

Both signals are combined into a single `smileScore`:

```js
const widthRatio = mouthWidth / faceWidth;
const rawSmile   = Math.max(0, (widthRatio - 0.37) * 6 + cornerLift * 8);
```

The score is smoothed over time (`smileIntensity += (rawSmile - smileIntensity) * 0.15`) to avoid flickering, and a threshold of `0.35` triggers the smile state. When active, Claude's pixel-art mouth switches from a flat line to a curved arc — corners raised, centre dipped, teeth visible — drawn pixel by pixel on the HTML Canvas.

---

## Audio

Sound is generated entirely in-code using the **Web Audio API** with a `square` wave oscillator — mimicking the single-channel beeper found on IBM PC motherboards. No audio files are loaded. Different frequencies and durations are sequenced to produce blink clicks and speech-like beeping patterns when Claude "talks".

---

## Responsive Layout

The app uses a fixed design canvas of `560 × 520 px`. On resize, JavaScript calculates `Math.min(scaleX, scaleY)` and applies a single `transform: scale()` to the root container, keeping all proportions pixel-perfect at any window or screen size.
