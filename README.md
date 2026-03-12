<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>3D Hand Particle Attractor</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: sans-serif; }
        #container { position: relative; width: 100vw; height: 100vh; }
        #video-container {
            position: absolute; bottom: 20px; right: 20px;
            width: 240px; height: 180px; border: 2px solid #444;
            border-radius: 10px; overflow: hidden; transform: scaleX(-1);
        }
        video { width: 100%; height: 100%; object-fit: cover; }
        #ui {
            position: absolute; top: 20px; left: 20px; color: white;
            pointer-events: none; text-shadow: 1px 1px 2px black;
        }
    </style>
</head>
<body>

<div id="ui">
    <h1>Hand Particle Attractor</h1>
    <p>Allow camera access and wave your hand.</p>
</div>

<div id="video-container">
    <video id="input_video"></video>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
/** * THREE.JS SETUP 
 **/
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Particle Configuration
const particleCount = 15000;
const positions = new Float32Array(particleCount * 3);
const velocities = new Float32Array(particleCount * 3);
const geometry = new THREE.BufferGeometry();

for (let i = 0; i < particleCount * 3; i++) {
    positions[i] = (Math.random() - 0.5) * 10;
    velocities[i] = 0;
}

geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
const material = new THREE.PointsMaterial({ color: 0x00ffff, size: 0.02, transparent: true, opacity: 0.8 });
const particleSystem = new THREE.Points(geometry, material);
scene.add(particleSystem);

camera.position.z = 5;

// Global attractor point (controlled by hand)
let attractor = new THREE.Vector3(0, 0, -100); 
let active = false;

/** * MEDIAPIPE HANDS SETUP 
 **/
const videoElement = document.getElementById('input_video');

function onResults(results) {
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        active = true;
        // Use the palm base (landmark 0) or middle finger base (landmark 9)
        const hand = results.multiHandLandmarks[0][9];
        
        // Map 0-1 coordinates to Three.js space
        attractor.x = (hand.x - 0.5) * -10; // Inverted for mirroring
        attractor.y = (hand.y - 0.5) * -10;
        attractor.z = (hand.z * -10); 
    } else {
        active = false;
    }
}

const hands = new Hands({
    locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
});

hands.setOptions({
    maxNumHands: 1,
    modelComplexity: 1,
    minDetectionConfidence: 0.5,
    minTrackingConfidence: 0.5
});
hands.onResults(onResults);

const cameraUtils = new Camera({
    videoElement: videoElement,
    onFrame: async () => {
        await hands.send({image: videoElement});
    },
    width: 640,
    height: 480
});
cameraUtils.start();

/** * ANIMATION LOOP 
 **/
function animate() {
    requestAnimationFrame(animate);

    const posAttr = geometry.attributes.position;
    
    for (let i = 0; i < particleCount; i++) {
        const i3 = i * 3;
        
        if (active) {
            // Calculate vector toward attractor
            let dx = attractor.x - posAttr.array[i3];
            let dy = attractor.y - posAttr.array[i3+1];
            let dz = attractor.z - posAttr.array[i3+2];
            
            // Simple physics: acceleration towards hand
            velocities[i3] += dx * 0.001;
            velocities[i3+1] += dy * 0.001;
            velocities[i3+2] += dz * 0.001;
        } else {
            // Drift randomly if no hand is detected
            velocities[i3] += (Math.random() - 0.5) * 0.001;
            velocities[i3+1] += (Math.random() - 0.5) * 0.001;
            velocities[i3+2] += (Math.random() - 0.5) * 0.001;
        }

        // Apply velocity
        posAttr.array[i3] += velocities[i3];
        posAttr.array[i3+1] += velocities[i3+1];
        posAttr.array[i3+2] += velocities[i3+2];

        // Friction (slows them down over time)
        velocities[i3] *= 0.95;
        velocities[i3+1] *= 0.95;
        velocities[i3+2] *= 0.95;
    }

    posAttr.needsUpdate = true;
    particleSystem.rotation.y += 0.001;
    renderer.render(scene, camera);
}

animate();

// Handle Resize
window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});
</script>
</body>
</html>
