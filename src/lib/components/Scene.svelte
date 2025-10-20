<script lang="ts">
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import { celestialBodies } from '$lib/data/celestialBodies.js';

  let container: HTMLDivElement | undefined;
  let selectedPlanet: any = null;

  export function getSelectedPlanet() {
    return selectedPlanet;
  }

  onMount(() => {
    // scene setup
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x000511);

    // camera setup
    const camera = new THREE.PerspectiveCamera(
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
    // lighting setup
    const ambientLight = new THREE.AmbientLight(0x333333);
    scene.add(ambientLight);

    const pointLight = new THREE.PointLight(0xffffff, 2, 500);
    pointLight.position.set(0, 0, 0);
    scene.add(pointLight);

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
    Object.entries(celestialBodies).forEach(([key, body]) => {
      const geometry = new THREE.SphereGeometry(body.radius, 32, 32);
      const material = new THREE.MeshStandardMaterial({
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
        const ringMaterial = new THREE.MeshBasicMaterial({
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
        const orbitMaterial = new THREE.LineBasicMaterial({ color: 0x444444, transparent: true, opacity: 0.3 });
        const orbit = new THREE.Line(orbitGeometry, orbitMaterial);
        scene.add(orbit);
      }
    });

    // raycaster for click detection
    const raycaster = new THREE.Raycaster();
    const mouse = new THREE.Vector2();

    function onMouseClick(event: MouseEvent) {
      mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
      mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

      raycaster.setFromCamera(mouse, camera);
      const intersects = raycaster.intersectObjects(planets);

      if (intersects.length > 0) {
        const clickedPlanet = intersects[0].object;
        selectedPlanet = clickedPlanet.userData;
        
        // zoom to planet view
        const targetPos = clickedPlanet.position.clone();
        const distance = clickedPlanet.userData.radius * 4;
        camera.position.set(
          targetPos.x + distance,
          targetPos.y + distance,
          targetPos.z + distance
        );
        camera.lookAt(targetPos);
      } else {
        selectedPlanet = null;
        // return to overview view
        camera.position.set(0, 150, 200);
        camera.lookAt(0, 0, 0);
      }
    }

    window.addEventListener('click', onMouseClick);

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
      planets.forEach(planet => {
        const data = planet.userData;

        // rotation on axis
        planet.rotation.y += data.rotationSpeed;

        // orbit around sun
        if (data.distance > 0) {
          data.angle += data.orbitSpeed;
          planet.position.x = Math.cos(data.angle) * data.distance;
          planet.position.z = Math.sin(data.angle) * data.distance;
        }
      });

      renderer.render(scene, camera);
    }

    animate();

    // cleanup on component unmount
    return () => {
      window.removeEventListener('click', onMouseClick);
      window.removeEventListener('resize', onWindowResize);
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
</style>