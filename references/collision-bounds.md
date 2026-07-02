# Collisions & World Bounds

Fixes the "objects pass through everything / escape the scene" problem.

## Simple AABB collision (good enough for most scenes)

```js
import * as THREE from 'three';

// Static objects: compute the box ONCE. Recomputing setFromObject every
// frame traverses the whole hierarchy — a per-frame cost that scales badly.
function makeStaticCollider(object) {
  const box = new THREE.Box3().setFromObject(object);
  return { box, update: () => {} };
}

// Moving objects: keep the box in sync each frame
function makeDynamicCollider(object) {
  const box = new THREE.Box3();
  const update = () => box.setFromObject(object);
  update();
  return { box, update };
}

// Before committing a move, check against others
function wouldCollide(moving, others) {
  moving.update();
  return others.some((o) => moving.box.intersectsBox(o.box));
}
```

Usage pattern: compute the *intended* next position, test it, and only apply it if it doesn't collide (or resolve by clamping to the contact point).

**AABB limits to keep in mind:**
- Boxes are **axis-aligned**: a thin wall rotated 45° gets a fat diagonal box, and players collide with air near its corners. For rotated architecture, either axis-align the walls or step up to a physics engine.
- For mesh-accurate checks against complex static geometry (terrain, level meshes), **three-mesh-bvh** is the standard: it accelerates raycasts and shape-casts against real triangles without a full physics engine.

## World bounds — clamp so nothing escapes

```js
const WORLD = {
  min: new THREE.Vector3(-50, 0, -50),
  max: new THREE.Vector3(50, 100, 50),
};

function clampToWorld(position) {
  position.x = THREE.MathUtils.clamp(position.x, WORLD.min.x, WORLD.max.x);
  position.y = THREE.MathUtils.clamp(position.y, WORLD.min.y, WORLD.max.y);
  position.z = THREE.MathUtils.clamp(position.z, WORLD.min.z, WORLD.max.z);
  return position;
}
```

## Ground floor

Never let objects fall below the floor. If you use gravity, clamp after integrating:

```js
if (object.position.y < object.userData.halfHeight) {
  object.position.y = object.userData.halfHeight;
  object.userData.velocityY = 0;
}
```

## When to reach for a real physics engine

Hand-rolled AABB is fine for: pickups, simple walls, keeping a character in bounds.

Switch to **Rapier** (`@dimforge/rapier3d`) or **cannon-es** when you need: sloped surfaces, stacking, realistic bounces, rotation-aware collisions, or many dynamic bodies. Don't hand-roll those — say so and wire up the lib instead. Rapier is the current recommendation for performance (Rust/WASM) and is well supported in react-three-fiber via `@react-three/rapier`.
