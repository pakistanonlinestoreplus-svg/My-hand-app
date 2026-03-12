<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Jarvis 3D Gesture Controller</title>
    <style>
        body { margin: 0; background: #000; overflow: hidden; color: #0ff; font-family: 'Courier New', monospace; }
        #video-wrap {
            position: absolute; top: 20px; left: 20px;
            width: 200px; height: 150px; border: 1px solid #0ff;
            border-radius: 10px; overflow: hidden; transform: scaleX(-1);
            z-index: 100; opacity: 0.6; box-shadow: 0 0 15px #0ff;
        }
        video { width: 100%; height: 100%; object-fit: cover; }
        #ui-panel {
            position: absolute; top: 20px; right: 20px; text-align: right;
            pointer-events: none; text-shadow: 0 0 10px #0ff;
        }
        .status { color: #0f0; font-weight: bold; }
    </style>
</head>
<body>

<div id="ui-panel">
    <h1>JARVIS OS v2.0</h1>
    <p>SYSTEM STATUS: <span class="status">ONLINE</span></p>
    <p>GESTURE ENGINE: <span id="hand-status">WAITING...</span></p>
</div>

<div id="video-wrap">
    <video id="input_video" autoplay playsinline></video>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
/** 1. THREE.JS SCENE (Visuals from Video) **/
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// JARVIS GRID (Background effect)
const gridHelper = new THREE.GridHelper(100, 50, 0x004444, 0x002222);
gridHelper.rotation.x = Math.PI / 2;
scene.add(gridHelper);

// Materials
const jointMat = new THREE.PointsMaterial({ color: 0x00ffff, size: 0.2, transparent: true, blending: THREE.AdditiveBlending });
const skeletonMat = new THREE.LineBasicMaterial({ color: 0x00ffff, transparent: true, opacity: 0.6 });

// Hand Objects
const handGroups = [new THREE.Group(), new THREE.Group()];
handGroups.forEach(g => scene.add(g));

const connections = [[0,1],[1,2],[2,3],[3,4],[0,5],[5,6],[6,7],[7,8],[5,9],[9,10],[10,11],[11,12],[9,13],[13,14],[14,15],[15,16],[13,17],[17,18],[18,19],[19,20],[0,17]];

camera.position.z = 20;

/** 2. HAND TRACKING ENGINE **/
const videoElement = document.getElementById('input_video');
const handStatus = document.getElementById('hand-status');

const hands = new Hands({ locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}` });
hands.setOptions({ maxNumHands: 2, modelComplexity: 1, minDetectionConfidence: 0.7, minTrackingConfidence: 0.7 });

hands.onResults((results) => {
    handGroups.forEach(g => { g.clear(); g.visible = false; });
    
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        handStatus.innerText = "TRACKING " + results.multiHandLandmarks.length + " HAND(S)";
        
        results.multiHandLandmarks.forEach((landmarks, index) => {
            const group = handGroups[index];
            group.visible = true;

            const pts = landmarks.map(lm => new THREE.Vector3((lm.x - 0.5) * -50, (lm.y - 0.5) * -40, (0.5 - lm.z) * 20));

            // Create Joints
            const geo = new THREE.BufferGeometry().setFromPoints(pts);
            group.add(new THREE.Points(geo, jointMat));

            // Create Skeleton Lines
            connections.forEach(([i, j]) => {
                const lineGeo = new THREE.BufferGeometry().setFromPoints([pts[i], pts[j]]);
                group.add(new THREE.Line(lineGeo, skeletonMat));
            });

            // Video me jaisa "Pinch" effect tha:
            const thumbTip = landmarks[4];
            const indexTip = landmarks[8];
            const dist = Math.sqrt(Math.pow(thumbTip.x - indexTip.x, 2) + Math.pow(thumbTip.y - indexTip.y, 2));
            
            if (dist < 0.05) {
                handStatus.innerText = "PINCH DETECTED ⚡";
                group.children.forEach(c => { if(c.material) c.material.color.set(0xff00ff); }); // Color change on pinch
            } else {
                group.children.forEach(c => { if(c.material) c.material.color.set(0x00ffff); });
            }
        });
    } else {
        handStatus.innerText = "SEARCHING...";
    }
});

new Camera(videoElement, {
    onFrame: async () => { await hands.send({image: videoElement}); },
    width: 640, height: 480
}).start();

/** 3. ANIMATION **/
function animate() {
    requestAnimationFrame(animate);
    gridHelper.position.z = Math.sin(Date.now() * 0.001) * 2; // Subtle background movement
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
