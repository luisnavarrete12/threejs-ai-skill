# World Coherence

Fixes the "AI generates infinite garbage / objects float / everything is random-sized" problem. The core idea: **give the scene a budget and composition rules before placing anything.**

## 1. Scene budget — nothing infinite

Never generate unbounded geometry or spawn loops. Define hard limits up front:

```js
const BUDGET = {
  maxObjects: 200,        // hard cap on scene objects
  maxTrisPerObject: 50000,
  world: { x: 100, y: 60, z: 100 }, // finite world size in units
};
```

If a generator loop could exceed these, clamp it. A scene that "keeps generating" is a bug, not a feature.

When many objects share a mesh (trees, rocks, crowd), render them with a single `THREE.InstancedMesh` instead of N separate meshes — hundreds of draw calls collapse into one, and the object budget above stays affordable.

## 2. Relative scale — objects sized against each other

Objects must be dimensioned relative to a known reference, not in absolute file units. Pick a reference (often a ~1.8-unit human height) and scale everything against it:

```js
const HUMAN_HEIGHT = 1.8; // world units = meters, by convention
// a door ~2.1, a chair ~0.9, a car ~1.5 tall, a building ~10+
```

Normalize each loaded model (see model-normalization.md) then scale to its real-world-ish size, so a chair never loads bigger than a house.

## 3. Everything sits on the ground

No object floats or sinks unless explicitly intended. After scaling, drop each object's bounding-box bottom onto the floor with the `placeOnGround` helper in `model-normalization.md` (pass a `groundY` offset if your floor isn't at y = 0).

## 4. Intentional layout — not Math.random()

Place objects with a real strategy derived from their sizes, not magic coordinates:
- **Grid:** spacing = largest object footprint + margin.
- **Radial:** objects around a center at a computed radius.
- **Clustered:** groups with intent (furniture near walls, props on surfaces).

Random scatter is acceptable ONLY when explicitly asked (e.g. "scatter rocks"), and even then bounded to the world and de-overlapped.

## 5. Camera frames something

The camera must look at actual content, never the void or the inside of a mesh. Compute a scene bounding box, aim the camera at its center, and set distance from its size:

```js
const sceneBox = new THREE.Box3().setFromObject(scene);
const center = sceneBox.getCenter(new THREE.Vector3());
const size = sceneBox.getSize(new THREE.Vector3()).length();
camera.position.copy(center).add(new THREE.Vector3(0, size * 0.4, size * 0.8));
camera.lookAt(center);
```

## Checklist
- [ ] Hard caps on object/tri count and world size
- [ ] All objects scaled relative to a known reference
- [ ] Everything resting on the ground plane
- [ ] Layout derived from sizes, not random
- [ ] Camera aimed at real content, distance from scene size
