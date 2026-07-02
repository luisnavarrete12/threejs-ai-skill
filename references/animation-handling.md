# Animation Handling

Covers three cases: playing animations a model already has, giving animation to a model that has none, and generating motion procedurally by code.

## 1. Play existing clips

```js
let mixer;
loader.load(url, (gltf) => {
  const model = gltf.scene;
  mixer = new THREE.AnimationMixer(model);
  if (gltf.animations.length) {
    // Prefer selecting by name — clip order is an export accident, not a contract
    const clip = THREE.AnimationClip.findByName(gltf.animations, 'Walk')
              ?? gltf.animations[0];
    mixer.clipAction(clip).play();
  }
  scene.add(model);
});
// in the loop:
mixer && mixer.update(dt); // dt from clock.getDelta()
```

Log `gltf.animations.map(a => a.name)` when unsure what a model ships with — guessing index 0 is how you end up playing "Death" as the idle.

## 2. Model has NO animation — options (honest limits)

A skill cannot *create* artist-quality animation data — that's an asset, not code. So be honest about what's possible:

- **Mixamo (humanoids):** free Adobe service that auto-rigs a humanoid model and lets you download it with walk/run/etc. baked in. Best default for human-shaped characters. **Won't work well for non-humanoids** (a dinosaur, a quadruped) because auto-rig expects human anatomy.
- **Retarget a clip from another file:** if the skeletons are compatible (matching bone names/hierarchy), load the clip from a separate file and play it via `AnimationMixer` directly. For skeletons that are *similar but not identical*, `SkeletonUtils.retargetClip(target, source, clip, options)` from `three/addons/utils/SkeletonUtils.js` can remap it — bone-name mapping via `options.names`. Compatibility is still the catch; verify visually.
- **Blender (any shape):** rig and animate by hand, or source a clip for that specific creature. Most work, full control — the realistic route for a dinosaur.
- **Procedural (see below):** animate by code. No artist needed, works on any shape, but limited fidelity.

Decision rule: humanoid → Mixamo; compatible rig + external clip → retarget; unusual creature → Blender or procedural.

**Duplicating an animated character:** never `model.clone()` a `SkinnedMesh` — clones share the source skeleton and animate in lockstep or break. Use `SkeletonUtils.clone(model)` and give each copy its own `AnimationMixer`.

## 3. Procedural motion — animate by code

> ⚠️ **Experimental — always look at the result.** Procedural walking can look decent for simple/mechanical rigs and can look bad for organic creatures. Never promise a good result sight-unseen; generate it, view it, then decide.

If the model has a bone hierarchy you can address, drive bones with math (sine waves offset per limb) to fake a gait:

```js
// Pseudo-pattern: oscillate legs out of phase, bob the body.
const t = clock.getElapsedTime();
const stride = 0.6, freq = 4;
leftLeg.rotation.x  = Math.sin(t * freq) * stride;
rightLeg.rotation.x = Math.sin(t * freq + Math.PI) * stride; // opposite phase
body.position.y = baseY + Math.abs(Math.sin(t * freq)) * 0.05; // subtle bob
```

Find bones by name with `model.getObjectByName('LeftLeg')` or dump the hierarchy with `model.traverse(o => o.isBone && console.log(o.name))` first — bone names are export-dependent.

Requirements and caveats:
- You need named/addressable bones (or clearly separable leg meshes). No rig → no procedural gait.
- Sync the oscillation frequency to travel speed, or it slides like §Locomotion warns.
- Good for robots, insects, simple toys; risky for realistic quadrupeds. Test before shipping.

## Checklist
- [ ] Existing clips: selected by name, mixer created and updated with dt
- [ ] No animation: correct route chosen (Mixamo / retarget / Blender / procedural)
- [ ] Duplicated characters use `SkeletonUtils.clone`, one mixer each
- [ ] Procedural: bones addressable, frequency synced to speed, **result visually checked**
