# Locomotion — walking without sliding or floating

> ⚠️ **Verify visually.** This is one of the hardest things in web 3D. The rules below get you to "decent," but always look at the result — foot-sliding is obvious to the eye and must be confirmed, not assumed.

The classic failure: a character's walk animation moves the legs **in place** (like a treadmill), but you translate the body via code at an unrelated speed. Legs and body desync → the character **slides**. Separately, if the model isn't seated on the floor it appears to **float** or sink.

## 1. The anti-slide rule — match translation speed to the animation

The body's travel speed must match the apparent speed of the feet. Two approaches:

**A. Code-driven movement, tuned to the clip (simplest).** Move the body by code, then pick the animation playback rate (or the move speed) so the planted foot stays still relative to the ground. Expose a tuning factor and adjust until sliding disappears:

```js
const walkSpeed = 1.4;                 // m/s body translation
mixer.timeScale = walkSpeed / CLIP_STRIDE_SPEED; // sync anim to travel
```

`CLIP_STRIDE_SPEED` is how many m/s the animation "implies" at 1x — found by tuning, since it depends on the clip.

**B. Root motion (most accurate, more complex).** Let the animation itself drive translation: read the root bone's per-frame displacement and apply it to the object, instead of moving by code. No desync by construction, but you must extract root motion from the clip.

For most web projects, **A** is enough. Reach for **B** for polished character work.

## 2. Stay on the ground

Raycast down from above the character each frame and place feet on the surface, so it follows terrain instead of floating or clipping. Create the raycaster **once** (not per frame), pass `true` to recurse if the ground is a group, and smooth the correction so steps and mesh seams don't make the character pop:

```js
const ray = new THREE.Raycaster();      // created once, reused
const DOWN = new THREE.Vector3(0, -1, 0);

function snapToGround(character, ground, dt) {
  ray.set(character.position.clone().add(new THREE.Vector3(0, 2, 0)), DOWN);
  const hit = ray.intersectObject(ground, true)[0]; // recursive: groups/terrain
  if (hit) {
    // damp instead of hard-set: no popping on steps or terrain seams
    character.position.y = THREE.MathUtils.damp(
      character.position.y, hit.point.y, 12, dt
    );
  }
}
```

## 3. Turn toward movement — don't snap

Rotate smoothly to face the movement direction:

```js
if (moveDir.lengthSq() > 0) {
  const targetYaw = Math.atan2(moveDir.x, moveDir.z);
  const q = new THREE.Quaternion().setFromAxisAngle(new THREE.Vector3(0,1,0), targetYaw);
  character.quaternion.slerp(q, 1 - Math.pow(0.001, dt)); // frame-rate independent damping
}
```

## 4. Blend states — idle ⇄ walk ⇄ run

Cross-fade between clips instead of hard-cutting, and drive the blend by actual speed:

```js
function setState(next) {
  if (next === current) return;
  actions[next].reset().fadeIn(0.2).play();
  actions[current].fadeOut(0.2);
  current = next;
}
// choose next by speed: 0 → idle, <2 → walk, else run
```

## Checklist
- [ ] Translation speed synced to walk clip (no sliding) — **confirmed by eye**
- [ ] Character raycast onto the ground, damped (no floating/clipping/popping)
- [ ] Smooth turn toward movement direction (no snapping)
- [ ] Cross-faded idle/walk/run driven by speed
