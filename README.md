<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Hand Particles - Laptop Edition</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; cursor: none; }
        
        /* Camera UI for Laptop */
        #video-wrap {
            position: absolute; bottom: 20px; left: 20px;
            width: 240px; height: 180px; border: 2px solid #00ffff;
            border-radius: 12px; overflow: hidden; transform: scaleX(-1);
            background: #111; box-shadow: 0 0 20px rgba(0,255,255,0.3);
            z-index: 100;
        }
        video { width: 100%; height: 100%; object-fit: cover; opacity: 0.5; }
        canvas#hand_canvas {
            position: absolute; top: 0; left: 0;
            width: 100%; height: 100%; z-index: 101;
        }

        #ui-overlay {
            position: absolute; top: 20px; width: 100%; text-align: center;
            color: #00ffff; font-family: 'Orbitron', sans-serif;
            letter-spacing: 3px; text-shadow: 0 0 10px #00ffff;
            pointer-events: none; z-index: 50;
        }
    </style>
</head>
<body>

<div id="ui-overlay">
    <h1 style="margin:0; font-size: 24px;">LAPTOP NEURAL FLOW</h1>
    <p style="font-size: 12px; opacity: 0.8;">Use hand to control the swarm. Move closer to zoom.</p>
</div>

<div id="video-wrap">
    <video id="input_video" playsinline></video>
    <canvas id="hand_canvas"></canvas>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
/** 1. THREE.JS ENGINE (Laptop High Quality) **/
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true, powerPreference: "high-performance" });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
document.body.appendChild(renderer.domElement);

// Laptop ke liye zyada particles (10,000)
const particleCount = 10000;
const positions = new Float32Array(particleCount * 3);
const velocities = new Float32Array(particleCount * 3);
const geometry = new THREE.BufferGeometry();

for (let i = 0; i < particleCount * 3; i++) {
    positions[i] = (Math.random() - 0.5) * 30;
    velocities[i] = 0;
}

geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));

const material = new THREE.PointsMaterial({ 
    color: 0x00ffff, 
    size: 0.08, 
    transparent: true, 
    opacity: 0.8, 
    blending: THREE.AdditiveBlending,
    depthWrite: false
});

const particleSystem = new THREE.Points(geometry, material);
scene.add(particleSystem);

camera.position.z = 12;

/** 2. HAND TRACKING (Laptop Wide Mode) **/
const videoElement = document.getElementById('input_video');
const canvasElement = document.getElementById('hand_canvas');
const canvasCtx = canvasElement.getContext('2d');

let targetPos = new THREE.Vector3(0, 0, 0);
let currentPos = new THREE.Vector3(0, 0, 0);
let isActive = false;

const hands = new Hands({ locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}` });
hands.setOptions({ maxNumHands: 1, modelComplexity: 1, minDetectionConfidence: 0.7, minTrackingConfidence: 0.7 });

hands.onResults((results) => {
    canvasCtx.clearRect(0, 0, canvasElement.width, canvasElement.height);
    
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        isActive = true;
        const landmarks = results.multiHandLandmarks[0];

        // Draw Skeleton in the small box
        canvasCtx.strokeStyle = "#00ffff";
        canvasCtx.lineWidth = 2;
        const connections = [[0,1],[1,2],[2,3],[3,4],[0,5],[5,6],[6,7],[7,8],[5,9],[9,10],[10,11],[11,12],[9,13],[13,14],[14,15],[15,16],[13,17],[17,18],[18,19],[19,20],[0,17]];
        connections.forEach(([i, j]) => {
            canvasCtx.beginPath();
            canvasCtx.moveTo(landmarks[i].x * canvasElement.width, landmarks[i].y * canvasElement.height);
            canvasCtx.lineTo(landmarks[j].x * canvasElement.width, landmarks[j].y * canvasElement.height);
            canvasCtx.stroke();
        });

        // Mapping for Laptop Wide Screen
        const palm = landmarks[9];
        targetPos.x = (palm.x - 0.5) * -40; // Horizontal range barha di
        targetPos.y = (palm.y - 0.5) * -25; // Vertical range barha di
        targetPos.z = (0.5 - palm.z) * 45;  // Zoom effect sensitive kar diya
    } else {
        isActive = false;
    }
});

const cameraUtils = new Camera(videoElement, {
    onFrame: async () => {
        canvasElement.width = 240; canvasElement.height = 180;
        await hands.send({image: videoElement});
    }
});
cameraUtils.start();

/** 3. ANIMATION LOOP (Butter Smooth) **/
function animate() {
    requestAnimationFrame(animate);

    // Smoothing haath ki movement ke liye
    currentPos.lerp(targetPos, 0.1);

    const pos = geometry.attributes.position.array;
    for (let i = 0; i < particleCount; i++) {
        const i3 = i * 3;
        
        if (isActive) {
            // Physics forces for Laptop
            velocities[i3] += (currentPos.x - pos[i3]) * 0.003;
            velocities[i3+1] += (currentPos.y - pos[i3+1]) * 0.003;
            velocities[i3+2] += (currentPos.z - pos[i3+2]) * 0.003;
        } else {
            // Idle Floating
            velocities[i3] += Math.sin(Date.now() * 0.001 + i) * 0.0005;
            velocities[i3+1] += Math.cos(Date.now() * 0.001 + i) * 0.0005;
        }
