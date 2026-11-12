# 白模运镜参考HTML模板（Three.js）

> 门5 预演门的落地工具：生成可运行的 Three.js 白模 HTML，浏览器打开即播，可录制 webm。
>
> 使用流程：复制下方代码 → 按需修改【配置区】→ 浏览器打开 → 播放/录制 → 校验落点与节奏 → 交付参考视频。

## 使用说明

1. 下方代码是"桌面键帽峡谷一镜到底 + 子弹时间"的完整可运行模板。
2. 修改点集中在 `// ===== 配置区 START =====` 与 `// ===== 配置区 END =====` 之间：
   - `OBJECTS`：巨物清单（位置/尺寸/颜色）
   - `CAM_KEYFRAMES`：相机关键帧（位置/目标/时间）
   - `ACTOR_PATH`：人物走位路径点
   - `WARP_EVENTS`：子弹时间事件表（真实时间 t 区间 + 速度倍率）
   - `DURATION`：总时长
3. 运行环境：任意现代浏览器（Chrome/Edge），Three.js 使用 CDN 引入，需联网。
4. 录制输出 webm，作为白模参考视频交付。

## 完整代码

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<title>小人国白模运镜预演 · 桌面键帽峡谷</title>
<style>
  body { margin:0; overflow:hidden; font-family: "Microsoft YaHei", sans-serif; background:#111; }
  #ui { position:fixed; top:12px; left:12px; z-index:10; color:#fff; background:rgba(0,0,0,.65); padding:10px 14px; border-radius:8px; font-size:13px; line-height:1.8; }
  #ui button { margin-right:6px; padding:4px 12px; cursor:pointer; }
  #info { position:fixed; bottom:12px; left:12px; z-index:10; color:#aaa; font-size:12px; background:rgba(0,0,0,.5); padding:6px 10px; border-radius:6px; }
</style>
</head>
<body>
<div id="ui">
  <div><b>白模预演</b> · 桌面键帽峡谷</div>
  <div>时长 {{DURATION}}s ｜ 播放/暂停/重置</div>
  <div><button onclick="togglePlay()">播放/暂停</button>
       <button onclick="resetAll()">重置</button>
       <button onclick="startRec()">开始录制</button>
       <button onclick="stopRec()">停止并下载</button></div>
  <div id="clock">t=0.0s τ=0.0s</div>
</div>
<div id="info">原点=白球｜人物=橙色胶囊｜巨物=冷灰｜红色=相机路径点｜绿色=落点标记环</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/controls/OrbitControls.js"></script>
<script>
// ===== 配置区 START =====
const DURATION = 12;                 // 视频总时长（真实时间，秒）

// 子弹时间事件表：真实时间 t 的各区间速度倍率（speed=1 正常，<1 慢动作）
const WARP_EVENTS = [
  { t0: 0.0, t1: 3.0, speed: 1.0 },  // 建立段：正常
  { t0: 3.0, t1: 6.0, speed: 1.0 },  // 引入段：正常
  { t0: 6.0, t1: 7.0, speed: 1.0 },  // 推进段：正常（接近撞击）
  { t0: 7.0, t1: 9.0, speed: 0.15 }, // 撞击点进入子弹时间（慢动作）
  { t0: 9.0, t1: 12.0, speed: 1.0 }  // 回落+收尾：恢复正常
];

// 巨物清单（键帽/杯子等）—— 根据需要增删
const OBJECTS = [
  { type:'box',    pos:[6,0.5,2],  size:[3,1,2.2], color:0x8899aa, label:'键帽A' },
  { type:'box',    pos:[2,0.5,-3], size:[3,1,2.2], color:0x8899aa, label:'键帽B' },
  { type:'box',    pos:[-4,0.5,1], size:[3,1,2.2], color:0x8899aa, label:'键帽C' },
  { type:'box',    pos:[-8,0.5,-2],size:[3,1,2.2], color:0x8899aa, label:'键帽D' },
  { type:'cylinder',pos:[9,3,0],   radius:1.6, height:6, color:0x7799bb, label:'马克杯' }
];

// 相机关键帧：[t, 相机位置(posX,posY,posZ), 目标(targetX,targetY,targetZ)]
const CAM_KEYFRAMES = [
  { t: 0.0,  pos:[12, 6, 9],  target:[0, 0, 0] },
  { t: 2.0,  pos:[9, 2.5, 6], target:[0, 1, 0] },
  { t: 5.0,  pos:[4, 1.4, 3], target:[1, 1, 0] },
  { t: 7.5,  pos:[0, 1.2, 1], target:[1, 1, 0] },  // 撞击/子弹时间点
  { t: 10.0, pos:[-5, 1.6, -2], target:[0, 1, 0] },
  { t: 12.0, pos:[-9, 5, -5],  target:[-2, 1, 0] }
];

// 人物走位路径点 [到达时间t, 位置(x,y,z)] —— y 可做跳跃抛物线
const ACTOR_PATH = [
  { t: 0.0, pos:[5.2, 1.0, 0.6] },   // 从键帽A旁探出
  { t: 3.0, pos:[4.2, 1.0, 0.4] },   // 跑到键帽A边缘
  { t: 5.5, pos:[2.8, 1.0, -0.6] },  // 走向键帽间隙
  { t: 6.5, pos:[1.8, 1.0, -1.2] },  // 助跑起点
  { t: 7.0, pos:[1.0, 3.2, -1.8] },  // 起跳腾空（撞击/子弹时间最高点）
  { t: 8.5, pos:[-0.5, 1.0, -2.4] }, // 落在键帽B顶面（落点）
  { t: 10.5,pos:[-1.8, 1.0, -2.0] }, // 站起回望
  { t: 12.0,pos:[-2.0, 1.0, -1.8] }  // 定格
];

// 落点标记环（白模落点对比用）—— 应放在人物最终落地位置
const LANDING_POINT = { x:-0.5, y:1.05, z:-2.4 };
// ===== 配置区 END =====

// ---- 场景搭建 ----
const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1a1a2e);
const camera = new THREE.PerspectiveCamera(50, window.innerWidth/window.innerHeight, 0.1, 200);
const renderer = new THREE.WebGLRenderer({ antialias:true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// 灯光
scene.add(new THREE.AmbientLight(0xffffff, 0.6));
const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
dirLight.position.set(8, 10, 5); scene.add(dirLight);

// 网格地面
const grid = new THREE.GridHelper(30, 20, 0x3a3a5a, 0x2a2a4a);
scene.add(grid);
const ground = new THREE.Mesh(
  new THREE.PlaneGeometry(30, 30),
  new THREE.MeshStandardMaterial({ color:0x22223a, transparent:true, opacity:0.5 })
);
ground.rotation.x = -Math.PI/2; scene.add(ground);

// 原点标记（白球 + AxesHelper）
const origin = new THREE.Mesh(new THREE.SphereGeometry(0.18, 16, 16), new THREE.MeshBasicMaterial({ color:0xffffff }));
scene.add(origin);
const axes = new THREE.AxesHelper(8); scene.add(axes);

// 巨物几何体
OBJECTS.forEach(o => {
  let geo, mat = new THREE.MeshStandardMaterial({ color:o.color, roughness:0.7, metalness:0.1 });
  if (o.type === 'box') geo = new THREE.BoxGeometry(o.size[0], o.size[1], o.size[2]);
  if (o.type === 'cylinder') geo = new THREE.CylinderGeometry(o.radius, o.radius*0.85, o.height, 24);
  const mesh = new THREE.Mesh(geo, mat);
  mesh.position.set(o.pos[0], o.pos[1], o.pos[2]);
  scene.add(mesh);
  if (o.label) { const s = makeLabel(o.label); s.position.copy(mesh.position); s.position.y += o.size ? o.size[1]+0.5 : o.height+0.5; scene.add(s); }
});

// 人物（橙色胶囊 = 圆柱+球）
const actor = new THREE.Group();
const body = new THREE.Mesh(new THREE.CylinderGeometry(0.35, 0.42, 1.2, 12), new THREE.MeshStandardMaterial({ color:0xffaa44 }));
const head = new THREE.Mesh(new THREE.SphereGeometry(0.32, 16, 16), new THREE.MeshStandardMaterial({ color:0xffcc88 }));
body.position.y = 0.6; head.position.y = 1.35;
actor.add(body, head); scene.add(actor);

// 落点标记环
const ring = new THREE.Mesh(new THREE.RingGeometry(0.55, 0.75, 32), new THREE.MeshBasicMaterial({ color:0x33ff66, side:THREE.DoubleSide, transparent:true, opacity:0.85 }));
ring.rotation.x = -Math.PI/2; ring.position.set(LANDING_POINT.x, LANDING_POINT.y, LANDING_POINT.z);
scene.add(ring);

// 相机路径点标记（红色小球）
CAM_KEYFRAMES.forEach(k => {
  const p = new THREE.Mesh(new THREE.SphereGeometry(0.14, 10, 10), new THREE.MeshBasicMaterial({ color:0xff3333 }));
  p.position.set(k.pos[0], k.pos[1], k.pos[2]); scene.add(p);
});

function makeLabel(text) {
  const c = document.createElement('canvas'); c.width = 256; c.height = 48;
  const ctx = c.getContext('2d'); ctx.fillStyle = 'rgba(0,0,0,0)'; ctx.fillRect(0,0,c.width,c.height);
  ctx.font = '28px Microsoft YaHei, sans-serif'; ctx.fillStyle = '#fff'; ctx.textAlign = 'center';
  ctx.fillText(text, 128, 32);
  const tex = new THREE.CanvasTexture(c);
  const s = new THREE.Sprite(new THREE.SpriteMaterial({ map:tex, transparent:true }));
  s.scale.set(2, 0.4, 1); return s;
}

// ---- 时间重映射 ----
function timeWarp(t) {
  let tau = 0;
  for (const e of WARP_EVENTS) {
    if (t <= e.t0) break;
    const seg = Math.min(t, e.t1) - e.t0;
    tau += seg * e.speed;
  }
  return tau;
}

// 分段插值（一维）
function segInterp(a, b, k) { return a + (b - a) * k; }
function cubic(a, b, c, d, t) { // Catmull-Rom
  const t2 = t*t, t3 = t2*t;
  return 0.5*((2*a)+(-a+b)*t+(2*a-5*b+4*c-d)*t2+(-a+3*b-3*c+d)*t3);
}

// 相机按关键帧插值（Catmull-Rom 平滑）
function evalCam(t) {
  const kf = CAM_KEYFRAMES;
  if (t <= kf[0].t) return { pos: kf[0].pos, target: kf[0].target };
  if (t >= kf[kf.length-1].t) return { pos: kf[kf.length-1].pos, target: kf[kf.length-1].target };
  let i = 1;
  while (i < kf.length-1 && kf[i].t < t) i++;
  const p0 = kf[Math.max(0, i-2)], p1 = kf[Math.max(0, i-1)], p2 = kf[i], p3 = kf[Math.min(kf.length-1, i+1)];
  const span = p2.t - p1.t || 1e-6;
  const k = Math.min(Math.max((t - p1.t) / span, 0), 1);
  return {
    pos: [cubic(p0.pos[0], p1.pos[0], p2.pos[0], p3.pos[0], k),
          cubic(p0.pos[1], p1.pos[1], p2.pos[1], p3.pos[1], k),
          cubic(p0.pos[2], p1.pos[2], p2.pos[2], p3.pos[2], k)],
    target: [cubic(p0.target[0], p1.target[0], p2.target[0], p3.target[0], k),
             cubic(p0.target[1], p1.target[1], p2.target[1], p3.target[1], k),
             cubic(p0.target[2], p1.target[2], p2.target[2], p3.target[2], k)]
  };
}

// 人物按路径插值（含跳跃抛物线：对 y 做二次凸起）
function evalActor(t) {
  const ap = ACTOR_PATH;
  if (t <= ap[0].t) return ap[0].pos;
  if (t >= ap[ap.length-1].t) return ap[ap.length-1].pos;
  let i = 1;
  while (i < ap.length-1 && ap[i].t < t) i++;
  const p1 = ap[i-1], p2 = ap[i];
  const span = p2.t - p1.t || 1e-6;
  let k = Math.min(Math.max((t - p1.t) / span, 0), 1);
  const ease = k*k*(3-2*k); // smoothstep
  return [
    segInterp(p1.pos[0], p2.pos[0], ease),
    segInterp(p1.pos[1], p2.pos[1], ease),
    segInterp(p1.pos[2], p2.pos[2], ease)
  ];
}

// ---- 播放控制 ----
let tReal = 0, playing = false, lastTs = 0;
function togglePlay() { playing = !playing; lastTs = performance.now(); }
function resetAll() { tReal = 0; playing = false; }

function animate(now) {
  requestAnimationFrame(animate);
  if (playing) {
    const dt = (now - lastTs) / 1000; lastTs = now;
    tReal = Math.min(tReal + dt, DURATION);
    if (tReal >= DURATION) playing = false;
  } else { lastTs = now; }

  const tau = timeWarp(tReal);
  const cam = evalCam(tReal);
  camera.position.set(cam.pos[0], cam.pos[1], cam.pos[2]);
  camera.lookAt(cam.target[0], cam.target[1], cam.target[2]);
  const ap = evalActor(tau);
  actor.position.set(ap[0], ap[1], ap[2]);

  document.getElementById('clock').textContent = 't=' + tReal.toFixed(1) + 's τ=' + tau.toFixed(1) + 's';
  renderer.render(scene, camera);
}

// ---- 录制（MediaRecorder 输出 webm）----
let recorder = null, chunks = [];
function startRec() {
  chunks = [];
  const stream = renderer.domElement.captureStream(60);
  recorder = new MediaRecorder(stream, { mimeType:'video/webm' });
  recorder.ondataavailable = e => { if (e.data.size) chunks.push(e.data); };
  recorder.start();
  resetAll(); playing = true;
}
function stopRec() {
  if (!recorder) return;
  recorder.stop();
  recorder.onstop = () => {
    const blob = new Blob(chunks, { type:'video/webm' });
    const a = document.createElement('a');
    a.href = URL.createObjectURL(blob);
    a.download = 'white-model-preview.webm';
    a.click();
    recorder = null;
  };
  playing = false;
}

animate(0);
</script>
</body>
</html>
```

## 交付清单（白模预演完成后）

1. `white-model-preview.webm`（白模参考视频，浏览器录制）
2. 落点对比结论（人物落点 vs 标记环偏差，≤0.3 单位）
3. 关键帧表（相机路径点 + 时间）
4. 子弹时间事件表（WARP_EVENTS 各段速度倍率）

> 白模只验证空间/相机/节奏，不继承人物外观与场景材质；最终提示词仍以人物多视角图 + 鸟瞰空镜为外观锚。
