<script setup>
import * as THREE from 'three'
import * as d3 from "d3";
import chinaJson from './Map/json/china.json'

// 添加场景scene
const scene = new THREE.Scene();

// 渲染器renderer
const renderer = new THREE.WebGLRenderer();

// 设置renderer背景颜色
renderer.setClearColor(0xffffff, 0.9);
// 设置renderer大小
renderer.setSize( window.innerWidth, window.innerHeight );
document.body.appendChild( renderer.domElement );

// 添加相机-PerspectiveCamera 透视相机 模拟人眼
const camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 5000);
// 相机位置
camera.position.set(0,-70, 70);
// 相机朝向
camera.lookAt(0, 0, 0);
// 坐标辅助线
const axesHelper = new THREE.AxesHelper( 500 );
scene.add( axesHelper );
const size = 100;
const divisions = 10;
// 网格辅助线
const gridHelper = new THREE.GridHelper( size, divisions );
scene.add( gridHelper );
// 摄像机辅助线
const helper = new THREE.CameraHelper( camera );
scene.add( helper );
// 颜色变量
const COLOR_ARR = ['#0465BD', '#357bcb', '#3a7abd']
// 墨卡托投影转换才能转换成平面坐标
const projection = d3.geoMercator().center([104.0, 37.5]).scale(80).translate([0, 0]);
// 画地图
chinaJson.features.forEach((elem, index) => {
    // 定一个省份3D对象
    const province = new THREE.Object3D();
    // 每个的 坐标 数组
    const { coordinates } = elem.geometry;
    const color = COLOR_ARR[index % COLOR_ARR.length]
    // 循环坐标数组
    coordinates.forEach(multiPolygon => {
    multiPolygon.forEach((polygon) => {
        const shape = new THREE.Shape();
        for (let i = 0; i < polygon.length; i++) {
            let [x, y] = projection(polygon[i]);
            if (i === 0) {
                shape.moveTo(x, -y);
            }
            shape.lineTo(x, -y);
        }
    // 挤压成立体形状
        const geometry = new THREE.ExtrudeGeometry(shape, {
            depth: 1,
            bevelEnabled: true,
            bevelSegments: 1,
            bevelThickness: 0.2
        });
        // 平面部分材质
        const material = new THREE.MeshBasicMaterial( {
            metalness: 1,
            color: color,
        } );
        // // 拉高部分材质
        const material1 = new THREE.MeshStandardMaterial( {
            metalness: 1,
            roughness: 1,
            color: color,
        } );
      // 应用材质
        const mesh = new THREE.Mesh(geometry, [
            material,
            material1
        ]);
        // 设置高度将省区分开来
        if (index % 2 === 0) {
            mesh.scale.set(1, 1, 1.2);
        }
        // 给mesh开启阴影
        mesh.castShadow = true
        mesh.receiveShadow = true
        mesh._color = color
        province.add(mesh);
        })
    })
    scene.add(province)
})

let ambientLight = new THREE.AmbientLight(0xffffff, 0.2); // 环境光
scene.add(ambientLight)
let hemiLight = new THREE.HemisphereLight('#80edff','#75baff', 0.3)
// 这个也是默认位置
hemiLight.position.set(20, -50, 0)
scene.add(hemiLight)
const light = new THREE.DirectionalLight( 0xffffff, 0.5 ); 
light.position.set( 20, -50, 20 );
light.castShadow = true;
light.shadow.mapSize.width = 1024;
light.shadow.mapSize.height = 1024;

const pointLight = new THREE.PointLight(0xffffff, 0.5)
pointLight.position.set( 20, -50, 50 );
pointLight.castShadow = true;
pointLight.shadow.mapSize.width = 1024;
pointLight.shadow.mapSize.height = 1024;
const pointLight2 = new THREE.PointLight(0xffffff, 0.5)
pointLight2.position.set( 50, -50, 20 );
pointLight2.castShadow = true;
pointLight2.shadow.mapSize.width = 1024;
pointLight2.shadow.mapSize.height = 1024;

const pointLight3 = new THREE.PointLight(0xffffff, 0.5)
pointLight3.position.set( -50, -50, 20 );
pointLight3.castShadow = true;
pointLight3.shadow.mapSize.width = 1024;
pointLight3.shadow.mapSize.height = 1024;

scene.add(pointLight);
scene.add(pointLight2);
scene.add(pointLight3);
scene.add(light);

// 添加鼠标滑动事件
const raycaster = new THREE.Raycaster();
const pointer = new THREE.Vector2();

function onPointerMove( event ) {

	// 将鼠标位置归一化为设备坐标。x 和 y 方向的取值范围是 (-1 to +1)

	pointer.x = ( event.clientX / window.innerWidth ) * 2 - 1;
	pointer.y = - ( event.clientY / window.innerHeight ) * 2 + 1;
  render()
}

function render() {
	// 通过摄像机和鼠标位置更新射线
	raycaster.setFromCamera( pointer, camera );
	// 计算物体和射线的焦点
	const intersects = raycaster.intersectObjects( scene.children );
  console.log(intersects)
	for ( let i = 0; i < intersects.length; i ++ ) {
		intersects[ i ]?.object.material[0].color.set( 'yellow' );
	}
	renderer.render( scene, camera );
}
window.addEventListener( 'pointermove', onPointerMove );
renderer.render(scene,camera);
</script>

<template>
</template>

<style>

</style>
