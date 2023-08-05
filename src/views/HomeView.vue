<template>
  <canvas ref="canvasRef"></canvas>
  <div class="loading-bar"></div>

  <!--The point that appear on the helmet model and display a text when hover-->
  <div class="point point-0">
    <div class="label">1</div>
    <div class="text">Lorem ipsum dolor sit amet consectetur adipisicing elit</div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted} from "vue"
import { OrbitControls} from "three/examples/jsm/controls/OrbitControls"
import {GLTFLoader} from "three/examples/jsm/loaders/GLTFLoader"
import * as THREE from "three"
import gsap from "gsap"
import * as dat from "dat.gui"


//GUI
const gui = new dat.GUI()

//CANVAS
let canvasRef = ref();

var controls

//Utilizziamo i valori del viewport per impostare la grandezza del canvas
const sizes = {
  width: window.innerWidth,
  height: window.innerHeight
}

//Aggiungiamo l'event listner per il resize del canvas
window.addEventListener("resize", ()=>{
  //update sizes for the canvas
  sizes.width = window.innerWidth
  sizes.height = window.innerHeight

  //update the camera aspect ratio
  camera.aspect = sizes.width / sizes.height
  camera.updateProjectionMatrix()

  //update renderer
  renderer.setSize(sizes.width, sizes.height)
  //HANDLE PIXEL RATIO
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
})


//HANDLE FULL SCREEN (Il codice aggiuntivo è per permettere il funzionamento corretto anche con browser Safari)
window.addEventListener("dblclick",()=>{
  const fullscreenElement = document.fullscreenElement ||  document.webkitFullscreenElement
  if(!fullscreenElement){
    if(canvasRef.value.requestFullscreen){
      canvasRef.value.requestFullscreen()
    }
    else if(canvasRef.value.webkitRequestFullscreen){
      canvasRef.value.webkitRequestFullscreen()
    }
    
  }
  else{
    if(document.exitFullscreen)
    {
      document.exitFullscreen()
    }
    else if(document.webkitExitFullscreen){
      document.webkitExitFullscreen()
    }
  }
})

let scene = new THREE.Scene()

const overlayGeo = new THREE.PlaneGeometry(2,2,1,1)
const overlayMat = new THREE.ShaderMaterial({
    transparent: true,
    uniforms:{
      uAlpha: { value: 1}
    },
    vertexShader:`
    void main()
    {
      gl_Position = vec4(position, 1.0);
    }
    `,
    fragmentShader:`
    uniform float uAlpha;
    
    void main()
    {
      gl_FragColor = vec4(0.0, 0.0, 0.0, uAlpha);
    }
    `
})
const overlay = new THREE.Mesh(overlayGeo, overlayMat)
scene.add(overlay)


let renderer

//Lights
const directionalLight = new THREE.DirectionalLight("#ffffff",3)
directionalLight.position.set(0.25, 3, -2.25)
directionalLight.castShadow = true
directionalLight.shadow.camera.far = 15
directionalLight.shadow.mapSize.set(1024, 1024)
gui.add(directionalLight, "intensity").min(0).max(10).step(0.001).name("lightIntensity")
gui.add(directionalLight.position, "x").min(-5).max(5).step(0.001).name("light X")
gui.add(directionalLight.position, "y").min(-5).max(5).step(0.001).name("light Y")
gui.add(directionalLight.position, "z").min(-5).max(5).step(0.001).name("light Z")
scene.add(directionalLight)

// const directionalLightCameraHelper = new THREE.CameraHelper(directionalLight.shadow.camera)
// scene.add(directionalLightCameraHelper)




// CAMERA
let camera = new THREE.PerspectiveCamera(
  75, 
  sizes.width / sizes.height, 
  0.1,
  100  
);
camera.position.set(0,0,20)
scene.add(camera);

const clock = new THREE.Clock()

//Animate Loop
const animate = () =>{
  const elapsedTime = clock.getElapsedTime()

  controls.update()
  
  renderer.render(scene, camera)
}

