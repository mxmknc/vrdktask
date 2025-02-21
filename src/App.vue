<script setup lang="ts">
import {ref, onMounted, watch} from 'vue';
import * as THREE from 'three';
import {OrbitControls} from 'three/examples/jsm/controls/OrbitControls.js';
import {RGBELoader} from "three/examples/jsm/loaders/RGBELoader.js";
import {RoundedBoxGeometry} from 'three/examples/jsm/geometries/RoundedBoxGeometry.js';

const threeCanvas = ref<HTMLCanvasElement | null>(null);
const threeControls = ref<HTMLCanvasElement | null>(null);
const preload = ref<HTMLCanvasElement | null>(null);
const doorWidth = ref(1);
const doorHeight = ref(2);

let scene: THREE.Scene;
let camera: THREE.PerspectiveCamera;
let renderer: THREE.WebGLRenderer;
let doorMesh: THREE.Mesh | null = null;


const initThree = async () => {


  // scene

  scene = new THREE.Scene();
  scene.background = new THREE.Color('#aaaaaa');

  camera = new THREE.PerspectiveCamera(35, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.set(0, 4, 10);

  renderer = new THREE.WebGLRenderer({canvas: threeCanvas.value!, antialias: true});
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.shadowMap.enabled = true;


  // light
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5);
  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.5);
  directionalLight.position.set(5, 10, 7);

  directionalLight.shadow.mapSize.width = 2048;
  directionalLight.shadow.mapSize.height = 2048;
  directionalLight.shadow.radius = 8;
  directionalLight.shadow.bias = -0.000001;

  directionalLight.castShadow = true;
  scene.add(ambientLight, directionalLight);


  // env
  const addHdriEnvironment = (scene: THREE.Scene, renderer: THREE.WebGLRenderer, hdriPath: string) => {

    return new Promise(resolve => {
      const pmremGenerator = new THREE.PMREMGenerator(renderer);
      pmremGenerator.compileEquirectangularShader();

      new RGBELoader()
          .setPath(hdriPath)
          .load('one.hdr', (texture) => {
            scene.environment = pmremGenerator.fromEquirectangular(texture).texture;

            const rotateEuler = new THREE.Euler(Math.PI / 2, 0, 0);
            scene.environmentRotation = rotateEuler;
            scene.environmentIntensity = 0.7;
            //scene.background = envMap;

            texture.dispose();
            pmremGenerator.dispose();

            resolve(true);
          });
    })

  };
  await addHdriEnvironment(scene, renderer, './hdri/');


  // ground
  const ground = new THREE.Mesh(
      new THREE.PlaneGeometry(14, 10),
      new THREE.MeshStandardMaterial({color: '#b0b4b7'})
  );
  ground.rotation.x = -Math.PI / 2;
  ground.receiveShadow = true;
  scene.add(ground);


  // sphere
  const sphere = new THREE.Mesh(
      new THREE.SphereGeometry(0.7),
      new THREE.MeshPhysicalMaterial({color: '#ff6347', roughness: 0.2, metalness: 0.2})
  );
  sphere.position.set(0, 1, 0);
  sphere.castShadow = true;
  scene.add(sphere);


  // cube
  const cubeGeometry = new RoundedBoxGeometry(
      1,
      1,
      1,
      4,
      0.2
  );
  const cubeMaterial = new THREE.MeshPhysicalMaterial({color: '#67818d', roughness: 0.4, metalness: 0.1});
  const cube = new THREE.Mesh(cubeGeometry, cubeMaterial);

  cube.position.set(-3, 0.5, 0);
  cube.castShadow = true;
  scene.add(cube);


  // wood texture
  const loadTexture = (loader: THREE.TextureLoader, path: string): Promise<THREE.Texture> => {
    return new Promise((resolve) => {
      loader.load(
          path,
          texture => resolve(texture)
      );
    });
  };

  const loadWoodTextures = async (path = '') => {
    const textureLoader = new THREE.TextureLoader().setPath(path);

    const [colorMap, normalMap, roughnessMap] = await Promise.all([
      loadTexture(textureLoader, 'wood_0050_color_1k.jpg'),
      loadTexture(textureLoader, 'wood_0050_normal_opengl_1k.jpg'),
      loadTexture(textureLoader, 'wood_0050_roughness_1k.jpg')
    ]);

    [colorMap, normalMap, roughnessMap].forEach(tex => tex.anisotropy = 16);

    return {colorMap, normalMap, roughnessMap};
  };

  const {colorMap, normalMap, roughnessMap} = await loadWoodTextures('./textures/wood/');


  // door material

  const doorMaterial = new THREE.MeshPhysicalMaterial({
    map: colorMap,
    normalMap: normalMap,
    roughnessMap: roughnessMap,
    roughness: 1,
    metalness: 0,
    reflectivity: 0,
    clearcoat: 0,
    clearcoatRoughness: 1,
  });
  const handleMaterial = new THREE.MeshStandardMaterial({color: '#424035', roughness: 0.4, metalness: 1});


  // door
  const createDoor = () => {
    if (doorMesh) {
      scene.remove(doorMesh);
      doorMesh.geometry.dispose();
    }

    const scaleFactor = 1;

    [colorMap, normalMap, roughnessMap].forEach((map) => {
      map.wrapS = THREE.RepeatWrapping;
      map.wrapT = THREE.RepeatWrapping;

      map.repeat.set(doorWidth.value * scaleFactor, doorHeight.value * scaleFactor);
      map.needsUpdate = true;
    });


    const doorGeometry = new THREE.BoxGeometry(doorWidth.value, doorHeight.value, 0.1);
    doorMesh = new THREE.Mesh(doorGeometry, doorMaterial);

    doorMesh.position.set(3, doorHeight.value / 2, 0);


    scene.add(doorMesh);


    // handle
    const handleOffset = 0.1;
    const handleWidth = 0.15;
    const handleHeight = 0.03;
    const handleDepth = 0.02;
    const baseWidth = 0.05;
    const baseDepth = 0.05;


    const baseGeometry = new THREE.BoxGeometry(baseWidth, baseWidth, baseDepth);
    const baseMesh = new THREE.Mesh(baseGeometry, handleMaterial);
    baseMesh.position.set(doorWidth.value / 2 - handleOffset, -0.1, 0.055);


    const handleGeometry = new THREE.BoxGeometry(handleWidth, handleHeight, handleDepth);
    const handleMesh = new THREE.Mesh(handleGeometry, handleMaterial);
    handleMesh.position.set((handleWidth / 2) + (baseWidth / 2) - 0.16, 0, 0.025);


    doorMesh.castShadow = true;

    baseMesh.add(handleMesh);
    doorMesh.add(baseMesh);

  };

  createDoor()


  // mount

  if (threeControls.value && threeControls.value.style) threeControls.value.style.display = 'flex';
  if (preload.value && preload.value.style) preload.value.style.display = 'none';

  watch([doorWidth, doorHeight], createDoor);

  const controls = new OrbitControls(camera, renderer.domElement);


  //animation

  let sphereVelocity = 2;
  const gravity = -7.81;
  const bounceDamping = 0.5;
  const floorHeight = 0;

  sphere.position.y = 3;


  let previousTime = performance.now();

  const sphereAnimate = () => {
    const currentTime = performance.now();
    const deltaTime = (currentTime - previousTime) / 1000;
    previousTime = currentTime;

    sphereVelocity += gravity * deltaTime;

    sphere.position.y += sphereVelocity * deltaTime;

    const deformationSpeed = 10;
    sphere.scale.x += (1 - sphere.scale.x) * deformationSpeed * deltaTime;
    sphere.scale.y += (1 - sphere.scale.y) * deformationSpeed * deltaTime;
    sphere.scale.z += (1 - sphere.scale.z) * deformationSpeed * deltaTime;

    if (sphere.position.y <= floorHeight + 0.7) {
      sphere.position.y = floorHeight + 0.7;
      sphereVelocity *= -bounceDamping;

      const impactDeformation = Math.min(Math.abs(sphereVelocity) * 0.1, 0.1);
      sphere.scale.set(1 + impactDeformation, 1 - impactDeformation, 1 + impactDeformation);

      if (Math.abs(sphereVelocity) < 0.3) {
        sphereVelocity = 0;
      }
    }
  };

  const animate = () => {
    requestAnimationFrame(animate);
    sphereAnimate();
    controls.update();
    renderer.render(scene, camera);
  };

  animate();

  window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  });
};

onMounted(initThree);
</script>

<template>
  <div class="w-full h-screen">
    <div ref="preload" class="preload">wait</div>
    <canvas ref="threeCanvas" class="w-full h-full"></canvas>
    <div class="controls" ref="threeControls">
      <div>
        <label>
          <text>Ширина</text>
          <input type="range" v-model="doorWidth" min="0.5" max="3" step="0.1"/>
        </label>
      </div>
      <div>
        <label style="margin-left:20px;">
          <text>Высота</text>
          <input type="range" v-model="doorHeight" min="1" max="5" step="0.1"/>
        </label>
      </div>
    </div>
  </div>
</template>

<style scoped>

.preload {
  width: 100%;
  height: 100%;
  position: absolute;
  top: 0;
  left: 0;
  display: flex;
  justify-content: center;
  align-items: center;
}

.controls {
  position: absolute;
  top: 20px;
  right: 20px;
  display: none;
  justify-content: center;
  align-items: center;
  background: rgba(255, 255, 255, 0.5);
  padding: 10px;
  border-radius: 8px;
}

input {
  margin-left: 10px;
}
</style>
