# Input & Controls

Fixes controls that feel broken: diagonal speed-up, frame-rate-dependent movement, keys that "stick," and mouse-look that flips upside down. These are invisible until you actually play the scene — which is why AI never includes them.

## 1. Frame-rate independence — always use deltaTime

Movement tied to frames means fast machines fly and slow machines crawl. Multiply every movement by the time since the last frame, and clamp the delta — after a tab switch `getDelta()` returns seconds, and an unclamped step teleports the player through walls:

```js
const clock = new THREE.Clock();
function animate() {
  const dt = Math.min(clock.getDelta(), 0.05); // clamp tab-switch spikes
  const speed = 5;                             // units per SECOND, not per frame
  player.position.addScaledVector(moveDir, speed * dt);
  controls.update();
  renderer.render(scene, camera);
  requestAnimationFrame(animate);
}
```

## 2. Multi-key state — track pressed keys, don't act in the event

Handle multiple simultaneous keys (forward + left) by tracking a state set, not by moving inside the keydown handler:

```js
const keys = new Set();
addEventListener('keydown', (e) => keys.add(e.code));
addEventListener('keyup',   (e) => keys.delete(e.code));
addEventListener('blur',    () => keys.clear()); // or keys "stick" on alt-tab

function readMove() {
  const dir = new THREE.Vector3();
  if (keys.has('KeyW')) dir.z -= 1;
  if (keys.has('KeyS')) dir.z += 1;
  if (keys.has('KeyA')) dir.x -= 1;
  if (keys.has('KeyD')) dir.x += 1;
  return dir;
}
```

Use `e.code` (physical key), not `e.key` — `e.key` breaks on non-QWERTY layouts and when Shift changes the character.

## 3. Normalize diagonals — the classic bug

Pressing W+D gives a vector of length √2 ≈ 1.41, so diagonal movement is ~40% faster. Always normalize:

```js
const dir = readMove();
if (dir.lengthSq() > 0) dir.normalize(); // now diagonal == straight speed
```

## 4. Mouse-look — clamp the pitch, apply in YXZ order

Free vertical look flips the world upside down, and applying pitch/yaw with the default Euler order (`XYZ`) makes the camera **roll sideways** as you look around. Both fixes:

```js
let yaw = 0, pitch = 0;
const MAX_PITCH = Math.PI / 2 - 0.01;
addEventListener('mousemove', (e) => {
  yaw   -= e.movementX * 0.002;
  pitch -= e.movementY * 0.002;
  pitch = THREE.MathUtils.clamp(pitch, -MAX_PITCH, MAX_PITCH);
});

camera.rotation.order = 'YXZ'; // yaw first, then pitch — or the camera rolls
// each frame:
camera.rotation.y = yaw;
camera.rotation.x = pitch;
```

Use Pointer Lock (`canvas.requestPointerLock()`) for FPS-style capture.

## 5. Move where the camera looks

`readMove()` returns a direction in **local** space; applying it raw means "W" always walks toward −Z world, regardless of where the player looks. Rotate the input by the yaw before applying:

```js
const dir = readMove();
if (dir.lengthSq() > 0) {
  dir.normalize().applyAxisAngle(new THREE.Vector3(0, 1, 0), yaw);
  player.position.addScaledVector(dir, speed * dt);
}
```

## Checklist
- [ ] All movement scaled by deltaTime (units per second), delta clamped
- [ ] Keys tracked in a state set via `e.code`, movement read each frame
- [ ] Direction vector normalized (diagonals fixed)
- [ ] Mouse-look pitch clamped; `rotation.order = 'YXZ'`; pointer lock if FPS
- [ ] Movement rotated by camera yaw
- [ ] Key set cleared on window blur
