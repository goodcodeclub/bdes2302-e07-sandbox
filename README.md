# bdes2302-e07

## Five copy/paste steps: Three.js + MediaPipe

Serve this folder with `python3 -m http.server 8000`, then open
`http://localhost:8000/diorama/`. Use localhost or HTTPS for webcam access.
Internet access is needed for the CDN libraries, Draco decoder, and face/hand models.

[diorama/index.html](diorama/index.html) is just the HTML shell: an empty body,
CDN setup, and six module scripts in the head (step 1 has two parts). All visible elements
are created in JavaScript. No build step is needed; Tailwind styles the interface.

Start with only the first numbered script tag. Copy each numbered file into
the same folder and add its script tag to the head in order:

1. **[1-scene.js](diorama/1-scene.js)** — a full-screen renderer, perspective
   camera, lights, and a render loop. The scene starts without any objects.
   `OrbitControls` lets you click-drag (or touch-drag) to orbit and scroll/pinch
   to zoom. Panning is disabled so the subject stays centered.
   This works alone. `updates` is a list of functions run before each render;
   later steps add behavior without replacing the render loop.
   Then add **[1-primitives.js](diorama/1-primitives.js)** immediately after it:
   this adds the matte cube on the left, glossy sphere on the right, and grid
   to the same scene without changing the setup.
2. **[2-model.js](diorama/2-model.js)** — load
   [LittlestTokyo.glb](diorama/LittlestTokyo.glb) with `GLTFLoader` and
   `DRACOLoader`, scale it, center it between the primitives on the grid, and
   play its animation. Keep the GLB alongside the scripts. Exports from step 1
   let this step reuse the same scene.
3. **[3-landmarks.js](diorama/3-landmarks.js)** — create a minimal Tailwind
   interface, request the webcam, and draw face/hand landmarks over mirrored
   video. **Landmarks do not move the camera yet; drag-to-orbit still works.** Face points are teal; hand points
   are amber. Nose **1**, cheeks **234 / 454**, eyes **33 / 263**, and fingertips
   **4 / 8** are numbered. MediaPipe coordinates are normalized (0–1); drawing
   converts them to pixels. The exported `tracking` object holds raw landmarks.
4. **[4-camera.js](diorama/4-camera.js)** — read those landmarks to drive the
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
   Without step 5, `cameraMapping.gain` stays at **1×** regardless of hand gestures.
5. **[5-fingers.js](diorama/5-fingers.js)** — optional finger-based amplification.
   Reads the hand landmarks already detected in step 3 and sets the shared
   `cameraMapping.gain` before the camera update runs. Adds its own instructions
   and live multiplier. Remove this script tag to keep face mapping at 1×.

Each step works with all previous steps; removing a later script does not
break earlier ones. ES module imports share the scene once, even though several
files import step 1. Step 4 imports step 3, so do not enable step 4 before step 3.
Step 5 imports step 4; add it last. Hand detection/display stays in step 3,
but only step 5 connects finger spacing to movement amplification.

### Controls (steps 4–5)

- Sit centered when enabling the webcam; Recenter recalibrates position and roll.
- With step 5, spread thumb and index to multiply movement/depth/roll by **1–4×**.
  Pinch or hide the hand for **1×**. Tip distance (**4 / 8**) is divided by palm
  length (**0 / 9**) to reduce sensitivity to hand distance.
- Without webcam tracking (including before step 4), drag to orbit and scroll
  to zoom. Stopping the webcam restores these controls at the current viewpoint.
- Stop webcam releases the camera. Frames are processed locally, not uploaded.

Face width is an approximate depth cue, not metric distance; head turning can
affect it. Using 2D cheek distance prevents head roll alone from shrinking it.

Model: [Littlest Tokyo by glenatron](https://sketchfab.com/models/94b24a60dc1b48248de50bf087c0f042),
licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
