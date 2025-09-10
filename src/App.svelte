<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { gsap } from 'gsap';
  import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';

  let container;

  // core
  let scene, camera, renderer, sections;
  const clock = new THREE.Clock();

  // objects
  let cube, sphere;
  let particlePoints, particleGeom, particleVel;
  let particleCollider; // still used for hover impulses
  let particleMaterial;
  let particleCenter = new THREE.Vector3(0, 0, 0); // attractor = sphere center
  const particleRadius = 2.2;

  // GLB room
  let room = null;
  let roomLoaded = false;
  let roomGroundY = 0;

  // camera state + parallax
  const camBase = new THREE.Vector3(0, 0, 5);
  const camPos  = new THREE.Vector3().copy(camBase);
  const camTarget = new THREE.Vector3(0, 0, 0);
  const lookAtProxy = { x: camTarget.x, y: camTarget.y, z: camTarget.z };
  const offsetTarget = new THREE.Vector3();
  const offsetCurrent = new THREE.Vector3();
  const MAX_OFFSET_X = 0.6, MAX_OFFSET_Y = 0.35;

  // raycast / mouse
  const raycaster = new THREE.Raycaster();
  const mouse = new THREE.Vector2(-2, -2);

  // 3D UI (attached to camera for the first scene)
  const projects = [
  'cellPACKgpu',
  'cellPAINT',
  'Illustrate',
  'Mesoscope',
  'Virus lesson XR',
  'Mesoscale Explorer'
];

  let allUiMeshes = [];     // convenience: links + projects

  let titleMesh = [];
  let linkMeshes = [];
  let projectMeshes = [];
  let panelMesh = null; // floating window
  let personPanel = null; // panel for people
  let personLine = null;  // line connecting cube face to panel

  // --- responsive state ---
  let isMobile = false;
  let uiScale = 1; // scales title/links/panel uniformly
  const DPR_MAX_DESKTOP = 2.0;
  const DPR_MAX_MOBILE  = 1.5;

  let projectsVisible = false;
  let isProjectsOpen = false;
  const LINK_Y = { Home: 0.9, Projects: 0.5, People: 0.1, Contact: -0.3 }; // base y (camera space)
  const LINK_X = 1.2;                    // right column x
  const STEP_Y = 0.28;                   // vertical spacing
  const GAP_AFTER_PROJECTS = 0.16;       // extra gap before submenu

  // --- PEOPLE / CUBE SETUP ---
  const people = [
    {
      name: 'Ludovic Autin',
      desc: `Ludovic Autin, Ph.D., Institute Investigator \nDr. Autin is an expert in computer graphics and molecular modeling with decades of experience developing computational techniques for mesoscale cellular modeling and visualization. He has been leading the development of the cellPACK suite, which enables the construction and generation of mesoscale atomistic models.`,
      img: 'Ludo.png'
    },
    {
      name: 'Quentin Tallon',
      desc: `Quentin Tallon, Postdoctoral Associate \nDr. Tallon has expertise in computer science and machine learning applied to biological data. He will develop the computational pipeline for generating synthetic tomograms and implement deep learning methods to improve interpretation of experimental cryo-ET data.`,
      img: 'quentin.jpg'
    }
  ];

  const peopleFaces = people.map(p => p.img);
  // If you have fewer than 6 images, the rest will repeat the last one.

  // --- viewport helpers (camera-local anchoring) ---
  const UI_Z = 2.0;           // distance from the camera for text/panel
  const UI_MARGIN = 0.15;     // margin from each screen edge (world units at z=UI_Z)
  const LINK_GAP = 0.12;      // spacing between menu items (extra gap beyond their height)


  function showContactOverlay() {
    const el = document.getElementById('contact-overlay');
    if (!el) return;
    el.classList.remove('hidden');
    el.setAttribute('aria-hidden', 'false');
  }

  function hideContactOverlay() {
    const el = document.getElementById('contact-overlay');
    if (!el) return;
    el.classList.add('hidden');
    el.setAttribute('aria-hidden', 'true');
  }

  // returns {halfW, halfH} of the frustum slice at distance d=UI_Z
  function viewportHalfAtDepth(d = UI_Z) {
    const halfH = Math.tan(THREE.MathUtils.degToRad(camera.fov) * 0.5) * d;
    const halfW = halfH * camera.aspect;
    return { halfW, halfH };
  }

  // returns world size of a plane-like mesh (using geometry + scale)
  function meshSize(mesh) {
    // works for PlaneGeometry; for CanvasText we used PlaneGeometry too
    const g = mesh.geometry;
    let w = 0.6, h = 0.35; // fallback
    if (g && g.parameters && 'width' in g.parameters && 'height' in g.parameters) {
      w = g.parameters.width; h = g.parameters.height;
    } else {
      // robust fallback: compute box
      const box = new THREE.Box3().setFromObject(mesh);
      const s = new THREE.Vector3();
      box.getSize(s);
      w = s.x; h = s.y;
    }
    const sx = mesh.scale?.x ?? 1, sy = mesh.scale?.y ?? 1;
    return { w: w * sx, h: h * sy };
  }

  function ensureSix(arr) {
    const out = arr.slice(0, 6);
    while (out.length < 6) out.push(out[out.length - 1] || 'placeholder.jpg');
    return out;
  }

  function loadPeopleMaterials(onDone) {
    const files = ensureSix(peopleFaces);
    const loader = new THREE.TextureLoader();
    const mats = new Array(6);
    let loaded = 0;

    files.forEach((name, i) => {
      // IMPORTANT: no leading slash so it works under /lab/
      const url = `./assets/images/people/${name}`;
      loader.load(url, (tex) => {
        tex.colorSpace = THREE.SRGBColorSpace;
        tex.anisotropy = Math.min(8, renderer.capabilities.getMaxAnisotropy?.() || 1);
        tex.needsUpdate = true;
        mats[i] = new THREE.MeshStandardMaterial({
          map: tex,
          roughness: 0.45,
          metalness: 0.05
        });
        // give ripple to every face material
        patchMaterialForRipple(mats[i], { color: 0x66ccff, fresnel: 0.8, falloff: 3.2, speed: 2.0, freq: 10.0 });
        if (++loaded === 6) onDone(mats);
      }, undefined, (err) => {
        console.warn('People face failed:', url, err);
        // fallback to a neutral material
        mats[i] = new THREE.MeshStandardMaterial({ color: 0x888888, roughness: 0.5, metalness: 0.05 });
        patchMaterialForRipple(mats[i], { color: 0x66ccff });
        if (++loaded === 6) onDone(mats);
      });
    });
  }

  // convenience: jump to a scene by name
  function gotoByName(name) {
    const i = scenes.findIndex(s => s.name === name);
    if (i >= 0) gotoScene(i);
  }

  // --- orbit math: ring axis = camera forward, ring centered on sphere ---
  function getOrbitTargets(radius = 1.6 * uiScale) {
    const targets = [];
    if (!sphere || !projectMeshes.length) return targets;

    const forward = new THREE.Vector3();
    camera.getWorldDirection(forward).normalize();

    // basis vectors in plane ⟂ forward
    const tmpUp = new THREE.Vector3(0,1,0);
    if (Math.abs(forward.dot(tmpUp)) > 0.9) tmpUp.set(1,0,0);
    const u = new THREE.Vector3().crossVectors(forward, tmpUp).normalize();
    const v = new THREE.Vector3().crossVectors(forward, u).normalize();

    const c = sphere.position;
    const n = projectMeshes.length;
    const step = (Math.PI * 2) / Math.max(1,n);
    const phase = 0; // tweak if you want a starting angle

    for (let i = 0; i < n; i++) {
      const a = phase + i * step;
      const t = new THREE.Vector3()
        .addScaledVector(u, Math.cos(a) * radius)
        .addScaledVector(v, Math.sin(a) * radius)
        .add(c);
      targets.push(t);
    }
    return targets;
  }

  // --- animate "open": from center of sphere → ring targets ---
  function animateProjectsToRing({ radius = 1.6 * uiScale, dur = 0.6 } = {}) {
    if (!projectMeshes.length || !sphere) return;
    const targets = getOrbitTargets(radius);

    // start at sphere center, fade in
    projectMeshes.forEach((m, i) => {
      m.visible = true;
      m.position.copy(sphere.position);
      // ensure material can fade
      if (m.material && 'opacity' in m.material) {
        m.material.transparent = true;
        m.material.opacity = 0.0;
      }
      // burst outward
      gsap.to(m.position, {
        x: targets[i].x, y: targets[i].y, z: targets[i].z,
        duration: dur, ease: 'power3.out', overwrite: true
      });
      gsap.to(m.material, {
        opacity: 1.0, duration: Math.min(0.35, dur * 0.6),
        ease: 'power2.out', overwrite: true
      });
    });
  }

  // --- animate "close": back to center, then hide ---
  function animateProjectsBackToCenter({ dur = 0.4 } = {}) {
    if (!projectMeshes.length || !sphere) return;
    projectMeshes.forEach((m) => {
      gsap.to(m.position, {
        x: sphere.position.x, y: sphere.position.y, z: sphere.position.z,
        duration: dur, ease: 'power3.in', overwrite: true,
        onComplete: () => { m.visible = false; }
      });
      if (m.material && 'opacity' in m.material) {
        gsap.to(m.material, {
          opacity: 0.0, duration: Math.min(0.3, dur),
          ease: 'power2.in', overwrite: true
        });
      }
    });
  }

  // --- follow ring while open (keeps ring facing camera smoothly) ---
  function followProjectsOrbit({ radius = 1.6 * uiScale, lerp = 0.12 } = {}) {
    if (!isProjectsOpen || !projectMeshes.length) return;
    const targets = getOrbitTargets(radius);
    for (let i = 0; i < projectMeshes.length; i++) {
      const m = projectMeshes[i];
      if (!m.visible) continue;
      // smooth-attach to moving ring
      m.position.lerp(targets[i], lerp);
      // billboard
      m.quaternion.copy(camera.quaternion);
    }
  }
  let wasProjectsOpen = false;

  function layoutAccordion(animate = true) {
    const tl = (mesh, x, y) => {
      const nx = x * uiScale, ny = y * uiScale;
      if (animate) gsap.to(mesh.position, { x:nx, y:ny, duration:0.25, ease:'power2.out' });
      else mesh.position.set(nx, ny, mesh.position.z);
    };

    const home = linkMeshes.find(m => m.userData.key === 'Home');
    const peop = linkMeshes.find(m => m.userData.key === 'People');
    const proj = linkMeshes.find(m => m.userData.key === 'Projects');
    const cont = linkMeshes.find(m => m.userData.key === 'Contact');

    home && tl(home, LINK_X, LINK_Y.Home);
    proj && tl(proj, LINK_X, LINK_Y.Projects);
    peop && tl(peop, LINK_X, LINK_Y.People);

    // detect open/close transitions
    if (isProjectsOpen && !wasProjectsOpen) {
      // OPEN: animate satellites from center → ring
      animateProjectsToRing({ radius: 1.6 * uiScale, dur: 0.6 });
      // optionally push Contact down in UI (keep your behavior)
      const pushDown = GAP_AFTER_PROJECTS + projects.length * STEP_Y;
      cont && tl(cont, LINK_X, LINK_Y.Contact - pushDown);
    } else if (!isProjectsOpen && wasProjectsOpen) {
      // CLOSE: ring → center → hide
      animateProjectsBackToCenter({ dur: 0.4 });
      cont && tl(cont, LINK_X, LINK_Y.Contact);
    } else {
      // steady state (no transition), keep Contact placed according to state
      if (isProjectsOpen) {
        const pushDown = GAP_AFTER_PROJECTS + projects.length * STEP_Y;
        cont && tl(cont, LINK_X, LINK_Y.Contact - pushDown);
      } else {
        cont && tl(cont, LINK_X, LINK_Y.Contact);
      }
    }

    wasProjectsOpen = isProjectsOpen;
    anchorUI(animate);
  }

  function setProjectsVisible(flag) {
    projectsVisible = flag;
    projectMeshes.forEach(m => { m.visible = flag; });
  }

  function updateResponsive() {
    const w = window.innerWidth, h = window.innerHeight;
    const aspect = w / h;

    // Simple mobile heuristic
    isMobile = (w <= 900) || ('ontouchstart' in window);

    // DPR clamp
    const dpr = Math.min(isMobile ? DPR_MAX_MOBILE : DPR_MAX_DESKTOP, window.devicePixelRatio || 1);
    renderer.setPixelRatio(dpr);
    renderer.setSize(w, h);

    // FOV adjustment
    const baseFov = 70;
    const extra = THREE.MathUtils.clamp((1/aspect) - 1, 0, 0.35);
    camera.fov = baseFov + extra * 18;
    camera.aspect = aspect;
    camera.updateProjectionMatrix();

    // UI scale ranges (affects text sizes but not anchoring math)
    const shortest = Math.min(w, h);
    uiScale = THREE.MathUtils.clamp(shortest / 900, 0.75, 1.2);

    // scale visual size of text/panel
    titleMesh && titleMesh.scale.setScalar(uiScale);
    linkMeshes?.forEach(m => m.scale.setScalar(uiScale));
    if (panelMesh) panelMesh.scale.setScalar(uiScale);

    // particle point size vs DPR
    if (particleMaterial) {
      particleMaterial.uniforms.uScale.value = dpr * 1.0;
      const basePx = isMobile ? 30.0 : 28.0;
      particleMaterial.uniforms.uSize.value = basePx * uiScale;
    }

    // Shadow map budget (lighter on mobile)
    const dir = scene.children.find(o => o.isDirectionalLight);
    if (dir) {
      dir.shadow.mapSize.set(isMobile ? 1024 : 2048, isMobile ? 1024 : 2048);
      dir.shadow.needsUpdate = true;
    }

    // *** Re-anchor UI after all above changes ***
    anchorUI(false);
  }

  // ---------- helpers -----------------
  function camFromSection(group, localCam = new THREE.Vector3(0,0,5), localTarget = new THREE.Vector3(0,0,0)) {
    const gp = new THREE.Vector3();
    group.getWorldPosition(gp);
    return {
      pos: gp.clone().add(localCam),
      target: gp.clone().add(localTarget)
    };
  }

  // ---------- ripple helpers ----------
  function patchMaterialForRipple(mat, {
    color = 0x66ccff, freq = 8.0, speed = 2.0, falloff = 2.5, fresnel = 1.25
  } = {}) {
    if (!mat || !mat.onBeforeCompile) return false;
    const ok = mat.isMeshStandardMaterial || mat.isMeshPhysicalMaterial || mat.isMeshLambertMaterial || mat.isMeshPhongMaterial || mat.isMeshBasicMaterial;
    if (!ok) return false;

    mat.onBeforeCompile = (shader) => {
      shader.uniforms.uHit = { value: new THREE.Vector3(999,999,999) };
      shader.uniforms.uTime = { value: 0 };
      shader.uniforms.uPulse = { value: 0 };
      shader.uniforms.uColor = { value: new THREE.Color(color) };
      shader.uniforms.uFreq = { value: freq };
      shader.uniforms.uSpeed = { value: speed };
      shader.uniforms.uFalloff = { value: falloff };
      shader.uniforms.uFresnel = { value: fresnel };

      shader.vertexShader = `varying vec3 vWorldPosition;\n` + shader.vertexShader;
      shader.fragmentShader = `varying vec3 vWorldPosition;\n` + shader.fragmentShader;

      shader.vertexShader = shader.vertexShader.replace(
        '#include <project_vertex>',
        `
        vWorldPosition = (modelMatrix * vec4( transformed, 1.0 )).xyz;
        #include <project_vertex>
        `
      );

      shader.fragmentShader = shader.fragmentShader.replace(
        '#include <common>',
        `
        #include <common>
        uniform vec3 uHit;
        uniform float uTime;
        uniform float uPulse;
        uniform vec3 uColor;
        uniform float uFreq;
        uniform float uSpeed;
        uniform float uFalloff;
        uniform float uFresnel;
        float ripple(vec3 p, vec3 c, float t){
          float d = length(p - c) + 1e-5;
          float w = sin(uFreq * (d - uSpeed * t));
          float env = exp(-uFalloff * d);
          return w * env;
        }
        `
      ).replace(
        '#include <lights_fragment_end>',
        `
        #include <lights_fragment_end>
        float r = ripple(vWorldPosition, uHit, uTime) * uPulse;
        vec3 nrm = normalize( vNormal );
        vec3 vdir = normalize( -vViewPosition );
        float fres = pow(1.0 - max(0.0, dot(nrm, vdir)), 2.0) * uFresnel;
        vec3 glow = uColor * clamp(0.0, 1.0, 0.25 * r + 0.15 * fres * uPulse);
        totalEmissiveRadiance += glow;
        `
      );

      mat.userData.shader = shader;
    };

    mat.needsUpdate = true;
    return true;
  }

  function cloneAndRippleMesh(mesh, opts) {
    if (!mesh || !mesh.isMesh || !mesh.material) return;
    if (Array.isArray(mesh.material)) {
      mesh.material = mesh.material.map((m) => {
        const cm = m.clone();
        patchMaterialForRipple(cm, opts);
        return cm;
      });
    } else {
      mesh.material = mesh.material.clone();
      patchMaterialForRipple(mesh.material, opts);
    }
  }

  // ---------- scenes (Room will be pushed after load) ----------
  let scenes = [
    {
      name: 'Home',
      // provisional until GLB gives us ground
      get cam() { return camFromSection(sections.home, new THREE.Vector3(0, 0, 6), new THREE.Vector3(0, 0, 0)); },
      enter: () => {
        hideContactOverlay();
        sections.home.visible = true;
        cube.visible = false;
        sphere.visible = false;
        particlePoints && (particlePoints.visible = false);
        particleCollider && (particleCollider.visible = false);
        if (isProjectsOpen) { isProjectsOpen = false; layoutAccordion(true); }
      }
    },
    {
      name: 'Projects',
      get cam() { return camFromSection(sections.project, new THREE.Vector3(0, 0, 5), new THREE.Vector3(0, 0, 0)); },
      enter: () => {
        hideContactOverlay();
        sections.home.visible = false;
        cube.visible = false;
        sphere.visible = true;
        particlePoints && (particlePoints.visible = false);
        particleCollider && (particleCollider.visible = false);
        gsap.fromTo(sphere.scale, { x:.85, y:.85, z:.85 }, { x:1, y:1, z:1, duration:.8, ease:'power2.out' });
        if (!isProjectsOpen) { isProjectsOpen = true; layoutAccordion(true); }
      }
    },
    {
      name: 'People',
      get cam() { return camFromSection(sections.people, new THREE.Vector3(0, 0, 5), new THREE.Vector3(0, 0, 0)); },
      enter: () => {
        hideContactOverlay();
        sections.home.visible = false;
        cube.visible = true;
        sphere.visible = true; // positioned in its own section; still hidden by camera framing
        particlePoints && (particlePoints.visible = false);
        particleCollider && (particleCollider.visible = false);
        gsap.to(cube.rotation, { x: 0.6, y: 0.8, duration: 1.2, ease: 'power2.out' });
        // small pulse on faces so you instantly “feel” the ripple shader
        const mats = Array.isArray(cube.material) ? cube.material : [cube.material];
        mats.forEach((m) => {
          const sh = m.userData?.shader;
          if (sh) { sh.uniforms.uHit.value.set(0,0,0); sh.uniforms.uPulse.value = 1.0; }
        });
        if (isProjectsOpen) { isProjectsOpen = false; layoutAccordion(true); }
      }
    },
    {
      name: 'Contact',
      get cam() { return camFromSection(sections.contact, new THREE.Vector3(0, 0, 5), new THREE.Vector3(0, 0, 0)); },
      enter: () => {
        showContactOverlay();
        sections.home.visible = false;
        cube.visible = false;
        sphere.visible = false;
        particlePoints && (particlePoints.visible = true);
        particleCollider && (particleCollider.visible = true);
        if (isProjectsOpen) { isProjectsOpen = false; layoutAccordion(true); }
      }
    }
  ];
  let idx = 0;


  function gotoScene(i) {
    if (i === idx || i < 0 || i >= scenes.length) return;
    idx = i;

    const s = scenes[idx];
    s.enter && s.enter();

    const c = s.cam; // {pos,target} computed from group
    gsap.killTweensOf(camBase);
    gsap.killTweensOf(lookAtProxy);

    gsap.to(camBase, { x:c.pos.x, y:c.pos.y, z:c.pos.z, duration:1.2, ease:'power3.out' });
    gsap.to(lookAtProxy, {
      x:c.target.x, y:c.target.y, z:c.target.z, duration:1.2, ease:'power3.out',
      onUpdate: () => camTarget.set(lookAtProxy.x, lookAtProxy.y, lookAtProxy.z)
    });
  }


  // wheel with wrap-around
  let isTransitioning = false;
  let wheelAccum = 0;
  const STEP_THRESHOLD = 80;

  function onWheel(e) {
    if (isTransitioning) return;
    wheelAccum += e.deltaY;
    if (Math.abs(wheelAccum) > STEP_THRESHOLD) {
      const dir = wheelAccum > 0 ? 1 : -1;
      wheelAccum = 0;
      stepScene(dir);
    }
  }

  function stepScene(dir) {
    const next = (idx + dir + scenes.length) % scenes.length;
    isTransitioning = true;
    gotoScene(next);
    setTimeout(() => { isTransitioning = false; }, 1200);
  }

  // ---------- CanvasText helpers (3D text & panels) ----------
  function makeTextMesh(text, { fontSize = 64, color = '#111', padding = 16, bg = null } = {}) {
    const ctx = document.createElement('canvas').getContext('2d');
    const font = `700 ${fontSize}px system-ui, -apple-system, Segoe UI, Roboto, sans-serif`;
    ctx.font = font;
    const metrics = ctx.measureText(text);
    const w = Math.ceil(metrics.width + padding * 2);
    const h = Math.ceil(fontSize + padding * 2);

    ctx.canvas.width = w;
    ctx.canvas.height = h;

    ctx.font = font;
    ctx.textBaseline = 'top';
    if (bg) {
      ctx.fillStyle = bg;
      ctx.fillRect(0,0,w,h);
    }
    ctx.fillStyle = color;
    ctx.fillText(text, padding, padding);

    const tex = new THREE.CanvasTexture(ctx.canvas);
    tex.colorSpace = THREE.SRGBColorSpace;
    tex.needsUpdate = true;

    const mat = new THREE.MeshBasicMaterial({ map: tex, transparent: true });
    mat.side = THREE.DoubleSide;  
    const aspect = w / h;
    const planeH = 0.35; // world units
    const planeW = planeH * aspect;

    const geo = new THREE.PlaneGeometry(planeW, planeH);
    const mesh = new THREE.Mesh(geo, mat);
    mesh.renderOrder = 10;
    mesh.userData.isText = true;
    return mesh;
  }

  function makePanelMesh(title, body, { width = 1.8 } = {}) {
    // simple rounded bg + text via canvas
    const ctx = document.createElement('canvas').getContext('2d');
    const W = 1024, H = 512;
    ctx.canvas.width = W;
    ctx.canvas.height = H;

    // bg rounded
    ctx.fillStyle = 'rgba(255,255,255,0.9)';
    roundRect(ctx, 16, 16, W-32, H-32, 24);
    ctx.fill();

    // title
    ctx.fillStyle = '#0f1115';
    ctx.font = 'bold 64px system-ui,-apple-system,Segoe UI,Roboto,sans-serif';
    ctx.fillText(title, 48, 72);

    // body
    ctx.font = '400 40px system-ui,-apple-system,Segoe UI,Roboto,sans-serif';
    wrapText(ctx, body, 48, 140, W - 96, 52);

    const tex = new THREE.CanvasTexture(ctx.canvas);
    tex.colorSpace = THREE.SRGBColorSpace;
    const mat = new THREE.MeshStandardMaterial({ map: tex, transparent: true, roughness: 0.8, metalness: 0.0 });
    mat.side = THREE.DoubleSide;
    // add ripple to panel (independent)
    patchMaterialForRipple(mat, { color: 0x66ccff, fresnel: 0.8, falloff: 3.2, speed: 2.0, freq: 10.0 });

    const aspect = W/H;
    const height = width / aspect;
    const geo = new THREE.PlaneGeometry(width, height);
    const mesh = new THREE.Mesh(geo, mat);
    mesh.castShadow = false;
    mesh.receiveShadow = true;
    mesh.userData.isPanel = true;
    return mesh;
  }

  function roundRect(ctx,x,y,w,h,r){
    ctx.beginPath();
    ctx.moveTo(x+r, y);
    ctx.arcTo(x+w, y, x+w, y+h, r);
    ctx.arcTo(x+w, y+h, x, y+h, r);
    ctx.arcTo(x, y+h, x, y, r);
    ctx.arcTo(x, y, x+w, y, r);
    ctx.closePath();
  }
  function wrapText(ctx, text, x, y, maxWidth, lineHeight) {
    const words = text.split(' ');
    let line = '';
    for (let n=0; n<words.length; n++) {
      const testLine = line + words[n] + ' ';
      const metrics = ctx.measureText(testLine);
      if (metrics.width > maxWidth && n > 0) {
        ctx.fillText(line, x, y);
        line = words[n] + ' ';
        y += lineHeight;
      } else {
        line = testLine;
      }
    }
    ctx.fillText(line, x, y);
  }

  // ---------- mouse handlers ----------
  let touchStartY = 0;
  let touchAccum = 0;

  function onTouchStart(e) {
    if (!e.touches?.length) return;
    touchStartY = e.touches[0].clientY;
    touchAccum = 0;
  }

  function onTouchMove(e) {
    if (!e.touches?.length) return;
    const dy = e.touches[0].clientY - touchStartY;
    touchAccum += dy;
    touchStartY = e.touches[0].clientY;

    // reuse your STEP_THRESHOLD, flip sign so swipe up = next
    if (Math.abs(touchAccum) > STEP_THRESHOLD) {
      const dir = touchAccum < 0 ? 1 : -1;
      touchAccum = 0;
      stepScene(dir);
    }
  }

  function onPointerMove(e) {
    const rect = renderer.domElement.getBoundingClientRect();
    const nx = ((e.clientX - rect.left) / rect.width) * 2 - 1;
    const ny = -((e.clientY - rect.top) / rect.height) * 2 + 1;
    mouse.x = nx; mouse.y = ny;

    // subtle parallax
    offsetTarget.set(-nx * MAX_OFFSET_X, -ny * MAX_OFFSET_Y, 0);

    // raycast against everything (scene objects + UI)
    raycaster.setFromCamera(mouse, camera);
    // const targets = [cube, sphere, particleCollider, room, ...linkMeshes, panelMesh].filter(Boolean);
    const targets = [cube, sphere, particleCollider, room, ...allUiMeshes, panelMesh].filter(Boolean);
    const hits = raycaster.intersectObjects(targets, true);

    // reset link hover flags
    // linkMeshes.forEach(m => m.userData.hover = 0);
    allUiMeshes.forEach(m => m.userData.hover = 0);

    let hoverProjects = false;
    let hoverContacts = false;
    let hoverAnyProjectItem = false;

    for (const h of hits) {
      const o = h.object;

      // ripple for anything hit (you already do)
      if (o?.material) {
        const mats = Array.isArray(o.material) ? o.material : [o.material];
        mats.forEach((m) => {
          const sh = m.userData?.shader;
          if (sh) { sh.uniforms.uHit.value.copy(h.point); sh.uniforms.uPulse.value = 1.0; }
        });
      }

      // UI hover flags
      if (allUiMeshes.includes(o)) {
        o.userData.hover = 1;
      //  if (o.userData.kind === 'project') hoverAnyProjectItem = true;
      //  if (o.userData.key === 'Projects')  hoverProjects = true;
      //  if (o.userData.key === 'Contact')  hoverContacts = true;
      }
    }

    // Accordion logic (open if over Projects or any project item; close if over Contacts)
    //const wantOpen  = hoverProjects || hoverAnyProjectItem;
    //const wantClose = hoverContacts && !hoverProjects && !hoverAnyProjectItem;

    // tiny hysteresis so it doesn’t flicker when crossing gaps
    //if (wantOpen && !isProjectsOpen) { isProjectsOpen = true;  layoutAccordion(true); }
    //if (wantClose && isProjectsOpen) { isProjectsOpen = false; layoutAccordion(true); }

  }

  function onPointerDown(e) {
    const rect = renderer.domElement.getBoundingClientRect();
    const nx = ((e.clientX - rect.left) / rect.width) * 2 - 1;
    const ny = -((e.clientY - rect.top) / rect.height) * 2 + 1;
    mouse.set(nx, ny);
    raycaster.setFromCamera(mouse, camera);

    const hits = raycaster.intersectObjects([cube, ...allUiMeshes], true);
    if (hits.length) {
      const hit = hits[0];
      if (hit.object === cube) {
        showPersonInfo(hit);
        return;
      }

      const obj = hit.object;
      const which = obj.userData.key;
      const kind  = obj.userData.kind || 'link';
      if (kind === 'link' && which === 'Contact') {
        if (isProjectsOpen) { isProjectsOpen = false; layoutAccordion(true); }
      }
      openPanel(which, kind);
      // menu navigate
      if (kind === 'link') {
        if (which === 'Home')    gotoByName('Home');          // virus (room)
        if (which === 'Projects'){
          isProjectsOpen = !isProjectsOpen;
          layoutAccordion(true);
          gotoByName('Projects');                         // projects scene = sphere
        }
        if (which === 'People')  gotoByName('People');    // people scene = cube
        if (which === 'Contact') gotoByName('Contact');     // contact scene = particles
      }

      // clicking a project submenu item stays on sphere but opens its panel
      if (kind === 'project') {
        // already handled by openPanel(which, 'project')
        if (!isProjectsOpen) { isProjectsOpen = true; layoutAccordion(true); }
        gotoByName('Projects');
      }
    }
  }

  function openPanel(which, kind='link') {
    const linkContents = {
      Home: 'Welcome!',
      People:   'Collaborators, mentors, and lab friends. Always hiring curious minds.',
      Projects: 'Open-source tools and interactive experiences for mesoscale biology.',
      Contacts: 'Say hello: autin@scripps.edu — let’s build weird & useful things.'
    };
    if (which === 'Contact') {
      // remove any 3D panel if present
      if (panelMesh) {
        panelMesh.parent?.remove(panelMesh);
        panelMesh.geometry?.dispose?.();
        panelMesh.material?.map?.dispose?.();
        panelMesh.material?.dispose?.();
        panelMesh = null;
      }
      showContactOverlay();
      return;
    }
    hideContactOverlay(); // for all other panels
    const projectBlurb = (name) => ({
      title: name,
      body:
  `About ${name}
  — Short summary: (fill me in)
  — Status: active
  — Links: (paper/site/code)
  — Notes: reach out if you want to try it`
    });

    // remove any existing panel (parented to camera)
    if (panelMesh) {
      panelMesh.parent?.remove(panelMesh);
      panelMesh.geometry?.dispose?.();
      panelMesh.material?.map?.dispose?.();
      panelMesh.material?.dispose?.();
      panelMesh = null;
    }

    // decide content
    let title, body;
    if (kind === 'project') {
      const p = projectBlurb(which);
      title = p.title;
      body  = p.body;
    } else {
      title = which;
      body  = linkContents[which] || '';
    }

    panelMesh = makePanelMesh(title, body, { width: 2.0 });
    camera.add(panelMesh);

    // left column under title
    panelMesh.position.set(-1.2 * uiScale, 0.3 * uiScale, -2.0);
    panelMesh.scale.setScalar(uiScale);

    // ALWAYS on top
    panelMesh.renderOrder = 20;
    panelMesh.material.depthTest  = false;
    panelMesh.material.depthWrite = false;

    panelMesh.visible = true;

    anchorUI(true); // reflow left/right after panel creation
  }

  function showPersonInfo(hit) {
    // determine which face of the cube was clicked
    const idx = Math.floor((hit.faceIndex || 0) / 2);
    const info = people[idx % people.length];
    if (!info) return;

    // cleanup existing
    if (personPanel) {
      personPanel.parent?.remove(personPanel);
      personPanel.geometry?.dispose?.();
      personPanel.material?.map?.dispose?.();
      personPanel.material?.dispose?.();
      personPanel = null;
    }
    if (personLine) {
      personLine.parent?.remove(personLine);
      personLine.geometry?.dispose?.();
      personLine.material?.dispose?.();
      personLine = null;
    }

    // panel
    personPanel = makePanelMesh(info.name, info.desc, { width: 2.2 });

    const start = hit.point.clone();
    const normal = hit.face.normal.clone();
    const worldNormal = normal.transformDirection(cube.matrixWorld).normalize();
    const end = start.clone().addScaledVector(worldNormal, 1.2);

    personPanel.position.copy(end);
    personPanel.lookAt(camera.position);
    sections.people.add(personPanel);
    personPanel.scale.set(0.001,0.001,0.001);
    gsap.to(personPanel.scale, { x:1, y:1, z:1, duration:0.5, ease:'power2.out' });

    // line
    const lineGeo = new THREE.BufferGeometry().setFromPoints([
      new THREE.Vector3(0,0,0),
      end.clone().sub(start)
    ]);
    const lineMat = new THREE.LineBasicMaterial({ color: 0x0f1115 });
    personLine = new THREE.Line(lineGeo, lineMat);
    personLine.position.copy(start);
    personLine.scale.set(0,0,0);
    sections.people.add(personLine);
    gsap.to(personLine.scale, { x:1, y:1, z:1, duration:0.5, ease:'power2.out' });
  }

  // ---------- PARTICLES as imposters (shader points) ----------
  function createParticles() {
    const N = 9000;

    const positions = new Float32Array(N * 3);
    particleVel = new Float32Array(N * 3);

    const tmp = new THREE.Vector3();
    for (let i = 0; i < N; i++) {
      // start around the attractor in a sphere
      tmp.set(Math.random()*2-1, Math.random()*2-1, Math.random()*2-1).normalize();
      tmp.multiplyScalar(Math.cbrt(Math.random()) * particleRadius * 0.95).add(particleCenter);
      positions[3*i] = tmp.x; positions[3*i+1] = tmp.y; positions[3*i+2] = tmp.z;

      // small random vel
      particleVel[3*i]   = (Math.random()*2 - 1) * 0.25;
      particleVel[3*i+1] = (Math.random()*2 - 1) * 0.25;
      particleVel[3*i+2] = (Math.random()*2 - 1) * 0.25;
    }

    particleGeom = new THREE.BufferGeometry();
    particleGeom.setAttribute('position', new THREE.BufferAttribute(positions, 3));

    // ShaderMaterial for soft round imposters (gl_PointCoord)
    const vertex = `
      uniform float uSize;
      uniform float uScale;
      void main() {
        vec4 mvPosition = modelViewMatrix * vec4(position, 1.0);
        float ps = uSize * uScale / max(1.0, -mvPosition.z);
        gl_PointSize = clamp(ps, 2.0, 200.0);
        gl_Position = projectionMatrix * mvPosition;
      }
    `;
    const fragment = `
      uniform vec3 uColor;
      void main() {
        vec2 uv = gl_PointCoord * 2.0 - 1.0;
        float d = dot(uv, uv);
        if (d > 1.0) discard;
        float edge = smoothstep(0.6, 1.0, d);   // soft edge
        float core = 1.0 - edge;
        gl_FragColor = vec4(uColor, core);      // brighter core
      }
    `;
    particleMaterial = new THREE.ShaderMaterial({
      uniforms: {
        uSize: { value: 18.0 },             // base size (px)
        uScale: { value: window.devicePixelRatio * 1.0 },
        uColor: { value: new THREE.Color(0x9fdcff) } // light blue
      },
      vertexShader: vertex,
      fragmentShader: fragment,
      transparent: true,
      depthWrite: false,
      blending: THREE.AdditiveBlending
    });

    particlePoints = new THREE.Points(particleGeom, particleMaterial);
    particlePoints.visible = false;
    particlePoints.frustumCulled = false; // avoid culling if the center is near the edge

    // invisible collider for mouse impulses
    const colGeo = new THREE.SphereGeometry(particleRadius, 32, 16);
    const colMat = new THREE.MeshBasicMaterial({ transparent: true, opacity: 0.0 });
    particleCollider = new THREE.Mesh(colGeo, colMat);
    particleCollider.position.copy(particleCenter);
    particleCollider.renderOrder = -1;

    sections.contact.add(particlePoints);
    sections.contact.add(particleCollider);
  }

  function disturbParticles(hitPointWorld) {
    if (!particleGeom) return;
    // convert to local space of particlePoints
    const hitLocal = particlePoints.worldToLocal(hitPointWorld.clone());

    const pos = particleGeom.attributes.position.array;
    const N = pos.length / 3;
    const influence = 0.9, strength = 3.2;

    for (let i = 0; i < N; i++) {
      const ix = 3*i;
      const dx = pos[ix]   - hitLocal.x;
      const dy = pos[ix+1] - hitLocal.y;
      const dz = pos[ix+2] - hitLocal.z;
      const d2 = dx*dx + dy*dy + dz*dz;
      if (d2 < influence*influence) {
        const d = Math.sqrt(Math.max(1e-8, d2));
        const s = strength * (1.0 - d / influence);
        particleVel[ix]   += (dx/d) * s;
        particleVel[ix+1] += (dy/d) * s;
        particleVel[ix+2] += (dz/d) * s;
      }
    }
  }

  function updateParticles(dt) {
    if (!particleGeom) return;
    const pos = particleGeom.attributes.position.array;
    const N = pos.length / 3;
    const damping = 0.993;

    const cx = particleCenter.x, cy = particleCenter.y, cz = particleCenter.z;
    const radius = particleRadius;

    for (let i = 0; i < N; i++) {
      const ix = 3*i;

      // gravity toward sphere center
      const dx = cx - pos[ix];
      const dy = cy - pos[ix+1];
      const dz = cz - pos[ix+2];
      const dist = Math.sqrt(dx*dx + dy*dy + dz*dz) + 1e-6;
      const g = 2.0; // attraction strength
      particleVel[ix]   += (dx/dist) * g * dt;
      particleVel[ix+1] += (dy/dist) * g * dt;
      particleVel[ix+2] += (dz/dist) * g * dt;

      // integrate
      pos[ix]   += particleVel[ix]   * dt;
      pos[ix+1] += particleVel[ix+1] * dt;
      pos[ix+2] += particleVel[ix+2] * dt;

      // soft boundary reflect to keep them in a sphere volume
      const rx = pos[ix] - cx, ry = pos[ix+1] - cy, rz = pos[ix+2] - cz;
      const rDist = Math.sqrt(rx*rx + ry*ry + rz*rz);
      if (rDist > radius) {
        const nx = rx / rDist, ny = ry / rDist, nz = rz / rDist;
        pos[ix]   = cx + nx * radius * 0.999;
        pos[ix+1] = cy + ny * radius * 0.999;
        pos[ix+2] = cz + nz * radius * 0.999;
        const vxn = particleVel[ix]*nx + particleVel[ix+1]*ny + particleVel[ix+2]*nz;
        particleVel[ix]   -= 2 * vxn * nx;
        particleVel[ix+1] -= 2 * vxn * ny;
        particleVel[ix+2] -= 2 * vxn * nz;
      }

      // damping
      particleVel[ix]   *= damping;
      particleVel[ix+1] *= damping;
      particleVel[ix+2] *= damping;
    }

    particleGeom.attributes.position.needsUpdate = true;
  }

  // ---------- GLB loader (scale 0.01, clone + ripple every mesh/material, shadows) ----------
  function loadRoom() {
    return new Promise((resolve, reject) => {
    const loader = new GLTFLoader();
    loader.load(
      './assets/polio_plane.glb',
      (gltf) => {
        room = gltf.scene;
        room.visible = true;

        // scale 1/100
        room.scale.setScalar(0.01);

        sections.home.add(room);

        // clone materials so each submesh gets ripple uniforms (works with shared + vertex colors)
        room.traverse((o) => {
          if (o.isMesh) {
            o.castShadow = true;
            o.receiveShadow = true;
            cloneAndRippleMesh(o, { color: 0x66ccff });
          }
        });
        // bbox after scaling for camera target height
        const box = new THREE.Box3().setFromObject(room);
        roomGroundY = box.min.y;

        // store better local offsets on the room group
        sections.home.userData.localCam  = new THREE.Vector3(2.2, roomGroundY + 1.6, 4.8);
        sections.home.userData.localLook = new THREE.Vector3(0,   roomGroundY + 0.5, 0);

        // if we’re currently on Room, retween to the precise ground-aware camera
        if (idx === 0) gotoScene(0);
        roomLoaded = true;
      },
      undefined,
      (err) => console.warn('GLB load error (optional room):', err?.message || err)
    );
    resolve();
    }, undefined, undefined);
  }

  // ---------- init / animate ----------
  async function init() {
    scene = new THREE.Scene();
    // --- section groups (vertical stack) ---
    const SECTION_GAP = 6; // space between sections
    sections = {
      home:      new THREE.Group(),
      people:      new THREE.Group(),
      project:    new THREE.Group(),
      contact: new THREE.Group()
    };
    scene.add(sections.home, sections.people, sections.project, sections.contact);

    // order = Room → Cube → Sphere → Particles
    ['home','project','people','contact'].forEach((key, i) => {
      sections[key].position.set(0, -i * SECTION_GAP, 0);
    });


    camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 1000);
    camera.position.copy(camBase);
    scene.add(camera); // 👈 this makes children of the camera render (title/links/panel)

    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(2, window.devicePixelRatio || 1));
    renderer.setClearColor(0xeeeeee, 1);
    renderer.outputColorSpace = THREE.SRGBColorSpace;

    // SHADOWS
    renderer.shadowMap.enabled = true;
    renderer.shadowMap.type = THREE.PCFSoftShadowMap;

    container.appendChild(renderer.domElement);

    // Lights
    const dir = new THREE.DirectionalLight(0xffffff, 1);
    dir.position.set(3, 6, 5);
    dir.castShadow = true;
    dir.shadow.mapSize.set(2048, 2048);
    dir.shadow.camera.near = 0.1;
    dir.shadow.camera.far = 50;
    dir.shadow.camera.left = -10;
    dir.shadow.camera.right = 10;
    dir.shadow.camera.top = 10;
    dir.shadow.camera.bottom = -10;
    scene.add(dir);

    scene.add(new THREE.AmbientLight(0xffffff, 0.45));

    // --- CUBE (PEOPLE) ---
    cube = new THREE.Mesh(
      new THREE.BoxGeometry(1,1,1),
      new THREE.MeshStandardMaterial({ color: 0x0077ff, roughness: 0.35, metalness: 0.1 })
    );
    cube.castShadow = true;
    cube.receiveShadow = true;
    sections.people.add(cube);
    // load portraits and apply to each face
    loadPeopleMaterials((mats) => {
      cube.material = mats;                 // 6 materials = 6 faces
      // already patched via patchMaterialForRipple inside the loader
    });
    // EXPLICIT ripple on cube too
    cloneAndRippleMesh(cube, { color: 0x5cc0ff });

    // sphere
    sphere = new THREE.Mesh(
      new THREE.SphereGeometry(0.9, 48, 48),
      new THREE.MeshStandardMaterial({ color: 0xff7700, roughness: 0.4, metalness: 0.15 })
    );
    sphere.position.set(0, 0, 0);
    sphere.castShadow = true;
    sphere.receiveShadow = true;
    sections.project.add(sphere);
    cloneAndRippleMesh(sphere, { color: 0xffa366 });

    setTimeout(() => {
      [cube, sphere].forEach((mesh) => {
        const mats = Array.isArray(mesh.material) ? mesh.material : [mesh.material];
        mats.forEach(m => {
          const sh = m.userData?.shader;
          if (sh) { sh.uniforms.uHit.value.copy(mesh.position); sh.uniforms.uPulse.value = 1.0; }
        });
      });
    }, 250);

    // PARTICLES (imposters) + collider
    createParticles();

    // 3D UI for Cube scene (attached to camera)
    titleMesh = makeTextMesh("Ludo's Lab", { fontSize: 72, color: '#0f1115' });
    camera.add(titleMesh);
    titleMesh.position.set(-1.5, 1.0, -2.0);
    titleMesh.visible = true;

    const items = ['Home', 'Projects', 'People', 'Contact'];
    items.forEach((label, i) => {
      const m = makeTextMesh(label, { fontSize: 48, color: '#0f1115', bg: 'rgba(255,255,255,0.65)' });
      m.userData.key = label;
      m.userData.hover = 0; 
      m.userData.basePos = new THREE.Vector3(1.2, 0.9 - i * 0.4, -2.0); // ← base
      camera.add(m);
      m.position.copy(m.userData.basePos);
      m.visible = true;
      // give them a tiny ripple too
      cloneAndRippleMesh(m, { color: 0x66ccff, fresnel: 0.6, falloff: 4.0 });
      linkMeshes.push(m);
    });

    // --- Right column: project list below the main links ---
    const startY = 0.2;      // below “Contacts”
    const stepY  = 0.28;

    projects.forEach((label, i) => {
      const m = makeTextMesh(label, {
        fontSize: 36,
        color: '#0f1115',
        bg: 'rgba(255,255,255,0.5)'
      });
      m.userData.key   = label;
      m.userData.kind  = 'project';              // 👈 identify as project
      m.userData.hover = 0;
      // m.userData.basePos = new THREE.Vector3(-1.2, startY - i * stepY, -2.0);
      // camera.add(m);
      // m.position.copy(m.userData.basePos);
      sections.project.add(m);
      m.position.set(0, 0, 0); // will be arranged on the ring
      
      m.scale.setScalar(uiScale);
      m.visible = false;

      // ripple on hover
      cloneAndRippleMesh(m, { color: 0x66ccff, fresnel: 0.6, falloff: 4.0 });
      projectMeshes.push(m);
    });

    // keep a union array for raycast/click/hover logic
    allUiMeshes = [...linkMeshes, ...projectMeshes];
    
    isProjectsOpen = false;
    projectMeshes.forEach((m)=>{ m.visible = false; });
    layoutAccordion(false);
    anchorUI(false);
    // room
    await loadRoom();

    sections.home.visible = true;
    cube.visible = false;
    sphere.visible = false;
    if (particlePoints) particlePoints.visible = false;
    if (particleCollider) particleCollider.visible = false;

    // start
    gotoScene(0);

    window.addEventListener('resize', onResize);
    window.addEventListener('wheel', onWheel, { passive: true });
    renderer.domElement.addEventListener('pointermove', onPointerMove, { passive: true });
    renderer.domElement.addEventListener('pointerdown', onPointerDown, { passive: true });
    renderer.domElement.addEventListener('touchstart', onTouchStart, { passive: true });
    renderer.domElement.addEventListener('touchmove', onTouchMove, { passive: true });
  
    updateResponsive(); 
  }

  function onResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
    // update particle pixel scaling
    if (particleMaterial) {
      particleMaterial.uniforms.uScale.value = window.devicePixelRatio * 1.0;
    }
    updateResponsive(); 
  }

  function layoutProjectsOrbit(radius = 1.8) {
    if (!projectMeshes.length || !sphere) return;

    // center (in the same parent space = sections.project)
    const cx = sphere.position.x;
    const cy = sphere.position.y;
    const cz = sphere.position.z;

    const n = projectMeshes.length;
    const step = (Math.PI * 2) / n;
    const phase = 0; // keep fixed; you can animate if you want
    const ringTilt = 0.0; // set >0 to slightly tilt ring (in radians)

    for (let i = 0; i < n; i++) {
      const a = phase + i * step;
      const x = cx + radius * Math.cos(a);
      const y = cy + radius * Math.sin(a);
      const z = cz + Math.sin(ringTilt) * 0.0; // flat ring; change if you want tilt

      const m = projectMeshes[i];
      m.position.set(x, y, z);
    }
  }

  function anchorUI(animate = true) {
    const { halfW, halfH } = viewportHalfAtDepth(UI_Z);

    // --- LEFT column: title + panel ---
    const place = (mesh, x, y) => {
      if (!mesh) return;
      mesh.position.z = -UI_Z;
      if (animate) {
        gsap.to(mesh.position, { x, y, duration: 0.25, ease: 'power2.out' });
      } else {
        mesh.position.set(x, y, -UI_Z);
      }
      // store anchor for wiggles
      mesh.userData.anchorPos = mesh.userData.anchorPos || new THREE.Vector3();
      mesh.userData.anchorPos.set(x, y, -UI_Z);
    };

    // Title: top-left
    if (titleMesh) {
      const { w: tw, h: th } = meshSize(titleMesh);
      const x = -halfW + UI_MARGIN + tw * 0.5;
      const y =  halfH - UI_MARGIN - th * 0.5;
      place(titleMesh, x, y);
    }

    // Panel: under title (if present)
    if (panelMesh) {
      const { w: pw, h: ph } = meshSize(panelMesh);
      // try to align its left edge to UI margin
      const px = -halfW + UI_MARGIN + pw * 0.5;
      // below title (leave a small gap)
      let py = halfH - UI_MARGIN;
      if (titleMesh) {
        const { h: th } = meshSize(titleMesh);
        py = py - th - 0.06; // gap
      }
      py = py - ph * 0.5;
      // if panel too tall for viewport, optionally hide or clamp
      panelMesh.visible = (ph + UI_MARGIN * 2) < (halfH * 2);
      if (panelMesh.visible) place(panelMesh, px, py);
    }

    // --- RIGHT column: menu links, top-right, hide overflow ---
    // We’ll stack from top-right downward with LINK_GAP
    const rightStartX =  halfW - UI_MARGIN;
    let cursorY       =  halfH - UI_MARGIN;

    // sort links in desired order (you already build as Home, Projects, People, Contact)
    const visibleLinks = [];
    for (const m of linkMeshes) {
      const { w, h } = meshSize(m);
      const nextY = cursorY - h * 0.5;
      const fits = (nextY - h * 0.5) > (-halfH + UI_MARGIN); // bottom check
      m.visible = fits;
      if (!fits) continue;

      const x = rightStartX - w * 0.5;
      const y = nextY;
      place(m, x, y);
      visibleLinks.push(m);

      cursorY = y - h * 0.5 - LINK_GAP; // move down
    }

    // If Projects toggles its submenu: satellites are positioned around sphere (handled elsewhere),
    // but if you want a fallback “right column” layout when sphere isn’t visible, you could add it here.
  }

  function animate() {
    requestAnimationFrame(animate);

    const dt = Math.min(0.033, clock.getDelta());
    const t = clock.elapsedTime;

    if (cube?.visible) cube.rotation.y += 0.003;

    // update ripple uniforms everywhere
    const bump = (mesh) => {
      if (!mesh?.material) return;
      const mats = Array.isArray(mesh.material) ? mesh.material : [mesh.material];
      mats.forEach((m) => {
        const sh = m?.userData?.shader;
        if (sh) {
          sh.uniforms.uTime.value = t;
          sh.uniforms.uPulse.value = Math.max(0, sh.uniforms.uPulse.value * 0.94);
        }
      });
    };
    bump(cube); bump(sphere);
    room?.traverse?.((o) => { if (o.isMesh) bump(o); });
    allUiMeshes.forEach(bump);
    // --- subtle hover wiggle for top-right links (around their anchorPos) ---
    linkMeshes.forEach((m) => {
      if (!m.visible || !m.userData.anchorPos) return;
      const h = m.userData.hover ? 1 : 0;
      m.userData.hoverAmt = THREE.MathUtils.damp(m.userData.hoverAmt ?? 0, h, 12, 1/60);

      const A = 0.06 * uiScale * (m.userData.hoverAmt || 0);
      const f = 6.0; // Hz-ish feel

      // wiggle along X only, around the anchored X/Y
      const ax = m.userData.anchorPos.x;
      const ay = m.userData.anchorPos.y;
      m.position.set(ax + Math.sin(clock.elapsedTime * f) * A, ay, m.userData.anchorPos.z);
    });
    if (panelMesh) bump(panelMesh);

    // update particles if visible
    if (particlePoints?.visible) updateParticles(dt);
    if (sphere?.visible) sphere.rotation.y += 0.003;

    // parallax easing
    offsetCurrent.lerp(offsetTarget, 0.08);
    camPos.copy(camBase).add(offsetCurrent);
    camera.position.copy(camPos);
    camera.lookAt(camTarget);

    // --- PROJECT SATELLITES: billboard + (optional) small scale pulse ---
    if (isProjectsOpen && projectMeshes?.length) {
      for (const m of projectMeshes) {
        if (!m.visible) continue;
        // face camera
        m.quaternion.copy(camera.quaternion);

        // OPTIONAL: tiny scale pulse on hover instead of position wiggle
        const h = m.userData.hover ? 1 : 0;
        m.userData.hoverAmt = THREE.MathUtils.damp(m.userData.hoverAmt ?? 0, h, 12, dt);
        const s = uiScale * (1.0 + 0.06 * (m.userData.hoverAmt || 0));
        m.scale.setScalar(s);
      }
    }
    // keep satellites locked to camera-facing ring while open
    followProjectsOrbit({ radius: 1.6 * uiScale, lerp: 0.14 });
    renderer.render(scene, camera);
  }

  onMount(() => {
    init();
    animate();
    const closeBtn = document.getElementById('contact-close');
    if (closeBtn) closeBtn.addEventListener('click', hideContactOverlay);
    return () => {
      window.removeEventListener('resize', onResize);
      window.removeEventListener('wheel', onWheel);
      renderer?.domElement?.removeEventListener('pointermove', onPointerMove);
      renderer?.domElement?.removeEventListener('pointerdown', onPointerDown);
      renderer?.dispose?.();
    };
  });
