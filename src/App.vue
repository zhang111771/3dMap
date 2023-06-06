<template>
  <div ref="container"></div>
</template>

<script setup>
import * as THREE from 'three'
import {ref,onMounted} from 'vue'
import {OrbitControls} from 'three/examples/jsm/controls/OrbitControls'
import * as d3 from 'd3'

const scene=new THREE.Scene()
const camera=new THREE.PerspectiveCamera(75,window.innerWidth/window.innerHeight,0.1,1000)
camera.position.set(0,0,100)
scene.add(camera)
const renderer=new THREE.WebGLRenderer({antialias:true})
renderer.setSize(window.innerWidth,window.innerHeight)
const container=ref(null)

const helper=new THREE.AxesHelper(5)
scene.add(helper)
function animate(){
  controls.update()
  renderer.render(scene,camera)
  requestAnimationFrame(animate)
}
const controls=new OrbitControls(camera,renderer.domElement)
controls.enableDamping=true
onMounted(()=>{
  container.value.appendChild(renderer.domElement)
  animate()
  const loader=new THREE.FileLoader()
  loader.load('./assets/100000_full.json',(data)=>{
    const jsonData=JSON.parse(data)
    operationData(jsonData)
  })
})
const map=new THREE.Object3D()
function operationData(jsonData){
  //获取所有的特征
  const features=jsonData.features
  features.forEach((feature)=>{
    //创建省份的物体
    const province=new THREE.Object3D()
    province.properties=feature.properties.name
    //获取经纬度坐标
    const coordinates=feature.geometry.coordinates
    if(feature.geometry.type=='Polygon'){
      coordinates.forEach((coordinate)=>{
        const mesh=createMesh(coordinate)
        mesh.properties=feature.properties.name
        province.add(mesh)
        const line=createLine(coordinate)
        province.add(line)
      })
    }
    if(feature.geometry.type==='MultiPolygon'){
      coordinates.forEach((item)=>{
        item.forEach((coordinate)=>{
          const mesh=createMesh(coordinate)
          mesh.properties=feature.properties.name
          province.add(mesh)
          const line=createLine(coordinate)
          province.add(line)
        })
      })
    }

    map.add(province)
  })

  scene.add(map)
}
//用d3转换为以北京为中心的世界平面坐标
const projection=d3.geoMercator().center([116.5,38.5]).translate([0,0])
function createMesh(polygon){

  const shape=new THREE.Shape()
  polygon.forEach((row,i)=>{
    const [longitude,latitude]=projection(row)
    if(i===0){
      shape.moveTo(longitude,-latitude)
    }
    shape.lineTo(longitude,-latitude)
  })

  //挤出形状
  const geometry=new THREE.ExtrudeBufferGeometry(shape,{depth:5})
//创建随机的颜色
const color=new THREE.Color(Math.random()*0xffffff)
  const material=new THREE.MeshBasicMaterial({
    color:color,
    transparent:true,
    opacity:0.5
  })
  return new THREE.Mesh(geometry,material)
}

//根据经纬度划线
function createLine(polygon){
  const lineGeometry=new THREE.BufferGeometry()
  const pointsArray=[]
  polygon.forEach((row,i)=>{
    const [longitude,latitude]=projection(row)
    pointsArray.push(new THREE.Vector3(longitude,-latitude,10))
  })
  //通过点生成线几何体
  lineGeometry.setFromPoints(pointsArray)
  //创建随机的颜色
const color=new THREE.Color(Math.random()*0xffffff)
const lineMaterial=new THREE.MeshBasicMaterial({color:color})
return new THREE.Line(lineGeometry,lineMaterial)

}
let lastPicker=null
const mouse=new THREE.Vector2()
window.addEventListener('click',(event)=>{
  //获取鼠标的位置
  mouse.x=(event.clientX/window.innerWidth)*2-1
  mouse.y=1-(event.clientY/window.innerHeight)*2
  //获取鼠标点击的位置
  const raycaster=new THREE.Raycaster()
  raycaster.setFromCamera(mouse,camera)
  //获取点击的点
  const intersects=raycaster.intersectObjects(map.children)
  if(intersects.length>0){
    if(lastPicker){
      lastPicker.material.color.copy(lastPicker.material.oldColor)
    }
    lastPicker=intersects[0].object
    lastPicker.material.oldColor=lastPicker.material.color.clone()
    lastPicker.material.color.set(0xffffff)//设置白色


  }else{
    if(lastPicker){
      lastPicker.material.color.copy(lastPicker.material.oldColor)//恢复原来的颜色
    }
  }

})
</script>

<style>
*{
  padding:0;
  margin:0;
}
</style>
