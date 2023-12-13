<template></template>

<script setup>
import { onMounted, onUnmounted } from 'vue';
import * as THREE from 'three';
import { Sky } from 'three/addons/objects/Sky.js';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

let camera, scene, renderer;
let sky, sun;
let model;

// 初始化场景
function initScene() {
  scene = new THREE.Scene();//场景
  camera = new THREE.PerspectiveCamera(//相机
    75,//视野角度
    window.innerWidth / window.innerHeight,//长宽比
    0.1,//近截面
    1000//远截面
  );
  camera.position.y = 10;

  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  document.body.appendChild(renderer.domElement);

  // 添加环境光
  const ambientLight = new THREE.AmbientLight(0xffffff, 0.5); // 白色环境光，亮度为0.5
  scene.add(ambientLight);

  // 添加平行光
  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.5); // 白色平行光，亮度为0.5
  directionalLight.position.set(0, 1, 0); // 设置光源位置
  scene.add(directionalLight);
}

// 初始化地面
function initGround() {
  const groundTexture = new THREE.TextureLoader().load('/src/assets/grasslight-big.jpg', function (texture) {
    // 纹理加载完成后重新渲染(解决图片加载问题,不然默认是纯黑)
    render();
  });
  const groundGeometry = new THREE.PlaneGeometry(16000, 16000);
  const groundMaterial = new THREE.MeshLambertMaterial({ map: groundTexture });
  const ground = new THREE.Mesh(groundGeometry, groundMaterial);
  ground.rotation.x = -Math.PI / 2;
  ground.material.map.repeat.set(1000, 1000);
  ground.material.map.wrapS = THREE.RepeatWrapping;
  ground.material.map.wrapT = THREE.RepeatWrapping;
  ground.receiveShadow = true;//阴影
  scene.add(ground);
}

// 初始化模型
function initModel() {
  const loader = new GLTFLoader();
  loader.load('/src/assets/Soldier.glb', function (gltf) {
    model = gltf.scene;
    scene.add(model);

    // const skeleton = new THREE.SkeletonHelper(model);
    // skeleton.visible = true;
    // scene.add(skeleton);
  }, undefined, function (error) {
    console.error('模型加载出错: ', error);
  });
}

// 初始化天空
function initSky() {
  sky = new Sky();
  sky.scale.setScalar(450000);
  scene.add(sky);

  sun = new THREE.Vector3();

  // 设置天空参数
  const skyParams = {
    turbidity: 10,
    rayleigh: 0.36,
    mieCoefficient: 0,
    mieDirectionalG: 0.7,
    elevation: 17,
    azimuth: 180,
    exposure: renderer.toneMappingExposure
  };

  updateSky(skyParams);
}

function updateSky(params) {
  const uniforms = sky.material.uniforms;
  uniforms['turbidity'].value = params.turbidity;
  uniforms['rayleigh'].value = params.rayleigh;
  uniforms['mieCoefficient'].value = params.mieCoefficient;
  uniforms['mieDirectionalG'].value = params.mieDirectionalG;

  const phi = THREE.MathUtils.degToRad(90 - params.elevation);
  const theta = THREE.MathUtils.degToRad(params.azimuth);
  sun.setFromSphericalCoords(1, phi, theta);

  uniforms['sunPosition'].value.copy(sun);

  renderer.toneMappingExposure = params.exposure;
  render();
}

// 处理窗口大小变化
function onWindowResize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
  render();
}

// 渲染场景
function render() {
  renderer.render(scene, camera);
}

// 组件挂载时初始化
onMounted(() => {
  initScene();
  initGround();
  initModel();
  initSky();
  window.addEventListener('resize', onWindowResize);
});

// 组件卸载时清理资源
onUnmounted(() => {
  window.removeEventListener('resize', onWindowResize);
  // 这里可以添加更多的清理逻辑，比如删除场景中的对象等
});

</script>
