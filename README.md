<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Jarvis Voxel Builder</title>
    <style>
        body { margin: 0; background: #000; overflow: hidden; color: #0ff; font-family: 'Courier New', monospace; }
        #video-wrap {
            position: absolute; top: 10px; left: 10px;
            width: 150px; height: 110px; border: 1px solid #0ff;
            border-radius: 5px; overflow: hidden; transform: scaleX(-1);
            z-index: 100; opacity: 0.5;
        }
        video { width: 100%; height: 100%; object-fit: cover; }
        #hud {
            position: absolute; top: 20px; width: 100%; text-align: center;
            pointer-events: none; text-shadow: 0 0 10px #0ff;
        }
    </style>
</head>
<body>

<div id="hud">
    <h2>VOXEL BUILDER ACTIVE</h2>
    <p>PINCH TO PLACE BLOCK 🧊</p>
</div>

<div id="video-wrap">
    <video id="input_video" autoplay playsinline></video>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
/** 1. THREE.JS SCENE SETUP **/
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Lighting
scene.add(new THREE.AmbientLight(0x404040));
const light = new THREE.DirectionalLight(0xffffff, 1);
light.position.set(1, 1, 1).normalize();
scene.add(light);

// Ghost Preview Block (Jo ungliyon ke beech dikhta hai)
const ghostGeo = new THREE.BoxGeometry(2, 2, 2);
const ghostMat = new THREE.MeshBasicMaterial({ color: 0x00ffff, wireframe: true, transparent: true, opacity: 0.5 });
const ghostBlock = new THREE.Mesh(ghostGeo, ghostMat);
scene.add(ghostBlock);

// Placed Blocks Array
const voxels = [];
const voxelGeo = new THREE.BoxGeometry(2, 2, 2);
const voxelMat = new THREE.MeshPhongMaterial({ color: 0x0088ff, shininess: 100 });

camera.position.z = 25;

/** 2. HAND TRACKING & VOXEL LOGIC **/
const videoElement = document.getElementById('input_video');
let lastPinchTime = 0;

function onResults(results) {
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        const lm = results.multiHandLandmarks[0];
        
        // Finger Positions
        const thumb = new THREE.Vector3((lm[4].x - 0.5) * -50, (lm[4].y - 0.5) * -40, (0.5 - lm[4].z) * 15);
        const index = new THREE.Vector3((lm[8].x - 0.5) * -50, (lm[8].y - 0.5) * -40, (0.5 - lm[8].z) * 15);
        
        // Midpoint (Jahan block hona chahiye)
        const midPoint = new THREE.Vector3().lerpVectors(thumb, index, 0.5);
        
        // Voxel Snapping: Blocks ko 2x2x2 ki grid mein fit karna
        const snapX = Math.round(midPoint.x / 2) * 2;
        const snapY = Math.round(midPoint.y / 2) * 2;
        const snapZ = Math.round(midPoint.z / 2) * 2;
        
        ghostBlock.position.set(snapX, snapY, snapZ);
        ghostBlock.visible = true;

        // PINCH DETECT (Block Place Karna)
        const dist = thumb.distanceTo(index);
        if (dist < 2.5 && (Date.now() - lastPinchTime > 1000)) {
            placeVoxel(snapX, snapY, snapZ);
            lastPinchTime = Date.now();
        }
    } else {
        ghostBlock.visible = false;
    }
}

function placeVoxel(x, y, z) {
    // Check if block already exists at this spot
    const exists = voxels.some(v => v.position.x === x && v.position.y === y && v.position.z === z);
    if (!exists) {
        const v = new THREE.Mesh(voxelGeo, voxelMat.clone());
        v.position.set(x, y, z);
        // Edges highlight (Jarvis feel)
        const edges = new THREE.EdgesGeometry(voxelGeo);
        const line = new THREE.LineSegments(edges, new THREE.LineBasicMaterial({ color: 0xffffff }));
        v.add(line);
        
        scene.add(v);
        voxels.push(v);
    }
}

const hands = new Hands({ locateFile: (f) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${f}` });
hands.setOptions({ maxNumHands: 1, modelComplexity: 1, minDetectionConfidence: 0.7 });
hands.onResults(onResults);

new Camera(videoElement, {
    onFrame: async () => { await hands.send({image: videoElement}); },
    width: 640, height: 480
}).start();

/** 3. ANIMATION **/
function animate() {
    requestAnimationFrame(animate);
    renderer.render(scene, camera);
}
animate();

window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});
</script>
</body>
</html>
