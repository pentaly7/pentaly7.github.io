<script setup>
import * as THREE from 'three'
import {onMounted, ref} from "vue";
import {WebGLRenderer} from "three";

/**@type {{value: HTMLCanvasElement}} */
const mainCanvasRef = ref(null)

onMounted(() => {
  const canvas = document.getElementById("main-canvas");
  const renderer = new WebGLRenderer({
    canvas,
    antialias: false,
    alpha: false,
  });
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.setSize(window.innerWidth, window.innerHeight);

  const scene = new THREE.Scene();
  scene.background = new THREE.Color(0x00000a);

  const camera = new THREE.PerspectiveCamera(
      62,
      window.innerWidth / window.innerHeight,
      0.1,
      2000,
  );
  camera.position.set(0, 95, 400);
  camera.lookAt(0, 0, 0);

  function makeGlow(size, stops) {
    const c = document.createElement("canvas");
    c.width = c.height = size;
    const ctx = c.getContext("2d");
    const g = ctx.createRadialGradient(
        size / 2,
        size / 2,
        0,
        size / 2,
        size / 2,
        size / 2,
    );
    stops.forEach(([t, col]) => g.addColorStop(t, col));
    ctx.fillStyle = g;
    ctx.fillRect(0, 0, size, size);
    return new THREE.CanvasTexture(c);
  }

  const N = 30000;
  const GM = 1200;
  const DT = 1 / 60;
  const R_MIN = 5;
  const R_MAX = 275;
  const V_MAX = 24;
  const EPS2 = 40;
  const R2_MIN = R_MIN * R_MIN;
  const R2_MAX = R_MAX * R_MAX;
  const V2_MAX = V_MAX * V_MAX;

  const pos = new Float32Array(N * 3);
  const vel = new Float32Array(N * 3);
  const col = new Float32Array(N * 3);

  function setColor(i3, r) {
    const t = Math.min(r / 255, 1);
    let R, G, B;
    if (t < 0.22) {
      const s = t / 0.22;
      R = 1.0 - s * 0.1;
      G = 1.0;
      B = 1.0;
    } else if (t < 0.5) {
      const s = (t - 0.22) / 0.28;
      R = 0.9 - s * 0.75;
      G = 1.0 - s * 0.3;
      B = 1.0;
    } else if (t < 0.75) {
      const s = (t - 0.5) / 0.25;
      R = 0.15 + s * 0.3;
      G = 0.7 - s * 0.5;
      B = 1.0 - s * 0.1;
    } else {
      const s = (t - 0.75) / 0.25;
      R = 0.45 - s * 0.35;
      G = 0.2 - s * 0.18;
      B = 0.9 - s * 0.55;
    }
    col[i3] = R;
    col[i3 + 1] = G;
    col[i3 + 2] = B;
  }

  function spawn(i) {
    const i3 = i * 3;
    const r = 22 + Math.pow(Math.random(), 0.65) * 253;
    const theta = Math.random() * Math.PI * 2;
    const yOff = (Math.random() - 0.5) * r * 0.1;

    pos[i3] = r * Math.cos(theta);
    pos[i3 + 1] = yOff;
    pos[i3 + 2] = r * Math.sin(theta);

    const vOrb =
        Math.sqrt(GM / r) * (0.8 + Math.random() * 0.4);
    const vDrft = vOrb * (0.06 + Math.random() * 0.11);

    vel[i3] = -Math.sin(theta) * vOrb - Math.cos(theta) * vDrft;
    vel[i3 + 1] = (Math.random() - 0.5) * 0.12;
    vel[i3 + 2] =
        Math.cos(theta) * vOrb - Math.sin(theta) * vDrft;

    setColor(i3, r);
  }

  for (let i = 0; i < N; i++) spawn(i);

  const geom = new THREE.BufferGeometry();
  geom.setAttribute(
      "position",
      new THREE.BufferAttribute(pos, 3),
  );
  geom.setAttribute("color", new THREE.BufferAttribute(col, 3));

  const mat = new THREE.PointsMaterial({
    size: 1.9,
    map: makeGlow(64, [
      [0, "rgba(255,255,255,1)"],
      [0.18, "rgba(255,255,255,.85)"],
      [0.45, "rgba(160,215,255,.3)"],
      [1, "rgba(0,0,0,0)"],
    ]),
    vertexColors: true,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
    transparent: true,
    opacity: 0.88,
    sizeAttenuation: true,
  });

  const points = new THREE.Points(geom, mat);
  scene.add(points);

  const coreSprite = new THREE.Sprite(
      new THREE.SpriteMaterial({
        map: makeGlow(128, [
          [0, "rgba(220,245,255,1)"],
          [0.08, "rgba(120,210,255,.9)"],
          [0.25, "rgba(60,80,255,.5)"],
          [0.55, "rgba(30,0,100,.15)"],
          [1, "rgba(0,0,20,0)"],
        ]),
        blending: THREE.AdditiveBlending,
        depthWrite: false,
        opacity: 0.95,
      }),
  );
  coreSprite.scale.set(30, 30, 1);
  scene.add(coreSprite);

  [
    { r: 110, a: 0.55, col: "80,40,200" },
    { r: 90, a: 0.4, col: "0,60,160" },
    { r: 130, a: 0.3, col: "0,130,150" },
  ].forEach(({ r, a, col: c }, idx) => {
    const sp = new THREE.Sprite(
        new THREE.SpriteMaterial({
          map: makeGlow(256, [
            [0, `rgba(${c},${a})`],
            [0.5, `rgba(${c},${a * 0.25})`],
            [1, "rgba(0,0,0,0)"],
          ]),
          blending: THREE.AdditiveBlending,
          depthWrite: false,
        }),
    );
    const angle = (idx * Math.PI * 2) / 3 + 0.6;
    sp.position.set(
        Math.cos(angle) * r,
        (Math.random() - 0.5) * 22,
        Math.sin(angle) * r,
    );
    sp.scale.set(180, 180, 1);
    scene.add(sp);
  });

  let mx = 0.5,
      my = 0.5;

  function tick() {
    for (let i = 0, i3 = 0; i < N; i++, i3 += 3) {
      const px = pos[i3],
          py = pos[i3 + 1],
          pz = pos[i3 + 2];
      const r2 = px * px + py * py + pz * pz;
      if (r2 < R2_MIN || r2 > R2_MAX) {
        spawn(i);
        continue;
      }

      const r = Math.sqrt(r2);
      const adt = (GM / (r2 + EPS2)) * DT;

      vel[i3] -= (px / r) * adt;
      vel[i3 + 1] -= (py / r) * adt * 0.07;
      vel[i3 + 2] -= (pz / r) * adt;

      const vx = vel[i3],
          vy = vel[i3 + 1],
          vz = vel[i3 + 2];
      const v2 = vx * vx + vy * vy + vz * vz;
      if (v2 > V2_MAX) {
        const s = V_MAX / Math.sqrt(v2);
        vel[i3] *= s;
        vel[i3 + 1] *= s;
        vel[i3 + 2] *= s;
      }
      pos[i3] += vel[i3] * DT;
      pos[i3 + 1] += vel[i3 + 1] * DT;
      pos[i3 + 2] += vel[i3 + 2] * DT;
    }
    geom.attributes.position.needsUpdate = true;
  }

  let camAngle = 0;
  function animate() {
    requestAnimationFrame(animate);
    tick();

    camAngle += 0.00014;
    camera.position.x = Math.sin(camAngle) * 400;
    camera.position.z = Math.cos(camAngle) * 400;
    camera.position.y = 95 + Math.sin(camAngle * 0.37) * 28;

    const lookX = (mx - 0.5) * 28;
    const lookY = (my - 0.5) * -14;
    camera.lookAt(lookX, lookY, 0);

    const pulse = 1 + 0.12 * Math.sin(Date.now() * 0.0018);
    coreSprite.scale.set(30 * pulse, 30 * pulse, 1);

    renderer.render(scene, camera);
  }

  animate();

  window.addEventListener("resize", () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  });
})

</script>

<template>
  <canvas id="main-canvas" ref="mainCanvasRef">

  </canvas>
</template>