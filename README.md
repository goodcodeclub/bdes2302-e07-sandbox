# bdes2302-e07

## Seven copy/paste steps: Three.js + MediaPipe

Serve this folder with `python3 -m http.server 8000`, then open
`http://localhost:8000/diorama/`. Use localhost or HTTPS for webcam access.
Internet access is needed for the CDN libraries, Draco decoder, and face/hand models.

[diorama/index.html](diorama/index.html) is just the HTML shell: an empty body,
CDN setup, and seven numbered module scripts in the head. All visible elements
are created in JavaScript. No build step is needed; Tailwind styles the interface.

Start with only the first numbered script tag. Copy each numbered file into
the same folder and add its script tag to the head in order:

1. **[1-scene.js](diorama/1-scene.js)** — a full-screen renderer, perspective
   camera, lights, and a render loop. The scene starts without any objects.
   `OrbitControls` lets you click-drag (or touch-drag) to orbit and scroll/pinch
   to zoom. Right-drag (or two-finger drag) pans. The orbit target is not reset
   each frame, so panning persists.
   This works alone. `updates` is a list of functions run before each render;
   later steps add behavior without replacing the render loop.
   The `rendering` object defaults to normal rendering; step 4 replaces its
   render/resize functions to add postprocessing without a second loop.
2. **[2-primitives.js](diorama/2-primitives.js)** — add the matte cube on the
   left, glossy sphere on the right, and grid
   to the same scene without changing the setup.
3. **[3-model.js](diorama/3-model.js)** — load
   [LittlestTokyo.glb](diorama/LittlestTokyo.glb) with `GLTFLoader` and
   `DRACOLoader`, scale it, center it between the primitives on the grid, and
   play its animation. Keep the GLB alongside the scripts. Exports from step 1
   let this step reuse the same scene.
4. **[4-postprocessing.js](diorama/4-postprocessing.js)** — add a standalone
   dropdown: **Bloom** (default), **Vignette**, **Pixellation**, or **ASCII**. `EffectComposer`
   renders the scene through the selected passes, ending with `OutputPass` for
   display color conversion. Pixelation uses its own scene-rendering pass.
   ASCII uses Three.js `AsciiEffect` to draw a character grid over the canvas;
   its output ignores pointer events so orbit/pan controls remain available.
   Tiny characters use measured letter spacing and fitted line height so the
   grid fills the viewport without leaving a strip along the edge.
   Resize updates the composer, ASCII output, and renderer. This step works before
   the webcam interface and can be omitted without affecting later steps.
5. **[5-landmarks.js](diorama/5-landmarks.js)** — create a minimal Tailwind
   interface, request the webcam, and draw face/hand landmarks over mirrored
   video. **Landmarks do not move the camera yet; drag-to-orbit still works.** Face points are teal; hand points
   are amber. Nose **1**, cheeks **234 / 454**, eyes **33 / 263**, and fingertips
   **4 / 8** are numbered. MediaPipe coordinates are normalized (0–1); drawing
   converts them to pixels. The exported `tracking` object holds raw landmarks.
6. **[6-camera.js](diorama/6-camera.js)** — read those landmarks to drive the
   camera. Nose X displacement controls orbit yaw: `-deltaX * 2π * gain`.
   A quarter-frame shift turns 90° at 1×; a half-frame shift reaches the back
   (180°). Nose Y controls elevation, clamped to ±1.3 radians to avoid
   flipping over the poles. `setFromSphericalCoords` places the camera on a
   sphere around the model center, allowing views from every side.
   Face-width ratio controls orbit radius: `6 * (initialWidth / currentWidth) ** 2`,
   amplified around radius=6 and clamped to 2–12. A larger face moves the camera closer.
   The aspect-corrected eye-line angle drives roll at 1.8×. `lookAt(focus)`
   keeps the model centered before applying local Z roll. Angles and radius
   are smoothed, so camera transitions follow an arc instead of cutting through
   the model. Mouse/touch controls are disabled only while webcam tracking is active.
   Without step 7, `cameraMapping.gain` stays at **1×** regardless of hand gestures.
7. **[7-fingers.js](diorama/7-fingers.js)** — optional finger-based amplification.
   Reads the hand landmarks already detected in step 5 and sets the shared
   `cameraMapping.gain` before the camera update runs. Adds its own instructions
   and live multiplier. Remove this script tag to keep face mapping at 1×.

Each step works with all previous steps; removing a later script does not
break earlier ones. ES module imports share the scene once, even though several
files import step 1. Step 6 imports step 5, so do not enable step 6 before step 5.
Step 7 imports step 6; add it last. Hand detection/display stays in step 5,
but only step 7 connects finger spacing to movement amplification.

### Controls (steps 6–7)

- Sit centered when enabling the webcam; Recenter recalibrates position and roll.
- With step 7, spread thumb and index to multiply movement/depth/roll by **1–4×**.
  Pinch or hide the hand for **1×**. Tip distance (**4 / 8**) is divided by palm
  length (**0 / 9**) to reduce sensitivity to hand distance.
- Without webcam tracking (including before step 6), drag to orbit and scroll
  to zoom; right-drag or two-finger drag to pan. Stopping the webcam restores
  these controls at the current viewpoint, initially aimed at the model center.
  Webcam mapping always uses the model center, independent of manual panning.
- Stop webcam releases the camera. Frames are processed locally, not uploaded.

Face width is an approximate depth cue, not metric distance; head turning can
affect it. Using 2D cheek distance prevents head roll alone from shrinking it.

Model: [Littlest Tokyo by glenatron](https://sketchfab.com/models/94b24a60dc1b48248de50bf087c0f042),
licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
