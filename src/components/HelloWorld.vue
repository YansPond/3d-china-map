<script setup>
import * as THREE from 'three'
import * as d3 from "d3";
import chinaJson from './Map/json/china.json'
import { OrbitControls } from "three-orbitcontrols-ts";

let uniforms = {
    time: {
        value: 0.0
    }
}
// 添加场景scene
const scene = new THREE.Scene();

// 渲染器renderer
const renderer = new THREE.WebGLRenderer({antialias: true});

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

const controls = new OrbitControls(camera, renderer.domElement);
// controls.autoRotate = true;
controls.addEventListener("change", () => renderer.render( scene, camera )); //监听鼠标、键盘事件

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
    const provinceline = new THREE.Object3D();

    // 每个的 坐标 数组
    const { coordinates } = elem.geometry;
    const color = COLOR_ARR[index % COLOR_ARR.length]
    // 循环坐标数组
    coordinates.forEach(multiPolygon => {
    multiPolygon.forEach((polygon) => {
        const shape = new THREE.Shape();
        const points = [];
        for (let i = 0; i < polygon.length; i++) {
            let [x, y] = projection(polygon[i]);
            points.push( new THREE.Vector3(x,-y,2));
            if (i === 0) {
                shape.moveTo(x, -y);
            }
            shape.lineTo(x, -y);
        }
        // 创建一条平滑的曲线
        const curve = new THREE.CatmullRomCurve3( points, true );
        const curvePoints = curve.getPoints( 5000 );
        const line1 = new THREE.BufferGeometry().setFromPoints( curvePoints );

        const length = curvePoints.length;
		var percents = new Float32Array( length );
		for (let i = 0; i < curvePoints.length; i += 1) {
			percents[i] = ( i / length );
		}
        console.log(line1);

		line1.setAttribute( 'percent', new THREE.BufferAttribute( percents, 1 ) );
		

      let material3 = new THREE.ShaderMaterial({
        uniforms: {
            time: uniforms.time,
            number: { type: 'f', value: 1.0 },
            speed: { type: 'f', value: 0.05 },
            length: { type: 'f', value:  0.5 },
            size: { type: 'f', value: 1.5},
            color: { value: new THREE.Color("#F8D764") }
        },
        vertexShader: `
            varying vec2 vUv;
            attribute float percent;
            uniform float time;
            uniform float number;
            uniform float speed;
            uniform float length;
            varying float opacity;
            uniform float size;
            void main()
            {
                vUv = uv;
                vec4 mvPosition = modelViewMatrix * vec4( position, 1.0 );
                float l = clamp(1.0-length,0.0,1.0);
                gl_PointSize = clamp(fract(percent*number + l - time*number*speed)-l ,0.0,1.) * size * (1./length);
                opacity = gl_PointSize/size;
                gl_Position = projectionMatrix * mvPosition;
            }
        `,
        fragmentShader: `
            #ifdef GL_ES
            precision mediump float;
            #endif
            varying float opacity;
            uniform vec3 color;
            void main(){
                if(opacity <=0.2){
                    gl_FragColor = vec4(0.0,0.0,0.0,1.0);
                } else {
                    gl_FragColor = vec4(color,1.0);
                }
            }
		`,
        transparent: true
      });

        const line = new THREE.Points( line1,  material3)

    // 挤压成立体形状
        const geometry = new THREE.ExtrudeGeometry(shape, {
            depth: 1,
            bevelEnabled: true,
            bevelSegments: 1,
            bevelThickness: 0.2
        });

        // 平面部分材质
        const material = new THREE.LineBasicMaterial( {
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
        provinceline.add(line);

        })
    })
    scene.add(province)
    scene.add(provinceline);

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
	for ( let i = 0; i < intersects.length; i ++ ) {
		intersects[ i ]?.object.material[0].color.set( 'yellow' );
	}
    renderer.render( scene, camera );
	
}
//window.addEventListener( 'pointermove', onPointerMove );

function animate(){
    uniforms.time.value += 0.07
    renderer.render( scene, camera );
    requestAnimationFrame(animate)
}

animate()


</script>

<template>
</template>

<style>

</style>
