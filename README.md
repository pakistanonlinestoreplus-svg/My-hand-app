<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Laptop Pro - Hand Particles</title>
    <style>
        body { margin: 0; background: #000; overflow: hidden; color: #0ff; font-family: 'Segoe UI', sans-serif; }
        
        /* Laptop ke liye camera box thora bara aur clear */
        #video-wrap {
            position: absolute; bottom: 25px; left: 25px;
            width: 240px; height: 180px; border: 2px solid #0ff;
            border-radius: 12px; overflow: hidden; transform: scaleX(-1);
            background: #111; box-shadow: 0 0 20px rgba(0,255,255,0.4);
            z-index: 100;
        }
        video { width: 100%; height: 100%; object-fit: cover; }
        
        #ui {
            position: absolute; top: 30px; width: 100%; text-align: center;
            pointer-events: none; letter-spacing: 2px; text-shadow: 0 0 10px #0ff;
        }
    </style>
</head>
<body>

<div id="ui">
    <h1 style="margin:0;">SYSTEM ACTIVE 💻</h1>
    <p>Hand movement se particles control karein</p>
</div>

<div id="video-wrap">
    <video id="input_video" autoplay playsinline></video>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
/** 1. THREE.JS SETUP (High Quality for Laptop) **/
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
document.body.appendChild(renderer.domElement);

// Laptop power: 10,000 particles
const particleCount = 10000;
const positions = new Float32Array(particleCount * 3);
const velocities = new Float32Array(particleCount * 3);
const geometry = new THREE.BufferGeometry();

for (let i = 0; i < particleCount * 3; i++) {
    positions[i] = (Math.random() - 0.5) * 25;
    velocities[i] = 0;
}

geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
const material = new THREE.PointsMaterial({ 
    color: 0x00ffff, 
    size: 0.07, 
    transparent: true, 
    opacity: 0.8,
    blending: THREE.AdditiveBlending 
});
const points = new THREE.Points(geometry, material);
scene.add(points);

camera.position.z = 10;

/** 2. HAND TRACKING SETUP **/
const videoElement = document.getElementById('input_video');
let attractor = new THREE.Vector3(0, 0, 0);
let smoothAttractor = new THREE.Vector3(0, 0, 0);
let active = false;

const hands = new Hands({
    locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
});

hands.setOptions({
    maxNumHands: 1,
    modelComplexity: 1, // Laptop can handle better tracking
    minDetectionConfidence: 0.5,
    minTrackingConfidence: 0.5
});

hands.onResults((results) => {
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        active = true;
        const hand = results.multiHandLandmarks[0][9]; // Palm center
        // Wide screen mapping
        attractor.x = (hand.x - 0.5) * -35; 
        attractor.y = (hand.y - 0.5) * -25;
        attractor.z = (0.5 - hand.z) * 30; 
    } else {
        active = false;
    }
});

/** 3. CAMERA START (Win 10 Fix) **/
// Laptop par aksar 'Camera' constructor ko simple pas karna parta hai
const cameraHelper = new Camera(videoElement, {
    onFrame: async () => {
        await hands.send({image: videoElement});
    },
    width: 640,
    height: 480
});

// Start checking for errors
cameraHelper.start().catch(err => {
    alert("Camera block hai! Settings mein ja kar allow karein.");
});

/** 4. ANIMATION LOOP **/
function animate() {
    requestAnimationFrame(animate);

    if (active) {
        // Smooth follow logic (Lerping)
        smoothAttractor.lerp(attractor, 0.1);
        
        const pos = geometry.attributes.position.array;
        for (let i = 0; i < particleCount; i++) {
            const i3 = i * 3;
            // High speed attraction
            velocities[i3] += (smoothAttractor.x - pos[i3]) * 0.004;
            velocities[i3+1] += (smoothAttractor.y - pos[i3+1]) * 0.004;
            velocities[i3+2] += (smoothAttractor.z - pos[i3+2]) * 0.004;

            pos[i3] += velocities[i3];
            pos[i3+1] += velocities[i3+1];
            pos[i3+2] += velocities[i3+2];

            // Friction
            velocities[i3] *= 0.88;
            velocities[i3+1] *= 0.88;
            velocities[i3+2] *= 0.88;
        }
        geometry.attributes.position.needsUpdate = true;
    }

    renderer.render(scene, camera);
}

animate();

// Resize handle
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});
</script>
</body>
</html>
