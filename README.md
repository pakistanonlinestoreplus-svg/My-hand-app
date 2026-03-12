<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Hand Power Pro Max</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; touch-action: none; }
        #video-container {
            position: absolute; bottom: 10px; left: 10px;
            width: 100px; height: 75px; border: 1px solid #f0f;
            border-radius: 5px; overflow: hidden; transform: scaleX(-1);
            opacity: 0.4; z-index: 10;
        }
        video { width: 100%; height: 100%; object-fit: cover; }
        #ui {
            position: absolute; top: 15px; width: 100%; text-align: center;
            color: #f0f; font-family: sans-serif; pointer-events: none;
            text-shadow: 0 0 15px #f0f; letter-spacing: 2px;
        }
    </style>
</head>
<body>

<div id="ui">
    <h3 style="margin:0;">HYPER SPEED MODE ⚡</h3>
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
const renderer = new THREE.WebGLRenderer({ antialias: false, powerPreference: "high-performance" });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Power Settings: Particles thore kam lekin speed double!
const particleCount = 4000; 
const positions = new Float32Array(particleCount * 3);
const velocities = new Float32Array(particleCount * 3);
const colors = new Float32Array(particleCount * 3);
const geometry = new THREE.BufferGeometry();

for (let i = 0; i < particleCount; i++) {
    const i3 = i * 3;
    positions[i3] = (Math.random() - 0.5) * 20;
    positions[i3+1] = (Math.random() - 0.5) * 20;
    positions[i3+2] = (Math.random() - 0.5) * 20;
    
    // Dynamic Colors (Pink/Cyan mix)
    colors[i3] = Math.random();
    colors[i3+1] = 0.5;
    colors[i3+2] = 1.0;
}

geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));

const material = new THREE.PointsMaterial({ 
    size: 0.09, 
    vertexColors: true,
    transparent: true, 
    opacity: 0.9,
    blending: THREE.AdditiveBlending 
});
const particleSystem = new THREE.Points(geometry, material);
scene.add(particleSystem);

camera.position.z = 8;

let attractor = new THREE.Vector3(0, 0, 0);
let active = false;
let speedFactor = 0.008; // High speed attraction

const videoElement = document.getElementById('input_video');
const hands = new Hands({ locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}` });

hands.setOptions({
    maxNumHands: 1,
    modelComplexity: 0,
    minDetectionConfidence: 0.5,
    minTrackingConfidence: 0.5
});

hands.onResults((results) => {
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        active = true;
        const hand = results.multiHandLandmarks[0][8]; // Index Finger tip for precision
        attractor.x = (hand.x - 0.5) * -16; 
        attractor.y = (hand.y - 0.5) * -12;
        attractor.z = (0.5 - hand.z) * 15;
    } else {
        active = false;
    }
});

const cameraUtils = new Camera(videoElement, {
    onFrame: async () => { await hands.send({image: videoElement}); },
    width: 480, height: 360
});
cameraUtils.start();

function animate() {
    requestAnimationFrame(animate);
    const pos = geometry.attributes.position.array;
    
    for (let i = 0; i < particleCount; i++) {
        const i3 = i * 3;
        if (active) {
            // Hyper Force Gravity
            velocities[i3] += (attractor.x - pos[i3]) * speedFactor;
            velocities[i3+1] += (attractor.y - pos[i3+1]) * speedFactor;
            velocities[i3+2] += (attractor.z - pos[i3+2]) * speedFactor;
        } else {
            // Chaos mode
            velocities[i3] += (Math.random() - 0.5) * 0.02;
            velocities[i3+1] += (Math.random() - 0.5) * 0.02;
        }

        pos[i3] += velocities[i3];
        pos[i3+1] += velocities[i3+1];
        pos[i3+2] += velocities[i3+2];

        // Kinetic friction (High response)
        velocities[i3] *= 0.88;
        velocities[i3+1] *= 0.88;
        velocities[i3+2] *= 0.88;
    }

    geometry.attributes.position.needsUpdate = true;
    particleSystem.rotation.z += 0.01; // Cyclone effect
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
