<template></template>

<script setup>
import { onMounted, onUnmounted, reactive } from 'vue';
import * as THREE from 'three';
import { Sky } from 'three/addons/objects/Sky.js';
import { GLTFLoader } from 'three/addons/loaders/GLTFLoader.js';

let camera, scene, renderer;
let sky, sun;
let model;
const cameraOffset = new THREE.Vector3(0, 5, 10); // 向上5个单位，向后10个单位

// 创建控制器状态对象
const controls = reactive({
  forward: false,
  backward: false,
  left: false,
  right: false,
  mouseX: 0,
  mouseY: 0,
  jump: false,
  onGround: false,
  velocityX: 0,
  velocityY: 0, // 垂直方向的速度
  jumpSpeed: 3, // 初始跳跃速度
  gravity: 0.1 // 重力加速度
});

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

  camera.position.set(0, 10, -30); // 举例，根据模型的实际位置调整
  camera.lookAt(new THREE.Vector3(0, 10, 0)); // 假设模型位于原点，可以调整这个位置

  renderer = new THREE.WebGLRenderer({ antialias: true });
  renderer.setPixelRatio(window.devicePixelRatio);
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.toneMapping = THREE.ACESFilmicToneMapping;
  document.body.appendChild(renderer.domElement);

  renderer.domElement.addEventListener('click', () => {
    renderer.domElement.requestPointerLock();
  });
}

// 初始化照明
function initLighting() {
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
    model.rotation.y = Math.PI; // 将模型旋转180度
    scene.add(model);
    //骨架
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

// 处理键盘按下事件
function onKeyDown(event) {
  switch (event.code) {
    case 'KeyW':
      controls.forward = true;
      break;
    case 'KeyS':
      controls.backward = true;
      break;
    case 'KeyA':
      controls.left = true;
      break;
    case 'KeyD':
      controls.right = true;
      break;
    case 'Space':
      if (controls.onGround && !controls.jump) {
        controls.jump = true;
        controls.onGround = false;
        controls.velocityY = controls.jumpSpeed; // 使用定义的初始跳跃速度
      }
      break;
  }
}

// 处理键盘释放事件
function onKeyUp(event) {
  switch (event.code) {
    case 'KeyW':
      controls.forward = false;
      break;
    case 'KeyS':
      controls.backward = false;
      break;
    case 'KeyA':
      controls.left = false;
      break;
    case 'KeyD':
      controls.right = false;
      break;
    case 'Space':
      controls.jump = false;
      break;
  }
}

// 处理鼠标移动事件
function onMouseMove(event) {
  if (document.pointerLockElement === renderer.domElement && model) {
    let deltaX = event.movementX || event.mozMovementX || event.webkitMovementX || 0;

    // 更新模型的水平旋转
    model.rotation.y -= deltaX * 0.005;
  }
}

// 更新模型位置和相机视角
function updateMovement() {
  if (model) {
    const speed = 0.1; // 可以调整速度
    const direction = new THREE.Vector3();

    if (controls.forward) {
      direction.setFromMatrixColumn(model.matrix, 0);
      direction.crossVectors(model.up, direction);
      model.position.addScaledVector(direction, speed);
    }

    if (controls.backward) {
      direction.setFromMatrixColumn(model.matrix, 0);
      direction.crossVectors(model.up, direction);
      model.position.addScaledVector(direction, -speed);
    }

    if (controls.left) {
      direction.setFromMatrixColumn(model.matrix, 0);
      model.position.addScaledVector(direction, -speed);
    }

    if (controls.right) {
      direction.setFromMatrixColumn(model.matrix, 0);
      model.position.addScaledVector(direction, speed);
    }

    if (controls.jump) {
      // 应用垂直速度
      model.position.y += controls.velocityY;
      controls.velocityY -= 0.2; // 应用重力，可以根据需要调整
    }

    // 防止模型穿过地面
    if (model.position.y < 0) {
      model.position.y = 0;
      controls.jump = false;
      controls.velocityY = 0;
    }
    // 应用重力
    if (!controls.onGround) {
      controls.velocityY -= controls.gravity; // 应用重力加速度
      model.position.y += controls.velocityY;

      // 检测是否着陆
      if (model.position.y <= 0) {
        model.position.y = 0;
        controls.onGround = true;
        controls.jump = false;
        controls.velocityY = 0;
      }
    }
  }
}

function updateCameraPosition() {
  if (model && camera) {
    const modelPosition = new THREE.Vector3();
    model.getWorldPosition(modelPosition);

    const offset = cameraOffset.clone();
    offset.applyQuaternion(model.quaternion);
    const cameraPosition = modelPosition.clone().add(offset);
    camera.position.copy(cameraPosition);

    camera.lookAt(modelPosition);
  }
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
  updateMovement();
  renderer.render(scene, camera);
}

function animate() {
  requestAnimationFrame(animate);

  updateMovement();
  updateCameraPosition(); // 更新相机位置

  render();
}
function lockChangeAlert() {
  if (document.pointerLockElement === renderer.domElement) {
    console.log('锁定鼠标');
    document.addEventListener('mousemove', onMouseMove, false);
  } else {
    console.log('解锁鼠标');
    document.removeEventListener('mousemove', onMouseMove, false);
  }
}

document.addEventListener('pointerlockchange', lockChangeAlert, false);
document.addEventListener('mozpointerlockchange', lockChangeAlert, false);
document.addEventListener('webkitpointerlockchange', lockChangeAlert, false);


// 组件挂载时初始化
onMounted(() => {
  initScene();
  initLighting();
  initGround();
  initModel();
  initSky();
  animate();
  window.addEventListener('resize', onWindowResize);
  document.addEventListener('keydown', onKeyDown);
  document.addEventListener('keyup', onKeyUp);
  document.addEventListener('mousemove', onMouseMove);
});

// 组件卸载时清理资源
onUnmounted(() => {
  renderer.dispose()
  window.removeEventListener('resize', onWindowResize);
  document.removeEventListener('keydown', onKeyDown);
  document.removeEventListener('keyup', onKeyUp);
  document.removeEventListener('mousemove', onMouseMove);
});

</script>
