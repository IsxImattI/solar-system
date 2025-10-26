<script lang="ts">
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { celestialBodies } from '$lib/data/celestialBodies.js';
  import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';


  let container: HTMLDivElement | undefined;
  let selectedPlanet: any = null;
  let controls: OrbitControls;
  let camera: THREE.PerspectiveCamera;
  let targetCameraPosition = new THREE.Vector3();
  let targetLookAt = new THREE.Vector3();
  let isAnimatingCamera = false;
  let isDragging = false;
  let mouseDownTime = 0;
  let isPaused = false;
  let timeSpeed = 1;
  let showWelcome = true;
  let showSidebar = false;
  let planets: THREE.Mesh[] = [];
  let sidebarTimeout: any = null;
  let isMobile = false;

  const keyState: { [key: string]: boolean } = {};
  const moveSpeed = 2;
  const cameraVelocity = new THREE.Vector3();
  const cameraDamping = 0.9;

  function handleSidebarMouseEnter() {
    if (sidebarTimeout) clearTimeout(sidebarTimeout);
    showSidebar = true;
 }

 function handleSidebarMouseLeave() {
  sidebarTimeout = setTimeout(() => {
    showSidebar = false;
    }, 300); 
  }

  function onMouseDown() {
    isDragging = false;
    mouseDownTime = Date.now();
  }

  function onMouseMove() {
    isDragging = true;
  }

  function toggleSidebar() {
    if (isMobile) {
      showSidebar = !showSidebar;
    }
  }

  function onKeyDown(event: KeyboardEvent) {
    const key = event.key.toLowerCase();
    if (['w', 'a', 's', 'd'].includes(key)) {
      keyState[key] = true;
      event.preventDefault();
    }
  }

  function onKeyUp(event: KeyboardEvent) {
    const key = event.key.toLowerCase();
    if (['w', 'a', 's', 'd'].includes(key)) {
      keyState[key] = false;
      event.preventDefault();
    }
  }

  onMount(() => {

    // check if mobile
    isMobile = window.innerWidth <= 768;

    window.addEventListener('resize', () => {
      isMobile = window.innerWidth <= 768;
    });
    // scene setup
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x000511);

    // camera setup
    camera = new THREE.PerspectiveCamera(
      75,
      window.innerWidth / window.innerHeight,
      0.1,
      1000
    );
    camera.position.set(0, 150, 200);
    camera.lookAt(0, 0, 0);

    // renderer setup
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    container?.appendChild(renderer.domElement);

    // orbit controls
    controls = new OrbitControls(camera, renderer.domElement);
    controls.enableDamping = true;
    controls.dampingFactor = 0.05;
    controls.minDistance = 30;
    controls.maxDistance = 800;
    controls.enablePan = true;

    // lighting setup
    const ambientLight = new THREE.AmbientLight(0xffffff, 0.8); 
    scene.add(ambientLight);

    const pointLight = new THREE.PointLight(0xffffff, 3, 1000);
    pointLight.position.set(0, 0, 0);
    scene.add(pointLight);

    // additional lights for better visibility
    const fillLight1 = new THREE.DirectionalLight(0xffffff, 0.5);
    fillLight1.position.set(100, 100, 100);
    scene.add(fillLight1);

    const fillLight2 = new THREE.DirectionalLight(0xffffff, 0.3);
    fillLight2.position.set(-100, -100, -100);
    scene.add(fillLight2);

    // stars background
    const starsGeometry = new THREE.BufferGeometry();
    const starsMaterial = new THREE.PointsMaterial({ color: 0xffffff, size: 0.7 });
    const starsVertices: number[] = [];
    for (let i = 0; i < 10000; i++) {
      const x = (Math.random() - 0.5) * 2000;
      const y = (Math.random() - 0.5) * 2000;
      const z = (Math.random() - 0.5) * 2000;
      starsVertices.push(x, y, z);
    }
    starsGeometry.setAttribute('position', new THREE.Float32BufferAttribute(starsVertices, 3));
    const stars = new THREE.Points(starsGeometry, starsMaterial);
    scene.add(stars);

    // create celestial bodies
    const textureLoader = new THREE.TextureLoader();

    Object.entries(celestialBodies).forEach(([key, body]) => {
      const geometry = new THREE.SphereGeometry(body.radius, 64, 64);
    
      // load texture if available
      const material = body.texture 
        ? new THREE.MeshStandardMaterial({
            map: textureLoader.load(body.texture),
            emissive: (body as any).emissive || 0x000000,
            emissiveIntensity: (body as any).emissiveIntensity || 0
          })
        : new THREE.MeshStandardMaterial({
            color: body.color,
            emissive: (body as any).emissive || 0x000000,
            emissiveIntensity: (body as any).emissiveIntensity || 0
          });
      
      const mesh = new THREE.Mesh(geometry, material);

      // position planets
      mesh.position.x = body.distance;
      mesh.userData = { ...body, key, angle: Math.random() * Math.PI * 2 };

      scene.add(mesh);
      planets.push(mesh);

      // add rings for Saturn
      if ((body as any).hasRings) {
        const ringGeometry = new THREE.RingGeometry(body.radius * 1.5, body.radius * 2.5, 64);
        const ringMaterial = (body as any).ringTexture
          ? new THREE.MeshBasicMaterial({
              map: textureLoader.load((body as any).ringTexture),
              side: THREE.DoubleSide,
              transparent: true,
              opacity: 0.8
            })
          : new THREE.MeshBasicMaterial({
              color: 0xc9b291,
              side: THREE.DoubleSide,
              transparent: true,
              opacity: 0.6
            });
        const ring = new THREE.Mesh(ringGeometry, ringMaterial);
        ring.rotation.x = Math.PI / 2;
        mesh.add(ring);
      }

      // add orbit line
      if (body.distance > 0) {
        const orbitGeometry = new THREE.BufferGeometry();
        const orbitPoints: number[] = [];
        for (let i = 0; i <= 64; i++) {
          const angle = (i / 64) * Math.PI * 2;
          orbitPoints.push(
            Math.cos(angle) * body.distance,
            0,
            Math.sin(angle) * body.distance
          );
        }
        orbitGeometry.setAttribute('position', new THREE.Float32BufferAttribute(orbitPoints, 3));
        const orbitMaterial = new THREE.LineBasicMaterial({ 
          color: 0x444444, 
          transparent: true, 
          opacity: 0.3 
        });
        const orbit = new THREE.Line(orbitGeometry, orbitMaterial);
        scene.add(orbit);
      }
    });

    // raycaster for click detection
    const raycaster = new THREE.Raycaster();
    const mouse = new THREE.Vector2();

    function onMouseClick(event: MouseEvent) {
      // ignore clicks that were actually drags
      const clickDuration = Date.now() - mouseDownTime;
      if (isDragging || clickDuration > 200) return;

      mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
      mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

      raycaster.setFromCamera(mouse, camera);
      const intersects = raycaster.intersectObjects(planets);

      if (intersects.length > 0) {
        const clickedPlanet = intersects[0].object as THREE.Mesh;
        selectedPlanet = clickedPlanet.userData;

        // set target for smooth camera transition
        const targetPos = clickedPlanet.position.clone();
        const distance = clickedPlanet.userData.radius * 4;

        targetCameraPosition.set(
          targetPos.x + distance,
          targetPos.y + distance * 0.5,
          targetPos.z + distance
        );
        targetLookAt.copy(targetPos);
        isAnimatingCamera = true;

        // disable controls temporarily during animation
        controls.enabled = false;
      } else {
        selectedPlanet = null;
        // return to overview
        targetCameraPosition.set(0, 150, 200);
        targetLookAt.set(0, 0, 0);
        isAnimatingCamera = true;
        controls.enabled = false;
      }
    }

    window.addEventListener('click', onMouseClick);
    window.addEventListener('mousedown', onMouseDown);
    window.addEventListener('mousemove', onMouseMove);
    window.addEventListener('keydown', onKeyDown);
    window.addEventListener('keyup', onKeyUp);

    function updateCameraMovement() {
      const moveDirection = new THREE.Vector3();

      if (keyState['w']) moveDirection.z -= 1;
      if (keyState['s']) moveDirection.z += 1;
      if (keyState['a']) moveDirection.x -= 1;
      if (keyState['d']) moveDirection.x += 1;

      if (moveDirection.length() > 0) {
        moveDirection.normalize();

        const cameraDirection = new THREE.Vector3();
        camera.getWorldDirection(cameraDirection);

        const right = new THREE.Vector3();
        right.crossVectors(cameraDirection, camera.up).normalize();

        const forward = new THREE.Vector3();
        forward.crossVectors(camera.up, right).normalize();

        const movement = new THREE.Vector3();
        movement.addScaledVector(right, moveDirection.x * moveSpeed);
        movement.addScaledVector(forward, -moveDirection.z * moveSpeed);

        cameraVelocity.add(movement);
      }

      cameraVelocity.multiplyScalar(cameraDamping);

      if (cameraVelocity.length() > 0.01) {
        camera.position.add(cameraVelocity);
        controls.target.add(cameraVelocity);

        if (isAnimatingCamera && (keyState['w'] || keyState['a'] || keyState['s'] || keyState['d'])) {
          isAnimatingCamera = false;
          controls.enabled = true;
        }
      } else {
        cameraVelocity.set(0, 0, 0);
      }
    }

    // handle window resize
    function onWindowResize() {
      camera.aspect = window.innerWidth / window.innerHeight;
      camera.updateProjectionMatrix();
      renderer.setSize(window.innerWidth, window.innerHeight);
    }
    window.addEventListener('resize', onWindowResize);

    // animation loop
    function animate() {
      requestAnimationFrame(animate);

      // rotate and orbit planets
      if (!isPaused) {
        planets.forEach(planet => {
            const data = planet.userData;

            planet.rotation.y += data.rotationSpeed * timeSpeed;

            if (data.distance > 0) {
                data.angle += data.orbitSpeed * timeSpeed;
                planet.position.x = Math.cos(data.angle) * data.distance;
                planet.position.z = Math.sin(data.angle) * data.distance;
            }
        });
      }

      // smooth camera animation
      if (isAnimatingCamera) {
        camera.position.lerp(targetCameraPosition, 0.05);

        const currentLookAt = new THREE.Vector3();
        camera.getWorldDirection(currentLookAt);
        currentLookAt.multiplyScalar(10).add(camera.position);
        currentLookAt.lerp(targetLookAt, 0.05);
        camera.lookAt(currentLookAt);

        // check if animation is complete
        if (camera.position.distanceTo(targetCameraPosition) < 0.5) {
          isAnimatingCamera = false;
          controls.enabled = true;
          controls.target.copy(targetLookAt);
        }
      }

      // if a planet is selected, update camera target to follow it
      if (selectedPlanet && !isAnimatingCamera) {
        const selectedMesh = planets.find(p => p.userData.key === selectedPlanet.key);
        if (selectedMesh) {
          controls.target.copy(selectedMesh.position);
        }
      }

      controls.update();
      updateCameraMovement();
      renderer.render(scene, camera);
    }

    animate();

    // cleanup on component unmount
    return () => {
      window.removeEventListener('mousedown', onMouseDown);
      window.removeEventListener('mousemove', onMouseMove);
      window.removeEventListener('click', onMouseClick);
      window.removeEventListener('resize', onWindowResize);
      window.removeEventListener('keydown', onKeyDown);
      window.removeEventListener('keyup', onKeyUp);
      controls.dispose();
      renderer.dispose();
    };
  });
