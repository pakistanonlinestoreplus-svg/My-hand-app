<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Hand Pro Max Fix</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; touch-action: none; }
        /* Camera Box - Lines sirf iske upar ayengi */
        #video-wrap {
            position: absolute; bottom: 15px; left: 15px;
            width: 160px; height: 120px; border: 2px solid #0ff;
            border-radius: 10px; overflow: hidden; transform: scaleX(-1);
            z-index: 20; background: #111;
        }
        video { width: 100%; height: 100%; object-fit: cover; opacity: 0.6; }
        canvas#hand_canvas {
            position: absolute; top: 0; left: 0;
            width: 100%; height: 100%; z-index: 21;
        }
        #ui {
            position: absolute; top: 20px; width: 100%; text-align: center;
            color: #0ff; font-family: sans-serif; pointer-events: none;
            text-shadow: 0 0 10px #0ff; font-weight: bold;
        }
    </style>
</head>
<body>

<div id="ui">HYPER SPEED & ZOOM MODE 🚀</div>

<div id="video-wrap">
    <video id="input_video" playsinline></video>
    <canvas id="hand_canvas"></canvas> </div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
/** 1. THREE.JS SETUP **/
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const particleCount = 4000;
const positions = new Float32Array(particleCount * 3);
const velocities = new Float32Array(particleCount * 3);
const geometry = new THREE.BufferGeometry();

for (let i = 0; i < particleCount * 3; i++) {
    positions[i] = (Math.random() - 0.5) * 20;
    velocities[i] = 0;
}

geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
const material = new THREE.PointsMaterial({ 
    color: 0x00ffff, size: 0.1, // Size thora barha diya
    transparent: true, opacity: 0.8, blending: THREE.AdditiveBlending 
});
const particleSystem = new THREE.Points(geometry, material);
scene.add(particleSystem);

camera.position.z = 10;

/** 2. TRACKING & CAMERA BOX LINES **/
const videoElement = document.getElementById('input_video');
const canvasElement = document.getElementById('hand_canvas');
const canvasCtx = canvasElement.getContext('2d');

let rawAttractor = new THREE.Vector3(0, 0, 0);
let active = false;

const hands = new Hands({ locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}` });
hands.setOptions({ maxNumHands: 1, modelComplexity: 1, minDetectionConfidence: 0.7 });

hands.onResults((results) => {
    canvasCtx.clearRect(0, 0, canvasElement.width, canvasElement.height);
    
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        active = true;
        const landmarks = results.multiHandLandmarks[0];

        // Drawing Lines ONLY in Camera Box
        canvasCtx.strokeStyle = "#ff00ff";
        canvasCtx.lineWidth = 3;
        const connections = [[0,1],[1,2],[2,3],[3,4],[0,5],[5,6],[6,7],[7,8],[5,9],[9,10],[10,11],[11,12],[9,13],[13,14],[14,15],[15,16],[13,17],[17,18],[18,19],[19,20],[0,17]];
        
        connections.forEach(([i, j]) => {
            canvasCtx.beginPath();
            canvasCtx.moveTo(landmarks[i].x * canvasElement.width, landmarks[i].y * canvasElement.height);
            canvasCtx.lineTo(landmarks[j].x * canvasElement.width, landmarks[j].y * canvasElement.height);
            canvasCtx.stroke();
        });

        // ZOOM FIX: hand.z ko scale kiya hai taake kareeb lane par particles phail jayein
        const palm = landmarks[9];
        rawAttractor.x = (palm.x - 0.5) * -25; // Speed increase
        rawAttractor.y = (palm.y - 0.5) * -20;
        rawAttractor.z = (0.5 - palm.z) * 35; // Depth sensitivity barha di
    } else {
        active = false;
    }
});

new Camera(videoElement, {
    onFrame: async () => {
        canvasElement.width = 160; canvasElement.height = 120; // Sync canvas size
        await hands.send({image: videoElement});
    }
}).start();

/** 3. ANIMATION LOOP (HYPER SPEED) **/
function animate() {
    requestAnimationFrame(animate);

    if (active) {
        const pos = geometry.attributes.position.array;
        for (let i = 0; i < particleCount; i++) {
            const i3 = i * 3;
            
            // Movement speed ko 0.005 kar diya (pehlay 0.0018 tha)
            velocities[i3] += (rawAttractor.x - pos[i3]) * 0.005;
            velocities[i3+1] += (rawAttractor.y - pos[i3+1]) * 0.005;
            velocities[i3+2] += (rawAttractor.z - pos[i3+2]) * 0.005;

            pos[i3] += velocities[i3];
            pos[i3+1] += velocities[i3+1];
            pos[i3+2] += velocities[i3+2];

            velocities[i3] *= 0.85; // Sharp stop logic
            velocities[i3+1] *= 0.85;
            velocities[i3+2] *= 0.85;
        }
        geometry.attributes.position.needsUpdate = true;
    }
    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
