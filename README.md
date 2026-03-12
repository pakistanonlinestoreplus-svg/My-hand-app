<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Laptop Dual Hand - Fix</title>
    <style>
        body { margin: 0; background: #000; overflow: hidden; color: #0ff; font-family: sans-serif; }
        #video-wrap {
            position: absolute; bottom: 20px; left: 20px;
            width: 240px; height: 180px; border: 2px solid #0ff;
            border-radius: 10px; overflow: hidden; transform: scaleX(-1);
            z-index: 100;
        }
        video { width: 100%; height: 100%; object-fit: cover; opacity: 0.3; }
        canvas#hand_canvas { position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index: 101; }
        #ui { position: absolute; top: 15px; width: 100%; text-align: center; pointer-events: none; }
    </style>
</head>
<body>

<div id="ui"><h2>SKELETON ACTIVE ✨ PARTICLES LOADING...</h2></div>

<div id="video-wrap">
    <video id="input_video" autoplay playsinline></video>
    <canvas id="hand_canvas"></canvas>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
// 1. THREE.JS BASIC SETUP
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Particles Setup (Thoda bada size taake nazar aayein)
const particleCount = 6000;
const positions = new Float32Array(particleCount * 3);
const velocities = new Float32Array(particleCount * 3);
const geometry = new THREE.BufferGeometry();

for (let i = 0; i < particleCount * 3; i++) {
    positions[i] = (Math.random() - 0.5) * 30; // Shuru mein phailay hue
    velocities[i] = 0;
}

geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
const material = new THREE.PointsMaterial({ 
    color: 0x00ffff, 
    size: 0.12, // Bada size
    transparent: true, 
    opacity: 0.9,
    blending: THREE.AdditiveBlending 
});
const points = new THREE.Points(geometry, material);
scene.add(points);

camera.position.z = 15;

// 2. TRACKING DATA
const videoElement = document.getElementById('input_video');
const canvasElement = document.getElementById('hand_canvas');
const ctx = canvasElement.getContext('2d');

let attractors = [new THREE.Vector3(0,0,0), new THREE.Vector3(0,0,0)];
let activeHands = 0;

const hands = new Hands({ locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}` });
hands.setOptions({ maxNumHands: 2, modelComplexity: 1, minDetectionConfidence: 0.5 });

hands.onResults((results) => {
    ctx.clearRect(0, 0, canvasElement.width, canvasElement.height);
    activeHands = results.multiHandLandmarks ? results.multiHandLandmarks.length : 0;

    if (results.multiHandLandmarks) {
        results.multiHandLandmarks.forEach((landmarks, index) => {
            // Update attractors for particles
            const p = landmarks[9]; // Middle finger palm point
            attractors[index].set((p.x - 0.5) * -40, (p.y - 0.5) * -30, (0.5 - p.z) * 20);

            // Draw Skeleton Lines
            ctx.strokeStyle = index === 0 ? "#00ffff" : "#ff00ff";
            ctx.lineWidth = 3;
            const conn = [[0,1],[1,2],[2,3],[3,4],[0,5],[5,6],[6,7],[7,8],[5,9],[9,10],[10,11],[11,12],[9,13],[13,14],[14,15],[15,16],[13,17],[17,18],[18,19],[19,20],[0,17]];
            conn.forEach(([a, b]) => {
                ctx.beginPath();
                ctx.moveTo(landmarks[a].x * canvasElement.width, landmarks[a].y * canvasElement.height);
                ctx.lineTo(landmarks[b].x * canvasElement.width, landmarks[b].y * canvasElement.height);
                ctx.stroke();
            });
        });
    }
});

new Camera(videoElement, {
    onFrame: async () => {
        canvasElement.width = 240; canvasElement.height = 180;
        await hands.send({image: videoElement});
    }
}).start();

// 3. ANIMATION (Physics Fix)
function animate() {
    requestAnimationFrame(animate);
    const pos = geometry.attributes.position.array;
    
    for (let i = 0; i < particleCount; i++) {
        const i3 = i * 3;
        let target = new THREE.Vector3(0,0,0);

        if (activeHands === 2) {
            // Mix particles between two hands
            target.lerpVectors(attractors[0], attractors[1], Math.random());
        } else if (activeHands === 1) {
            target.copy(attractors[0]);
        } else {
            // Jab haath na ho toh particles ko center mein ghumao
            target.x = Math.sin(Date.now() * 0.001 + i) * 5;
            target.y = Math.cos(Date.now() * 0.001 + i) * 5;
        }

        // Stronger Attraction
        velocities[i3] += (target.x - pos[i3]) * 0.005;
        velocities[i3+1] += (target.y - pos[i3+1]) * 0.005;
        velocities[i3+2] += (target.z - pos[i3+2]) * 0.005;

        pos[i3] += velocities[i3];
        pos[i3+1] += velocities[i3+1];
        pos[i3+2] += velocities[i3+2];

        // Damping
        velocities[i3] *= 0.85;
        velocities[i3+1] *= 0.85;
        velocities[i3+2] *= 0.85;
    }

    geometry.attributes.position.needsUpdate = true;
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
