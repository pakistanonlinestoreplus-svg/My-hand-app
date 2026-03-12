<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Dual Hand Pro Control</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; color: #0ff; font-family: sans-serif; }
        #video-wrap {
            position: absolute; bottom: 20px; left: 20px;
            width: 320px; height: 240px; border: 2px solid #0ff;
            border-radius: 12px; overflow: hidden; transform: scaleX(-1);
            z-index: 100; background: #111;
        }
        video { width: 100%; height: 100%; object-fit: cover; opacity: 0.4; }
        canvas#hand_canvas { position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index: 101; }
        #ui { position: absolute; top: 20px; width: 100%; text-align: center; pointer-events: none; text-shadow: 0 0 10px #0ff; }
    </style>
</head>
<body>

<div id="ui">
    <h2>DUAL HAND SYSTEM ACTIVE ⚡</h2>
    <p>Dono hathon se control karein | Pinch to Zoom</p>
</div>

<div id="video-wrap">
    <video id="input_video" autoplay playsinline></video>
    <canvas id="hand_canvas"></canvas>
</div>

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

const particleCount = 8000;
const positions = new Float32Array(particleCount * 3);
const velocities = new Float32Array(particleCount * 3);
const geometry = new THREE.BufferGeometry();

for (let i = 0; i < particleCount * 3; i++) {
    positions[i] = (Math.random() - 0.5) * 30;
    velocities[i] = 0;
}

geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
const material = new THREE.PointsMaterial({ color: 0x00ffff, size: 0.08, transparent: true, opacity: 0.8, blending: THREE.AdditiveBlending });
const points = new THREE.Points(geometry, material);
scene.add(points);

camera.position.z = 15;

/** 2. HAND TRACKING SETUP **/
const videoElement = document.getElementById('input_video');
const canvasElement = document.getElementById('hand_canvas');
const canvasCtx = canvasElement.getContext('2d');

let attractors = [new THREE.Vector3(0,0,0), new THREE.Vector3(0,0,0)];
let handsActive = [false, false];
let globalZoom = 1;

const hands = new Hands({ locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}` });
hands.setOptions({ maxNumHands: 2, modelComplexity: 1, minDetectionConfidence: 0.6, minTrackingConfidence: 0.6 });

hands.onResults((results) => {
    canvasCtx.clearRect(0, 0, canvasElement.width, canvasElement.height);
    handsActive = [false, false];

    if (results.multiHandLandmarks) {
        results.multiHandLandmarks.forEach((landmarks, index) => {
            if (index > 1) return;
            handsActive[index] = true;

            // Update Attractor position
            const palm = landmarks[9];
            attractors[index].x = (palm.x - 0.5) * -40;
            attractors[index].y = (palm.y - 0.5) * -30;
            attractors[index].z = (0.5 - palm.z) * 20;

            // Draw Lines in Camera Box
            canvasCtx.strokeStyle = index === 0 ? "#00ffff" : "#ff00ff";
            canvasCtx.lineWidth = 2;
            const connections = [[0,1],[1,2],[2,3],[3,4],[0,5],[5,6],[6,7],[7,8],[5,9],[9,10],[10,11],[11,12],[9,13],[13,14],[14,15],[15,16],[13,17],[17,18],[18,19],[19,20],[0,17]];
            connections.forEach(([i, j]) => {
                canvasCtx.beginPath();
                canvasCtx.moveTo(landmarks[i].x * canvasElement.width, landmarks[i].y * canvasElement.height);
                canvasCtx.lineTo(landmarks[j].x * canvasElement.width, landmarks[j].y * canvasElement.height);
                canvasCtx.stroke();
            });
        });

        // Zoom Logic: Dono hathon ka darmiyani fasla
        if (results.multiHandLandmarks.length === 2) {
            const h1 = results.multiHandLandmarks[0][9];
            const h2 = results.multiHandLandmarks[1][9];
            const dist = Math.sqrt(Math.pow(h1.x - h2.x, 2) + Math.pow(h1.y - h2.y, 2));
            globalZoom = THREE.MathUtils.lerp(globalZoom, dist * 5, 0.1);
        }
    }
});

const cameraHelper = new Camera(videoElement, {
    onFrame: async () => {
        canvasElement.width = 320; canvasElement.height = 240;
        await hands.send({image: videoElement});
    }
});
cameraHelper.start();

/** 3. ANIMATION LOOP **/
function animate() {
    requestAnimationFrame(animate);
    const pos = geometry.attributes.position.array;
    
    for (let i = 0; i < particleCount; i++) {
        const i3 = i * 3;
        
        let target = new THREE.Vector3(0,0,0);
        if (handsActive[0] && handsActive[1]) {
            // Agar 2 haath hain, toh particles dono ke darmiyan phailenge
            target.lerpVectors(attractors[0], attractors[1], (i / particleCount));
        } else if (handsActive[0]) {
            target.copy(attractors[0]);
        } else {
            // Agar haath nahi hai, toh center mein reset
            target.set(0, 0, -5);
        }

        velocities[i3] += (target.x - pos[i3]) * 0.003;
        velocities[i3+1] += (target.y - pos[i3+1]) * 0.003;
        velocities[i3+2] += (target.z - pos[i3+2]) * 0.003;

        pos[i3] += velocities[i3];
        pos[i3+1] += velocities[i3+1];
        pos[i3+2] += velocities[i3+2];

        velocities[i3] *= 0.92;
        velocities[i3+1] *= 0.92;
        velocities[i3+2] *= 0.92;
    }

    geometry.attributes.position.needsUpdate = true;
    // Zoom apply
    particleSystem.scale.set(globalZoom, globalZoom, globalZoom);
    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
