<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Hand Magic Pro</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; touch-action: none; }
        #video-container {
            position: absolute; bottom: 15px; right: 15px;
            width: 110px; height: 85px; border: 2px solid #0ff;
            border-radius: 10px; overflow: hidden; transform: scaleX(-1);
            opacity: 0.5; z-index: 10;
        }
        video { width: 100%; height: 100%; object-fit: cover; }
        #ui {
            position: absolute; top: 20px; left: 20px; color: #0ff;
            font-family: 'Segoe UI', sans-serif; pointer-events: none;
            text-shadow: 0 0 10px #000;
        }
    </style>
</head>
<body>

<div id="ui">
    <h2 style="margin:0;">Hand Magic ✨</h2>
    <p style="margin:5px 0; font-size:12px;">Haath kareeb layein zoom karne ke liye!</p>
</div>

<div id="video-container">
    <video id="input_video" playsinline></video>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
/** SCENE SETUP **/
const scene = new THREE.Scene();
// Camera ko thora door rakha hai taake particles screen se bahar na bhagain
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: false }); 
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
document.body.appendChild(renderer.domElement);

/** PARTICLES **/
const particleCount = 6000; 
const positions = new Float32Array(particleCount * 3);
const velocities = new Float32Array(particleCount * 3);
const geometry = new THREE.BufferGeometry();

for (let i = 0; i < particleCount * 3; i++) {
    positions[i] = (Math.random() - 0.5) * 15;
    velocities[i] = 0;
}

geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
// Particles ka size thora barha diya hai (0.07) taake zoom achha lage
const material = new THREE.PointsMaterial({ 
    color: 0x00ffff, 
    size: 0.07, 
    transparent: true, 
    opacity: 0.8,
    blending: THREE.AdditiveBlending 
});
const particleSystem = new THREE.Points(geometry, material);
scene.add(particleSystem);

camera.position.z = 7; // Thora door se view

/** TRACKING LOGIC **/
let attractor = new THREE.Vector3(0, 0, 0);
let active = false;
const videoElement = document.getElementById('input_video');

function onResults(results) {
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        active = true;
        const hand = results.multiHandLandmarks[0][9]; // Middle finger base
        
        // Coordination Mapping (Z ko sensitive kiya hai zoom ke liye)
        attractor.x = (hand.x - 0.5) * -14; 
        attractor.y = (hand.y - 0.5) * -10;
        attractor.z = (0.5 - hand.z) * 10; // Z-axis tracking for depth
    } else {
        active = false;
    }
}

const hands = new Hands({
    locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
});

hands.setOptions({
    maxNumHands: 1,
    modelComplexity: 0, 
    minDetectionConfidence: 0.5,
    minTrackingConfidence: 0.5
});
hands.onResults(onResults);

const cameraUtils = new Camera(videoElement, {
    onFrame: async () => { await hands.send({image: videoElement}); },
    width: 480, height: 360
});
cameraUtils.start();

/** ANIMATION **/
function animate() {
    requestAnimationFrame(animate);
    const posAttr = geometry.attributes.position;
    
    for (let i = 0; i < particleCount; i++) {
        const i3 = i * 3;
        if (active) {
            // Speed aur attraction ko balance kiya hai
            velocities[i3] += (attractor.x - posAttr.array[i3]) * 0.003;
            velocities[i3+1] += (attractor.y - posAttr.array[i3+1]) * 0.003;
            velocities[i3+2] += (attractor.z - posAttr.array[i3+2]) * 0.003;
        } else {
            // Idle movement jab haath na ho
            velocities[i3] += (Math.random() - 0.5) * 0.001;
            velocities[i3+1] += (Math.random() - 0.5) * 0.001;
            velocities[i3+2] += (Math.random() - 0.5) * 0.001;
        }

        posAttr.array[i3] += velocities[i3];
        posAttr.array[i3+1] += velocities[i3+1];
        posAttr.array[i3+2] += velocities[i3+2];

        // Friction taake particles control mein rahain
        velocities[i3] *= 0.91;
        velocities[i3+1] *= 0.91;
        velocities[i3+2] *= 0.91;
    }

    posAttr.needsUpdate = true;
    particleSystem.rotation.y += 0.002; // Halki si rotation style ke liye
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
        video { width: 100%; height: 100%; object-fit: cover; }
        #ui {
            position: absolute; top: 15px; left: 15px; color: #0ff;
            font-family: sans-serif; pointer-events: none; font-size: 12px;
        }
    </style>
</head>
<body>

<div id="ui">
    <h2>Hand Magic ✨</h2>
    <p>Camera allow karein aur haath dikhayein.</p>
</div>

<div id="video-container">
    <video id="input_video" playsinline></video>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: false }); // Mobile performance ke liye false
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio > 1 ? 2 : 1);
document.body.appendChild(renderer.domElement);

// Android ke liye kam particles (Behtar Performance)
const particleCount = 5000; 
const positions = new Float32Array(particleCount * 3);
const velocities = new Float32Array(particleCount * 3);
const geometry = new THREE.BufferGeometry();

for (let i = 0; i < particleCount * 3; i++) {
    positions[i] = (Math.random() - 0.5) * 10;
    velocities[i] = 0;
}

geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
const material = new THREE.PointsMaterial({ color: 0x00ffff, size: 0.04, transparent: true, opacity: 0.7 });
const particleSystem = new THREE.Points(geometry, material);
scene.add(particleSystem);

camera.position.z = 5;

let attractor = new THREE.Vector3(0, 0, -100); 
let active = false;

const videoElement = document.getElementById('input_video');

function onResults(results) {
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        active = true;
        const hand = results.multiHandLandmarks[0][9];
        // Mirroring fix for Front Camera
        attractor.x = (hand.x - 0.5) * -12; 
        attractor.y = (hand.y - 0.5) * -12;
        attractor.z = (hand.z * -5); 
    } else {
        active = false;
    }
}

const hands = new Hands({
    locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
});

hands.setOptions({
    maxNumHands: 1,
    modelComplexity: 0, // 0 is faster for mobile
    minDetectionConfidence: 0.5,
    minTrackingConfidence: 0.5
});
hands.onResults(onResults);

const cameraUtils = new Camera(videoElement, {
    onFrame: async () => {
        await hands.send({image: videoElement});
    },
    width: 480,
    height: 360
});
cameraUtils.start();

function animate() {
    requestAnimationFrame(animate);
    const posAttr = geometry.attributes.position;
    
    for (let i = 0; i < particleCount; i++) {
        const i3 = i * 3;
        if (active) {
            let dx = attractor.x - posAttr.array[i3];
            let dy = attractor.y - posAttr.array[i3+1];
            let dz = attractor.z - posAttr.array[i3+2];
            
            // Attraction Logic
            velocities[i3] += dx * 0.002;
            velocities[i3+1] += dy * 0.002;
            velocities[i3+2] += dz * 0.002;
        }

        posAttr.array[i3] += velocities[i3];
        posAttr.array[i3+1] += velocities[i3+1];
        posAttr.array[i3+2] += velocities[i3+2];

        velocities[i3] *= 0.92;
        velocities[i3+1] *= 0.92;
        velocities[i3+2] *= 0.92;
    }

    posAttr.needsUpdate = true;
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