</script>

<div bind:this={container} class="scene-container" class:scene-visible={!showWelcome}></div>

<!-- welcome screen -->
{#if showWelcome}
  <div class="welcome-overlay">
    <div class="welcome-content">
      <h1>🌌 Interactive Solar System</h1>
      <p class="tagline">Explore our cosmic neighborhood in 3D</p>
      
      <div class="welcome-features">
        <div class="feature">
          <span class="icon">🪐</span>
          <h3>Realistic Planets</h3>
          <p>High-quality textures from NASA imagery</p>
        </div>
        <div class="feature">
          <span class="icon">🎮</span>
          <h3>Interactive Controls</h3>
          <p>Click, drag, zoom, and explore freely</p>
        </div>
        <div class="feature">
          <span class="icon">⏱️</span>
          <h3>Time Control</h3>
          <p>Speed up or slow down orbital motion</p>
        </div>
      </div>

      <button class="start-btn" on:click={() => showWelcome = false}>
        Start Exploring →
      </button>

      <div class="credits">
        <p>Built with SvelteKit & Three.js</p>
        <p>Planet textures from <a href="https://planetpixelemporium.com" target="_blank">Planet Pixel Emporium</a></p>
      </div>
    </div>
  </div>
{/if}

<!-- sidebar toggle button -->
<button class="sidebar-toggle" on:mouseenter={!isMobile ? handleSidebarMouseEnter : undefined} on:click={toggleSidebar}>
  {showSidebar ? '✕' : 'ℹ️'}
</button>
<!-- sidebar panel (always rendered, no {#if}) -->
<div 
  class="sidebar" 
  class:sidebar-open={showSidebar}
  role="complementary" 
  on:mouseenter={!isMobile ? handleSidebarMouseEnter : undefined}
  on:mouseleave={!isMobile ? handleSidebarMouseLeave : undefined}
>
  <h2>About This Project</h2>
  
  <div class="sidebar-section">
    <h3>📖 Description</h3>
    <p>An interactive 3D visualization of our solar system built with modern web technologies.</p>
  </div>

  <div class="sidebar-section">
    <h3>🎯 Features</h3>
    <ul>
      <li>Real planetary textures</li>
      <li>Smooth camera controls</li>
      <li>Time manipulation (0.5x - 10x speed)</li>
      <li>Detailed planet information</li>
      <li>Realistic orbital mechanics</li>
    </ul>
  </div>

  <div class="sidebar-section">
    <h3>🛠️ Tech Stack</h3>
    <div class="tech-badges">
      <span class="badge">SvelteKit</span>
      <span class="badge">Three.js</span>
      <span class="badge">TypeScript</span>
      <span class="badge">WebGL</span>
    </div>
  </div>

  <div class="sidebar-section">
    <h3>🌍 Planets</h3>
    <div class="planet-list">
      {#each Object.entries(celestialBodies) as [key, body]}
        <button class="planet-btn" on:click={() => {
          const planet = planets.find(p => p.userData.key === key);
          if (planet) {
            selectedPlanet = planet.userData;
            const targetPos = planet.position.clone();
            const distance = planet.userData.radius * 4;
            targetCameraPosition.set(
              targetPos.x + distance,
              targetPos.y + distance * 0.5,
              targetPos.z + distance
            );
            targetLookAt.copy(targetPos);
            isAnimatingCamera = true;
            controls.enabled = false;
          }
        }}>
          {body.name}
        </button>
      {/each}
    </div>
  </div>

  <div class="sidebar-section credits-sidebar">
    <h3>👨‍💻 Created By</h3>
    <p>Matej Gyergyek</p>
    <a href="https://github.com/IsxImattI/solar-system" target="_blank" class="github-link">
      View on GitHub →
    </a>
  </div>
</div>

{#if selectedPlanet}
  <div class="info-panel">
    <h2>{selectedPlanet.name}</h2>
    <p>{selectedPlanet.description}</p>
    <ul>
      <li><strong>Type:</strong> {selectedPlanet.type}</li>
      <li><strong>Diameter:</strong> {selectedPlanet.diameter}</li>
      <li><strong>Mass:</strong> {selectedPlanet.mass}</li>
    </ul>
    <button on:click={() => selectedPlanet = null}>Close</button>
  </div>
{/if}

<div class="controls-hint">
    <p>🖱️ Left-click & drag to rotate • Scroll to zoom • Right-click & drag to pan</p>
  </div>
  <div class="time-controls">
  <button on:click={() => isPaused = !isPaused} class="play-pause-btn">
    {isPaused ? '▶️' : '⏸️'}
  </button>
  
  <div class="speed-controls">
    <button on:click={() => timeSpeed = Math.max(0.1, timeSpeed - 0.5)}>-</button>
    <span class="speed-indicator">{timeSpeed.toFixed(1)}x</span>
    <button on:click={() => timeSpeed = Math.min(10, timeSpeed + 0.5)}>+</button>
  </div>
 </div>

<style>
  .scene-container {
    width: 100vw;
    height: 100vh;
    position: fixed;
    top: 0;
    left: 0;
    visibility: hidden;
  }

  .scene-container.scene-visible {
    visibility: visible;
  }

  .info-panel {
    position: fixed;
    top: 20px;
    right: 20px;
    background: rgba(0, 5, 20, 0.9);
    color: white;
    padding: 20px;
    border-radius: 10px;
    max-width: 300px;
    border: 1px solid rgba(255, 255, 255, 0.2);
    backdrop-filter: blur(10px);
  }

  .info-panel h2 {
    margin-top: 0;
    color: #fdb813;
  }

  .info-panel ul {
    list-style: none;
    padding: 0;
    margin: 10px 0;
  }

  .info-panel li {
    margin: 8px 0;
  }

  .info-panel button {
    background: #fdb813;
    color: #000511;
    border: none;
    padding: 8px 16px;
    border-radius: 5px;
    cursor: pointer;
    font-weight: bold;
    margin-top: 10px;
  }

  .info-panel button:hover {
    background: #ffc849;
  }

  .controls-hint {
  position: fixed;
  bottom: 20px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(0, 5, 20, 0.8);
  color: white;
  padding: 10px 20px;
  border-radius: 20px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  backdrop-filter: blur(10px);
  z-index: 10;
  font-size: 0.9rem;
  }
  
  .controls-hint p {
    margin: 0;
    opacity: 0.9;
  }

  /* time controls */
.time-controls {
  position: fixed;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 15px;
  align-items: center;
  background: rgba(0, 5, 20, 0.9);
  backdrop-filter: blur(10px);
  padding: 15px 25px;
  border-radius: 50px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  z-index: 98;
}

.play-pause-btn {
  background: rgba(253, 184, 19, 0.2);
  color: white;
  border: 1px solid #fdb813;
  padding: 8px 20px;
  border-radius: 25px;
  cursor: pointer;
  font-size: 1rem;
  transition: all 0.2s;
}

.play-pause-btn:hover {
  background: rgba(253, 184, 19, 0.4);
  transform: scale(1.05);
}

.speed-controls {
  display: flex;
  gap: 10px;
  align-items: center;
}

.speed-controls button {
  background: rgba(255, 255, 255, 0.1);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.3);
  width: 32px;
  height: 32px;
  border-radius: 50%;
  cursor: pointer;
  font-size: 1.1rem;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
}

.speed-controls button:hover {
  background: rgba(255, 255, 255, 0.2);
  transform: scale(1.1);
}

.speed-indicator {
  color: white;
  font-size: 1rem;
  min-width: 50px;
  text-align: center;
  font-weight: bold;
}

/* welcome screen */
.welcome-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0, 5, 20, 0.95);
  backdrop-filter: blur(10px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  animation: fadeIn 0.5s;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

.welcome-content {
  max-width: 800px;
  padding: 40px;
  text-align: center;
  color: white;
}

.welcome-content h1 {
  font-size: 3rem;
  margin: 0 0 10px 0;
  background: linear-gradient(45deg, #fdb813, #ffc849, #fff);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.tagline {
  font-size: 1.3rem;
  opacity: 0.9;
  margin-bottom: 40px;
}

.welcome-features {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 30px;
  margin: 40px 0;
}

.feature {
  padding: 20px;
  background: rgba(255, 255, 255, 0.05);
  border-radius: 10px;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.feature .icon {
  font-size: 2.5rem;
  display: block;
  margin-bottom: 10px;
}

.feature h3 {
  margin: 10px 0;
  font-size: 1.1rem;
}

.feature p {
  opacity: 0.8;
  font-size: 0.9rem;
  margin: 0;
}

.start-btn {
  background: #fdb813;
  color: #000;
  border: none;
  padding: 15px 40px;
  border-radius: 30px;
  font-size: 1.1rem;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.3s;
  margin-top: 20px;
}

.start-btn:hover {
  background: #ffc849;
  transform: translateY(-2px);
  box-shadow: 0 10px 20px rgba(253, 184, 19, 0.3);
}

.credits {
  margin-top: 40px;
  opacity: 0.6;
  font-size: 0.9rem;
}

.credits a {
  color: #fdb813;
  text-decoration: none;
}

.credits a:hover {
  text-decoration: underline;
}

/* sidebar toggle */
.sidebar-toggle {
  position: fixed;
  top: 20px;
  left: 20px;
  background: rgba(0, 5, 20, 0.9);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.2);
  padding: 8px 12px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1rem;
  z-index: 101; 
  transition: all 0.5s cubic-bezier(0.4, 0, 0.2, 1);
  backdrop-filter: blur(10px);
}

.sidebar-toggle:hover {
  background: rgba(253, 184, 19, 0.9);
  color: #000;
}

/* sidebar */
.sidebar {
  position: fixed;
  top: 0;
  left: 0;
  width: 320px;
  height: 100vh;
  max-height: 100vh;
  background: rgba(0, 5, 20, 0.95);
  backdrop-filter: blur(10px);
  border-right: 1px solid rgba(255, 255, 255, 0.2);
  padding: 60px 20px 20px 20px;
  padding-bottom: 120px;
  overflow-y: scroll; 
  overflow-x: hidden;
  z-index: 99;
  color: white;
  transform: translateX(-100%);
  transition: transform 0.5s cubic-bezier(0.4, 0, 0.2, 1);
  -webkit-overflow-scrolling: touch;
  box-sizing: border-box; 
}

.sidebar::-webkit-scrollbar {
  display: none;
}

.sidebar-open {
  transform: translateX(0);
}

@keyframes slideInSmooth {
  from {
    transform: translateX(-100%);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}

.sidebar h2 {
  margin: 0 0 20px 0;
  color: #fdb813;
  font-size: 1.5rem;
}

.sidebar-section {
  margin-bottom: 25px;
  padding-bottom: 20px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.1);
}

.sidebar-section h3 {
  margin: 0 0 10px 0;
  font-size: 1.1rem;
  color: #ffc849;
}

.sidebar-section p {
  margin: 0;
  opacity: 0.9;
  line-height: 1.6;
}

.sidebar-section ul {
  margin: 10px 0;
  padding-left: 20px;
  opacity: 0.9;
}

.sidebar-section li {
  margin: 8px 0;
}

.tech-badges {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-top: 10px;
}

.badge {
  background: rgba(253, 184, 19, 0.2);
  color: #fdb813;
  padding: 5px 12px;
  border-radius: 15px;
  font-size: 0.85rem;
  border: 1px solid rgba(253, 184, 19, 0.3);
}

.planet-list {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 8px;
  margin-top: 10px;
}

.planet-btn {
  background: rgba(255, 255, 255, 0.05);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.2);
  padding: 8px;
  border-radius: 5px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: all 0.2s;
}

.planet-btn:hover {
  background: rgba(253, 184, 19, 0.2);
  border-color: #fdb813;
}

.credits-sidebar {
  border-bottom: none;
}

.github-link {
  display: inline-block;
  margin-top: 10px;
  color: #fdb813;
  text-decoration: none;
  font-weight: bold;
}

.github-link:hover {
  text-decoration: underline;
}

/* responsive */
@media (max-width: 768px) {
  /* welcome screen */
  .welcome-overlay {
    overflow-y: auto;
    padding: 20px 0;
  }

  .welcome-content {
    padding: 40px 20px;
  }

  .welcome-content h1 {
    font-size: clamp(2rem, 8vw, 3rem);
  }

  .tagline {
    font-size: clamp(1rem, 4vw, 1.3rem);
    margin-bottom: 30px;
  }

  .welcome-features {
    grid-template-columns: 1fr;
    gap: 20px;
    margin: 30px 0;
  }

  .feature {
    padding: 15px;
  }

  .feature .icon {
    font-size: 2rem;
  }

  .start-btn {
    padding: 12px 30px;
    font-size: 1rem;
  }

  /* sidebar */
  .sidebar {
    width: 85vw;
    max-width: 320px;
  }

  .sidebar-toggle {
    position: fixed;
    top: 120px;
    left: 20px;
    background: rgba(0, 5, 20, 0.9);
    color: white;
    border: 1px solid rgba(255, 255, 255, 0.2);
    padding: 6px 10px;
    border-radius: 8px;
    cursor: pointer;
    font-size: 1rem; 
    z-index: 100;
    transition: all 0.2s;
    backdrop-filter: blur(10px);
  }

  /* time controls */
  .time-controls {
    top: 10px;
    left: 10px;
    right: 10px;
    transform: none;
    flex-direction: column;
    gap: 10px;
    padding: 10px;
  }

  .play-pause-btn {
    width: 100%;
    font-size: 1rem;
  }

  .speed-controls {
    width: 100%;
    justify-content: space-between;
  }

  .speed-controls button {
    padding: 5px 10px;
    font-size: 0.8rem;
  }

  .speed-indicator {
    text-align: center;
    font-size: 0.85rem;
    min-width: auto;
  }

  /* info panel */
  .info-panel {
    top: auto;
    bottom: 80px;
    right: 10px;
    left: 10px;
    max-width: none;
    padding: 15px;
  }

  .info-panel h2 {
    font-size: 1.3rem;
  }

  /* controls hint */
  .controls-hint {
    bottom: 10px;
    left: 10px;
    right: 10px;
    transform: none;
    font-size: 0.75rem;
    padding: 8px 15px;
  }

  .controls-hint p {
    margin: 0;
  }
}

@media (max-width: 480px) {
  /* welcome Screen */
  .welcome-content h1 {
    font-size: 1.8rem;
  }

  .tagline {
    font-size: 0.95rem;
    margin-bottom: 20px;
  }

  .welcome-features {
    gap: 15px;
    margin: 20px 0;
  }

  .feature {
    padding: 12px;
  }

  .feature .icon {
    font-size: 1.8rem;
  }

  .feature h3 {
    font-size: 1rem;
  }

  .feature p {
    font-size: 0.85rem;
  }

  .start-btn {
    padding: 10px 25px;
    font-size: 0.95rem;
  }

  .credits {
    font-size: 0.8rem;
  }

  /* sidebar */
  .sidebar {
    position: fixed;
    top: 0;
    left: 0;
    width: 320px;
    height: 100vh;
    background: rgba(0, 5, 20, 0.95);
    backdrop-filter: blur(10px);
    border-right: 1px solid rgba(255, 255, 255, 0.2);
    padding: 60px 20px 20px 20px; 
    overflow-y: auto;  
    overflow-x: hidden;
    z-index: 99;
    color: white;
    transform: translateX(-100%);
    transition: transform 0.5s cubic-bezier(0.4, 0, 0.2, 1);
    -webkit-overflow-scrolling: touch;
  }

  .sidebar h2 {
    font-size: 1.3rem;
  }

  .sidebar-section h3 {
    font-size: 1rem;
  }

  .sidebar-section p,
  .sidebar-section li {
    font-size: 0.9rem;
  }

  .planet-list {
    grid-template-columns: 1fr;
  }

  .planet-btn {
    padding: 10px;
    font-size: 0.95rem;
  }

  /* time Controls */
  .time-controls {
    top: 5px;
    left: 5px;
    right: 5px;
    padding: 8px;
    gap: 8px;
  }

  .speed-controls button {
    padding: 4px 8px;
    font-size: 0.75rem;
  }

  .speed-indicator {
    font-size: 0.8rem;
  }

  /* info Panel */
  .info-panel {
    bottom: 70px;
    right: 5px;
    left: 5px;
    padding: 12px;
  }

  .info-panel h2 {
    font-size: 1.2rem;
  }

  .info-panel p,
  .info-panel li {
    font-size: 0.85rem;
  }

  .info-panel button {
    padding: 6px 12px;
    font-size: 0.85rem;
  }

  /* controls Hint */
  .controls-hint {
    bottom: 5px;
    left: 5px;
    right: 5px;
    padding: 6px 12px;
    font-size: 0.7rem;
  }

  .sidebar-toggle {
    position: fixed;
    top: 80px; 
    left: 20px;
    background: rgba(0, 5, 20, 0.9);
    color: white;
    border: 1px solid rgba(255, 255, 255, 0.2);
    padding: 10px 15px;
    border-radius: 8px;
    cursor: pointer;
    font-size: 1.2rem;
    z-index: 100;
    transition: all 0.2s;
    backdrop-filter: blur(10px);
  }
}

/* landscape mode on small devices */
@media (max-height: 500px) and (orientation: landscape) {
  .time-controls {
    top: 5px;
    padding: 5px;
    flex-direction: row;
    flex-wrap: wrap;
  }

  .controls-hint {
    display: none;
  }

  .info-panel {
    bottom: 5px;
    max-height: 40vh;
    overflow-y: auto;
  }
}
</style>