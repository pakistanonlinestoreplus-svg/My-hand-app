<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Laptop Fix - Hand Particles</title>
    <style>
        body { margin: 0; background: #000; overflow: hidden; }
        #video-wrap {
            position: absolute; bottom: 20px; left: 20px;
            width: 240px; height: 180px; border: 2px solid #0ff;
            z-index: 100; transform: scaleX(-1);
        }
        video { width: 100%; height: 100%; background: #222; }
        canvas#hand_canvas { position: absolute; top: 0; left: 0; width: 100%; height: 100%; }
    </style>
</head>
<body>

<div id="video-wrap">
    <video id="input_video" autoplay playsinline></video>
    <canvas id="hand_canvas"></canvas>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
// SCENE SETUP
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer();
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// PARTICLES
const particleCount = 8000;
const positions = new Float32Array(particleCount * 3);
const velocities = new Float32Array(particleCount * 3);
const geo = new THREE.BufferGeometry();
for(let i=0; i<particleCount*3; i++) positions[i] = (Math.random()-0.5)*20;
geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
const mat = new THREE.PointsMaterial({color: 0x00ffff, size: 0.08, transparent: true, blending: THREE.AdditiveBlending});
const points = new THREE.Points(geo, mat);
scene.add(points);
camera.position.z = 10;

// CAMERA & TRACKING
const video = document.getElementById('input_video');
const canvas = document.getElementById('hand_canvas');
const ctx = canvas.getContext('2d');
let target = new THREE.Vector3(0,0,0);
let active = false;

const hands = new Hands({locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`});
hands.setOptions({maxNumHands: 1, modelComplexity: 1, minDetectionConfidence: 0.5});

hands.onResults((results) => {
    ctx.clearRect(0,0,canvas.width,canvas.height);
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        active = true;
        const lm = results.multiHandLandmarks[0][9];
        target.x = (lm.x - 0.5) * -30;
        target.y = (lm.y - 0.5) * -20;
        target.z = (0.5 - lm.z) * 20;
    } else { active = false; }
});

// Laptop Camera Start Fix
const cam = new Camera(video, {
    onFrame: async () => {
        canvas.width = 240; canvas.height = 180;
        await hands.send({image: video});
    }
});
cam.start().catch(err => alert("Camera error: " + err));

function animate() {
    requestAnimationFrame(animate);
    if(active) {
        const p = geo.attributes.position.array;
        for(let i=0; i<particleCount; i++) {
            let i3 = i*3;
            velocities[i3] += (target.x - p[i3]) * 0.003;
            velocities[i3+1] += (target.y - p[i3+1]) * 0.003;
            velocities[i3+2] += (target.z - p[i3+2]) * 0.003;
            p[i3] += velocities[i3]; p[i3+1] += velocities[i3+1]; p[i3+2] += velocities[i3+2];
            velocities[i3] *= 0.9; velocities[i3+1] *= 0.9; velocities[i3+2] *= 0.9;
        }
        geo.attributes.position.needsUpdate = true;
    }
    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