onMounted(() => {
  

  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value,
    antialias: true
  });
  
  renderer.setSize(sizes.width, sizes.height)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio,2)) // in questo modo il max valore di pixel ratio utilizzaro è 2 (altrimenti devi renderizzare troppi pixel, dispiego enorme di potenza gpu)
  renderer.outputColorSpace = THREE.SRGBColorSpace

  renderer.toneMapping = THREE.ReinhardToneMapping
  renderer.toneMappingExposure = 3

  gui.add(renderer, "toneMapping", {
    Not: THREE.NoToneMapping,
    Linear: THREE.LinearToneMapping,
    Reinhard: THREE.ReinhardToneMapping,
    Cineon: THREE.CineonToneMapping,
    ACESFilmic: THREE.ACESFilmicToneMapping
  }).onChange(()=>{
    renderer.toneMapping = Number(renderer.toneMapping)
    updateAllMaterials()
  })


  //add the toneMappingExposure too to dat.GUI
  gui.add(renderer, "toneMappingExposure").min(0).max(10).step(0.001)


  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFSoftShadowMap


  renderer.render(scene, camera)
  renderer.setAnimationLoop(animate)

  //CONTROLS
  controls = new OrbitControls(camera, canvasRef.value)
  controls.enableDamping = true; // permette uno effetto smooth una volta che rilasciamo l'elemento, rallenta fino a fermarsi

  ////////////////////////////////// MIX HTML & WEBGL /////////////////////////////////////////////////////
  //In this lesson we are going to mix the webgl and html
  //So let's go to the template parts to create some point , and when hovered display a text



//LOADER AND LOADING SECTION

//Loading Bar ELEMENT
const loadingBarElement = document.querySelector(".loading-bar")


const loadingManager = new THREE.LoadingManager(

//Loaded
  ()=>{
    window.setTimeout(()=>{
    gsap.to(overlayMat.uniforms.uAlpha, {duration:3, value:0})
    loadingBarElement.classList.add("ended")
    loadingBarElement.style.transform = ""
    }, 500)  
  },

  (itemUrl, itemsLoaded, itemsTotal)=>{

    const progressRatio = itemsLoaded / itemsTotal 
    
    if(loadingBarElement)
    {
      loadingBarElement.style.transform = "scaleX("+ progressRatio +")"
    }
   

    console.log("progress")
  }
)



const gltfLoader = new GLTFLoader(loadingManager)

const cubeTextureLoader = new THREE.CubeTextureLoader(loadingManager)


const environmentMap = cubeTextureLoader.load([
  "/textures/environmentMaps/0/px.jpg",
  "/textures/environmentMaps/0/nx.jpg",
  "/textures/environmentMaps/0/py.jpg",
  "/textures/environmentMaps/0/ny.jpg",
  "/textures/environmentMaps/0/pz.jpg",
  "/textures/environmentMaps/0/nz.jpg",
])


environmentMap.colorSpace = THREE.SRGBColorSpace

scene.background = environmentMap



gltfLoader.load("/models/DamagedHelmet/glTF/DamagedHelmet.gltf", (gltf)=>{
  
  gltf.scene.scale.set(10,10,10)
  gltf.scene.position.set(0,-4,0)
  gltf.scene.rotation.y = -Math.PI * 0.25 
  scene.add(gltf.scene)
  
  gui.add(gltf.scene.rotation, "y").min(-Math.PI).max(Math.PI).step(0.001).name("rotation")

  updateAllMaterials()
})



const updateAllMaterials = () =>{
  scene.traverse((child)=>{ 
  
    if(child instanceof THREE.Mesh && child.material instanceof THREE.MeshStandardMaterial){

    child.material.envMap = environmentMap
    child.material.envMapIntensity = debugObject.envMapIntensity
    child.material.needsUpdate = true

    child.castShadow = true
    child.receiveShadow = true
  }
  
  })
}


const debugObject = {}

debugObject.envMapIntensity = 5

gui.add(debugObject, "envMapIntensity").min(0).max(10).step(0.001).name("EnvMap Intensity").onChange(updateAllMaterials)

});

onUnmounted(() => {
  renderer.setAnimationLoop(null)
});

</script>

<style>
canvas {
  width: 100%;
  height: 100%;
  display: block;
}

.loading-bar{
  position:absolute;
  top: 50%;
  width: 100%;
  height: 2px;
  background: #fff;
  transform: scaleX(0);
  transform-origin: top left;
  transition: transform 0.7s;
  will-change: transform;
}

.loading-bar.ended{
  transform-origin: top right;
  transition: transform 1.8s ease-in-out;
}

.point{
  position:absolute;
  top:50%;
  left:50%;
}

.point:hover .text{
  opacity:1;
}

.point .label{
  position:absolute;
  top:-20px;
  left: -20px;
  width:40px;
  height:40px;
  background: #00000077;
  border: 1px solid #ffffffae;
  border-radius: 50%;
  color:#fff;
  font-family:Helvetica, Arial, sans-serif;;
  text-align: center;
  line-height: 40px;
  font-weight: 100;
  font-size: 14px;
  cursor:help;
}

.point .text{
  position:absolute;
  top:30px;
  left:-100px;
  width:200px;
  padding: 20px;
  border-radius: 4px;
  background: #00000077;
  color: #ffffff;
  text-align: center;
  line-height: 1.3em;
  font-family:  Helvetica, Arial, sans-serif;
  font-weight: 100;
  font-size: 14px;
  opacity:0;
  transition: opacity 0.3s;
  pointer-events: none; 
}


</style>