</script>

<style>
  :global(html,body){ margin:0; height:100%; background:#eee; color:#0f1115; }
  .stage{ width:100vw; height:100vh; overflow:hidden; }
  .ui{
    position:fixed; left:0; right:0; top:0;
    display:flex; justify-content:center; gap:16px; padding:14px;
    pointer-events:none; font-family:system-ui,-apple-system,Segoe UI,Roboto,sans-serif;
    mix-blend-mode:multiply; opacity:.9;
  }
  .chip{ pointer-events:auto; padding:6px 10px; border-radius:9999px; background:rgba(0,0,0,.06); border:1px solid rgba(0,0,0,.12); font-size:12px; }

  /* CONTACT OVERLAY */
  .contact {
    position: fixed;
    inset: 0;
    display: grid;
    place-items: center;
    z-index: 20;
    pointer-events: none;
  }
  .contact.hidden { display: none; }

  .contact .card {
    width: min(900px, 94vw);
    max-height: min(90vh, 1100px);
    overflow: auto;
    border-radius: 20px;
    background: rgba(255,255,255,0.97);
    box-shadow: 0 20px 60px rgba(0,0,0,.18);
    padding: clamp(16px, 2.5vw, 28px);
    pointer-events: auto;
    font-family: system-ui, -apple-system, Segoe UI, Roboto, sans-serif;
    color: #0f1115;
  }

  .contact .top {
    display: grid;
    grid-template-columns: 1fr auto;
    gap: 16px;
    align-items: center;
    margin-bottom: 10px;
  }
  .contact .top .title h2 { margin: 0 0 4px 0; font-size: clamp(20px, 2.2vw, 28px); }
  .contact .top .title .affil { margin: 0; opacity: .85; font-size: clamp(14px, 1.6vw, 16px); }
  .contact .top .logo img, .contact .top .logo span { height: 36px; display: inline-block; }

  .contact .rows { display: grid; gap: 10px; margin: 10px 0 14px; }
  .contact .row { display: grid; grid-template-columns: 120px 1fr; gap: 12px; align-items: start; }
  .contact .label { font-weight: 700; opacity: .8; }
  .contact .value .name { font-weight: 600; }
  .contact .value a { color: #0a66c2; text-decoration: none; }
  .contact .value a:hover { text-decoration: underline; }

  .contact .mapwrap {
    margin-top: 8px;
    border-radius: 14px;
    overflow: hidden;
    border: 1px solid rgba(0,0,0,.08);
  }
  .contact .mapwrap iframe {
    display: block;
    width: 100%;
    height: min(56vh, 520px);
    border: 0;
  }

  .contact .social {
    display: flex;
    gap: 12px;
    margin-top: 12px;
  }
  .contact .icon {
    width: 40px; height: 40px; border-radius: 999px;
    background: #0f1115; color: #fff; display: grid; place-items: center;
    text-decoration: none; transition: transform .15s ease, background .15s ease;
}
.contact .icon:hover { transform: translateY(-1px); background: #0a66c2; }

.contact .close {
  margin-top: 14px;
  border: 0; border-radius: 999px;
  padding: 10px 16px; font-weight: 600;
  background: #0f1115; color: #fff; cursor: pointer;
}

@media (max-width: 560px) {
  .contact .row { grid-template-columns: 1fr; }
  .contact .label { opacity: .6; }
}

</style>

<div class="stage" bind:this={container}></div>

<div class="ui">
  <div class="chip">Scroll to navigate</div>
  <div class="chip">{roomLoaded ? ['Home','Projects','People','Contact'][idx] : ['Projects','People','Contact'][idx]}</div>
</div>

<!-- CONTACT OVERLAY -->
<div id="contact-overlay" class="contact hidden" aria-hidden="true">
  <div class="card">
    <header class="top">
      <div class="title">
        <h2>Ludo’s Lab — Contact</h2>
        <p class="affil">
          Department of Integrative Structural and Computational Biology at Scripps Research
        </p>
      </div>
      <div class="logo">
        <img
          src="./assets/images/logo_scripps.png"
          alt="Scripps Research"
          on:error={(e) => {
            const span = document.createElement('span');
            span.textContent = 'Scripps Research';
            e.currentTarget.replaceWith(span);
          }}
        />
      </div>
    </header>

    <div class="rows">
      <div class="row">
        <div class="label">PI</div>
        <div class="value">
          <div class="name">Ludovic Autin</div>
          <a href="mailto:autin@scripps.edu">autin@scripps.edu</a>
        </div>
      </div>

      <div class="row">
        <div class="label">Admin</div>
        <div class="value">
          <div class="name">Michelle Wilson</div>
          <a href="mailto:michelle@scripps.edu">michelle@scripps.edu</a>
        </div>
      </div>

      <div class="row">
        <div class="label">Address</div>
        <div class="value">
          <div>10550 North Torrey Pines Rd</div>
          <div>La Jolla, CA 92037</div>
        </div>
      </div>
    </div>

    <div class="mapwrap" aria-label="Map to Scripps Research">
      <iframe
        title="Scripps Research Map"
        loading="lazy"
        referrerpolicy="no-referrer-when-downgrade"
        src="https://www.google.com/maps?q=10550+N+Torrey+Pines+Rd,+La+Jolla,+CA+92037&output=embed">
      </iframe>
    </div>

    <div class="social">
      <a class="icon" href="https://x.com/autinlab" target="_blank" rel="noreferrer" aria-label="X">
        <!-- X logo -->
        <svg viewBox="0 0 24 24" width="22" height="22" aria-hidden="true"><path d="M18.244 2H21l-6.51 7.44L22 22h-6.9l-4.54-5.9L4.5 22H2l6.98-7.98L2 2h6.9l4.2 5.46L18.244 2Zm-2.4 18h2.41L8.27 4h-2.5l10.07 16Z" fill="currentColor"/></svg>
      </a>
      <a class="icon" href="https://github.com/autinlab" target="_blank" rel="noreferrer" aria-label="GitHub">
        <svg viewBox="0 0 24 24" width="22" height="22" aria-hidden="true"><path d="M12 .5a12 12 0 0 0-3.79 23.4c.6.11.82-.26.82-.58v-2.03c-3.34.73-4.04-1.41-4.04-1.41-.55-1.39-1.35-1.76-1.35-1.76-1.1-.75.08-.74.08-.74 1.22.09 1.86 1.25 1.86 1.25 1.08 1.85 2.83 1.31 3.52 1 .11-.8.42-1.31.76-1.61-2.66-.3-5.46-1.33-5.46-5.92 0-1.31.47-2.39 1.24-3.23-.12-.3-.54-1.52.12-3.17 0 0 1.01-.32 3.3 1.23a11.5 11.5 0 0 1 6 0c2.3-1.55 3.3-1.23 3.3-1.23.66 1.65.24 2.87.12 3.17.77.84 1.24 1.92 1.24 3.23 0 4.6-2.8 5.61-5.47 5.91.43.37.81 1.1.81 2.22v3.3c0 .32.21.69.82.58A12 12 0 0 0 12 .5Z" fill="currentColor"/></svg>
      </a>
      <a class="icon" href="https://bsky.app/profile/autinlab.bsky.social" target="_blank" rel="noreferrer" aria-label="Bluesky">
        <svg viewBox="0 0 512 512" width="22" height="22" aria-hidden="true"><path d="M256 256c62-101 140-173 182-197 26-15 54-22 74-10 19 11 14 44-2 79-26 57-88 126-139 165 71-13 141-29 157 12 9 22-3 49-19 67-46 52-129-2-179-41 9 51 18 113-4 139-17 21-47 24-70 0-22-23-22-85-13-139-51 39-133 93-179 41-16-18-28-45-19-67 16-41 86-25 157-12-51-39-113-108-139-165-16-35-21-68-2-79 20-12 48-5 74 10 42 24 120 96 182 197Z" fill="currentColor"/></svg>
      </a>
    </div>

    <button class="close" id="contact-close">Close</button>
  </div>
</div>
