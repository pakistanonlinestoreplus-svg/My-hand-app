<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Skeletal Hand Flow</title>
    <style>
        body { margin: 0; overflow: hidden; background: #000; touch-action: none; }
        #video-container {
            position: absolute; bottom: 15px; left: 15px;
            width: 120px; height: 90px; border: 2px solid #0ff;
            border-radius: 8px; overflow: hidden; transform: scaleX(-1);
            opacity: 0.4; z-index: 10;
        }
        video { width: 100%; height: 100%; object-fit: cover; }
        #ui {
            position: absolute; top: 15px; width: 100%; text-align: center;
            color: #0ff; font-family: 'Segoe UI', sans-serif; pointer-events: none;
            text-shadow: 0 0 10px #0ff;
        }
    </style>
</head>
<body>

<div id="ui">FULL SKELETAL FLOW ⚡</div>
<div id="video-container"><video id="input_video" playsinline></video></div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
/** SCENE SETUP **/
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

/** PARTICLES SETUP **/
const particleCount = 4500; 
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
    size: 0.08, 
    transparent: true, 
    opacity: 0.8, 
    blending: THREE.AdditiveBlending 
});
const particleSystem = new THREE.Points(geometry, material);
scene.add(particleSystem);

/** FULL HAND TRACKING LINES SETUP **/
// 21 points standard skeletal structure
const connections = [[0,1],[1,2],[2,3],[3,4],[0,5],[5,6],[6,7],[7,8],[5,9],[9,10],[10,11],[11,12],[9,13],[13,14],[14,15],[15,16],[13,17],[17,18],[18,19],[19,20],[0,17]];
const handLineMaterial = new THREE.LineBasicMaterial({ color: 0xff00ff, linewidth: 2 });
const handLinesGroup = new THREE.Group();
const pointsGeometries = [];

for (let i = 0; i < connections.length; i++) {
    const pointsGeom = new THREE.BufferGeometry().setFromPoints([new THREE.Vector3(0,0,0), new THREE.Vector3(0,0,0)]);
    pointsGeometries.push(pointsGeom);
    const line = new THREE.Line(pointsGeom, handLineMaterial);
    handLinesGroup.add(line);
}
scene.add(handLinesGroup);

camera.position.z = 8;

/** SMOOTHING LOGIC (Using full skeleton centroid) **/
let rawAttractor = new THREE.Vector3(0, 0, 0); 
let smoothAttractor = new THREE.Vector3(0, 0, 0); 
let active = false;

// MediaPipe Setup (Model complexity barha di full tracking ke liye)
const videoElement = document.getElementById('input_video');
const hands = new Hands({ locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}` });
hands.setOptions({ 
    maxNumHands: 1, 
    modelComplexity: 1, // Full complexity for skeleton
    minDetectionConfidence: 0.7 
});

hands.onResults((results) => {
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        active = true;
        const landmarks = results.multiHandLandmarks[0];
        
        // --- Feature 2: All Fingers Tracking with Lines ---
        const handPoints = landmarks.map(lm => {
            // Coordinate mapping with mirroring fix
            return new THREE.Vector3((lm.x - 0.5) * -16, (lm.y - 0.5) * -12, (0.5 - lm.z) * 10);
        });

        // Update lines connecting the points
        for (let i = 0; i < connections.length; i++) {
            const start = handPoints[connections[i][0]];
            const end = handPoints[connections[i][1]];
            pointsGeometries[i].setFromPoints([start, end]);
            pointsGeometries[i].attributes.position.needsUpdate = true;
        }

        // Particle target (Use centroid of all 21 points for stability)
        const centroid = new THREE.Vector3();
        handPoints.forEach(p => centroid.add(p));
        rawAttractor.copy(centroid.divideScalar(21));

    } else {
        active = false;
        handLinesGroup.visible = false;
    }
});

new Camera(videoElement, {
    onFrame: async () => { await hands.send({image: videoElement}); },
    width: 480, height: 360
}).start();

/** ANIMATION LOOP **/
function animate() {
    requestAnimationFrame(animate);

    if (active) {
        // LERP for smooth flow towards hand
        smoothAttractor.lerp(rawAttractor, 0.15);
        handLinesGroup.visible = true;

        const pos = geometry.attributes.position.array;
        for (let i = 0; i < particleCount; i++) {
            const i3 = i * 3;
            
            // --- Feature 1: Stop drift when hand is detected ---
            // Yahan hum strict attraction use karein ge, random noise hata diya hai
            velocities[i3] += (smoothAttractor.x - pos[i3]) * 0.0018;
            velocities[i3+1] += (smoothAttractor.y - pos[i3+1]) * 0.0018;
            velocities[i3+2] += (smoothAttractor.z - pos[i3+2]) * 0.0018;

            pos[i3] += velocities[i3];
            pos[i3+1] += velocities[i3+1];
            pos[i3+2] += velocities[i3+2];

            // Increased Damping (Friction) so they don't 'bounce' when hand stops
            velocities[i3] *= 0.90; 
            velocities[i3+1] *= 0.90;
            velocities[i3+2] *= 0.90;
        }
        geometry.attributes.position.needsUpdate = true;
    } else {
        // Optional: Jab haath camera se chala jaye, particles ko reset karein ya rotate karein.
        particleSystem.rotation.y += 0.002;
    }

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
