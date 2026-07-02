# Model Normalization Helpers

Drop-in helpers to fix the "wrong scale / wrong orientation / floating off-axis" problem when loading GLTF/GLB/FBX models.

## Normalize scale and center

```js
import * as THREE from 'three';

/**
 * Uniformly scales a model so its largest dimension equals `targetSize`
 * world units, and re-centers its pivot to the origin.
 * Returns useful metrics for placement.
 */
function normalizeModel(model, targetSize = 1) {
  // Measure from a clean transform — a pre-set position/scale corrupts the math
  model.position.set(0, 0, 0);
  model.rotation.set(0, 0, 0);
  model.scale.setScalar(1);
  model.updateMatrixWorld(true);

  const box = new THREE.Box3().setFromObject(model);
  const size = box.getSize(new THREE.Vector3());
  const center = box.getCenter(new THREE.Vector3());

  // Uniform scale based on the largest axis
  const maxAxis = Math.max(size.x, size.y, size.z);
  const scale = targetSize / maxAxis;
  model.scale.setScalar(scale);

  // Re-center: move the pivot so the model sits at its own origin
  model.position.sub(center.multiplyScalar(scale));
  model.updateMatrixWorld(true);

  // Recompute post-scale height so callers can sit it on the ground
  const scaledBox = new THREE.Box3().setFromObject(model);
  const scaledSize = scaledBox.getSize(new THREE.Vector3());

  return { scale, height: scaledSize.y, box: scaledBox };
}
```

**SkinnedMesh caveat:** `Box3.setFromObject` uses each mesh's bounding box, which for skinned meshes reflects the **bind pose** — often a T-pose with arms out, so the measured width is inflated. For characters, pass `precise = true` (`new THREE.Box3().setFromObject(model, true)`) to measure actual vertices, or measure a known static child mesh instead.

**FBX caveat:** FBX files are very often authored in centimeters. If a model loads ~100× too big, that's why — `normalizeModel` absorbs it, which is exactly the point of never trusting file units.

## Sit a model on the ground plane (y = 0)

```js
function placeOnGround(model, groundY = 0) {
  const box = new THREE.Box3().setFromObject(model);
  model.position.y += groundY - box.min.y; // rest the lowest point on the floor
}
```

## Orientation note

There is no reliable automatic way to know a model's intended "forward" — it depends on how the artist exported it. Do NOT guess. Options, in order of preference:
1. Ask/expose an explicit rotation the user can set.
2. If the model has named nodes or a known convention, use that.
3. Default to no rotation and make it visible/tweakable, rather than silently rotating.

## Loader boilerplate (GLB, with Draco)

```js
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/addons/loaders/DRACOLoader.js';

const draco = new DRACOLoader();
draco.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.6/');
const loader = new GLTFLoader();
loader.setDRACOLoader(draco);

loader.load(url, (gltf) => {
  const model = gltf.scene;
  normalizeModel(model, 2);
  placeOnGround(model);
  scene.add(model);
}, undefined, (err) => {
  console.error('Model failed to load — check path/CORS/format:', err);
});
```

Always include the error callback — a silent load failure is exactly how you get an invisible or untextured model.

Two related loader gotchas:
- Models compressed with **meshopt** or textures in **KTX2** need `MeshoptDecoder` / `KTX2Loader` wired the same way as Draco, or geometry/textures silently fail.
- **Cloning a character:** `model.clone()` breaks `SkinnedMesh` (clones share the original skeleton). Use `SkeletonUtils.clone(model)` from `three/addons/utils/SkeletonUtils.js` to duplicate rigged models.
