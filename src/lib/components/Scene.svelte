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


  function onMouseDown() {
    isDragging = false;
    mouseDownTime = Date.now();
  }

  function onMouseMove() {
    isDragging = true;
  }

  export function getSelectedPlanet() {
    return selectedPlanet;
  }

  onMount(() => {
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

    // store planet meshes for interaction and animation
    const planets: THREE.Mesh[] = [];

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
      renderer.render(scene, camera);
    }

    animate();

    // cleanup on component unmount
    return () => {
      window.removeEventListener('mousedown', onMouseDown);
      window.removeEventListener('mousemove', onMouseMove);
      window.removeEventListener('click', onMouseClick);
      window.removeEventListener('resize', onWindowResize);
      controls.dispose();
      renderer.dispose();
    };
  });
</script>

<div bind:this={container} class="scene-container"></div>

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
    <button on:click={() => timeSpeed = 0.5} class:active={timeSpeed === 0.5}>0.5x</button>
    <button on:click={() => timeSpeed = 1} class:active={timeSpeed === 1}>1x</button>
    <button on:click={() => timeSpeed = 2} class:active={timeSpeed === 2}>2x</button>
    <button on:click={() => timeSpeed = 5} class:active={timeSpeed === 5}>5x</button>
    <button on:click={() => timeSpeed = 10} class:active={timeSpeed === 10}>10x</button>
  </div>
  
  <span class="speed-indicator">Speed: {timeSpeed}x {isPaused ? '(Paused)' : ''}</span>
 </div>

<style>
  .scene-container {
    width: 100vw;
    height: 100vh;
    overflow: hidden;
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

  .time-controls {
  position: fixed;
  top: 20px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 15px;
  align-items: center;
  background: rgba(0, 5, 20, 0.9);
  padding: 15px 20px;
  border-radius: 10px;
  border: 1px solid rgba(255, 255, 255, 0.2);
  backdrop-filter: blur(10px);
  z-index: 10;
}

.play-pause-btn {
  background: #fdb813;
  border: none;
  padding: 8px 16px;
  border-radius: 5px;
  cursor: pointer;
  font-size: 1.2rem;
  transition: background 0.2s;
}

.play-pause-btn:hover {
  background: #ffc849;
}

.speed-controls {
  display: flex;
  gap: 5px;
}

.speed-controls button {
  background: rgba(255, 255, 255, 0.1);
  color: white;
  border: 1px solid rgba(255, 255, 255, 0.3);
  padding: 6px 12px;
  border-radius: 5px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: all 0.2s;
}

.speed-controls button:hover {
  background: rgba(255, 255, 255, 0.2);
}

.speed-controls button.active {
  background: #fdb813;
  border-color: #fdb813;
  font-weight: bold;
}

.speed-indicator {
  color: white;
  font-size: 0.9rem;
  min-width: 120px;
}
</style>