<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Smooth Hand Particles</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; touch-action: none; }
        #video-container {
            position: absolute; bottom: 15px; left: 15px;
            width: 100px; height: 75px; border: 1px solid #0ff;
            border-radius: 8px; overflow: hidden; transform: scaleX(-1);
            opacity: 0.3; z-index: 10;
        }
        video { width: 100%; height: 100%; object-fit: cover; }
        #ui {
            position: absolute; top: 20px; width: 100%; text-align: center;
            color: #0ff; font-family: 'Segoe UI', sans-serif; pointer-events: none;
            text-shadow: 0 0 10px #0ff; font-weight: bold;
        }
    </style>
</head>
<body>

<div id="ui">SMOOTH FLOW MODE ✨</div>
<div id="video-container"><video id="input_video" playsinline></video></div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
/** SCENE SETUP **/
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

/** PARTICLES SETUP **/
const particleCount = 5000;
const positions = new Float32Array(particleCount * 3);
const velocities = new Float32Array(particleCount * 3);
const geometry = new THREE.BufferGeometry();

for (let i = 0; i < particleCount * 3; i++) {
    positions[i] = (Math.random() - 0.5) * 15;
    velocities[i] = 0;
}

geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
const material = new THREE.PointsMaterial({ 
    color: 0x00ffff, 
    size: 0.06, 
    transparent: true, 
    opacity: 0.7, 
    blending: THREE.AdditiveBlending 
});
const particleSystem = new THREE.Points(geometry, material);
scene.add(particleSystem);

camera.position.z = 8;

/** SMOOTHING LOGIC **/
let rawAttractor = new THREE.Vector3(0, 0, 0); // Direct camera data
let smoothAttractor = new THREE.Vector3(0, 0, 0); // Smooth output
let active = false;

// MediaPipe Setup
const videoElement = document.getElementById('input_video');
const hands = new Hands({ locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}` });
hands.setOptions({ maxNumHands: 1, modelComplexity: 0, minDetectionConfidence: 0.6 });

hands.onResults((results) => {
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        active = true;
        const hand = results.multiHandLandmarks[0][9]; // Middle finger base (stable point)
        rawAttractor.x = (hand.x - 0.5) * -15;
        rawAttractor.y = (hand.y - 0.5) * -12;
        rawAttractor.z = (0.5 - hand.z) * 10;
    } else {
        active = false;
    }
});

new Camera(videoElement, {
    onFrame: async () => { await hands.send({image: videoElement}); },
    width: 480, height: 360
}).start();

/** ANIMATION LOOP **/
function animate() {
    requestAnimationFrame(animate);

    // LERP: Isse haath ki movement smooth ho jati hai
    // 0.1 ka matlab hai 10% movement har frame mein, jo jhatkay khatam kar deta hai
    smoothAttractor.lerp(rawAttractor, 0.1);

    const pos = geometry.attributes.position.array;
    for (let i = 0; i < particleCount; i++) {
        const i3 = i * 3;
        
        if (active) {
            // Attraction force thori kam kar di taake smooth lage
            velocities[i3] += (smoothAttractor.x - pos[i3]) * 0.0015;
            velocities[i3+1] += (smoothAttractor.y - pos[i3+1]) * 0.0015;
            velocities[i3+2] += (smoothAttractor.z - pos[i3+2]) * 0.0015;
        }

        pos[i3] += velocities[i3];
        pos[i3+1] += velocities[i3+1];
        pos[i3+2] += velocities[i3+2];

        // Damping (Friction): Isse particles control mein rehte hain
        velocities[i3] *= 0.94;
        velocities[i3+1] *= 0.94;
        velocities[i3+2] *= 0.94;
    }

    geometry.attributes.position.needsUpdate = true;
    particleSystem.rotation.y += 0.001;
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
