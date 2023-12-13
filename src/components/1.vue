<template>
    <!-- <div id="blocker">
      <div id="instructions">
        <p style="font-size:36px">暂停</p>
      </div>
    </div> -->
  </template>
  
  <script setup>
  import { onMounted } from 'vue';
  import * as THREE from 'three';
  import { PointerLockControls } from 'three/addons/controls/PointerLockControls.js';
  import { GUI } from 'three/addons/libs/lil-gui.module.min.js';
  import { Sky } from 'three/addons/objects/Sky.js';
  import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';
  
  let camera, scene, renderer, controls;
  
  let sky, sun;
  
  
  function init() {
    //场景
    scene = new THREE.Scene();
    //相机
    camera = new THREE.PerspectiveCamera(
      75,//视野角度
      window.innerWidth / window.innerHeight,//长宽比
      0.1,//近截面
      1000//远截面
    );
  
    camera.position.y = 10;
  
    // pause();//暂停
  
    // 地面
    const gt = new THREE.TextureLoader().load('/src/assets/grasslight-big.jpg');//加载文件
    const gg = new THREE.PlaneGeometry(16000, 16000);//地面总大
    const gm = new THREE.MeshBasicMaterial({ color: 0xffffff, map: gt });//材质
  
    const ground = new THREE.Mesh(gg, gm);
    ground.rotation.x = - Math.PI / 2;
    ground.material.map.repeat.set(1000, 1000);
    ground.material.map.wrapS = THREE.RepeatWrapping;
    ground.material.map.wrapT = THREE.RepeatWrapping;
    ground.material.map.colorSpace = THREE.SRGBColorSpace;
    ground.receiveShadow = true;//好像是阴影
    scene.add(ground);
  
    //渲染器
    renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.toneMapping = THREE.ACESFilmicToneMapping;
    document.body.appendChild(renderer.domElement);
  
    //加载模型
    const loader = new GLTFLoader();
    loader.load('/src/assets/Soldier.glb', function (gltf) {
      let model = gltf.scene
      scene.add(model);
  
      let skeleton = new THREE.SkeletonHelper(model);
      skeleton.visible = true;
      scene.add(skeleton);
  
    });
  
  
    initSky()
    window.addEventListener('resize', onWindowResize);
  }
  
  //渲染
  function animate() {
    requestAnimationFrame(animate);
    //旋转
    // cube.rotation.x += 0.01;
    // cube.rotation.y += 0.01;
  
    render();
  }
  
  function onWindowResize() {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
    render();
  }
  //天空
  function initSky() {
    // 添加天空
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
  
  
  function render() {
    renderer.render(scene, camera);
  }
  //暂停遮罩
  function pause() {
    controls = new PointerLockControls(camera, document.body);
  
    const blocker = document.getElementById('blocker');
    const instructions = document.getElementById('instructions');
  
    instructions.addEventListener('click', function () {
      controls.lock();
    });
  
    controls.addEventListener('lock', function () {
      instructions.style.display = 'none';
      blocker.style.display = 'none';
    });
  
    controls.addEventListener('unlock', function () {
      blocker.style.display = 'block';
      instructions.style.display = '';
    });
  
    scene.add(controls.getObject());
  }
  
  onMounted(() => {
    init();
    animate();
  })
  </script>