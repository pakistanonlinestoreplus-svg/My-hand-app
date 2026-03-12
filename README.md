<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Jarvis Block Engine - Fixed</title>
    <style>
        body { margin: 0; background: #000; overflow: hidden; color: #0ff; font-family: 'Courier New', monospace; }
        #video-wrap {
            position: absolute; top: 15px; left: 15px;
            width: 200px; height: 150px; border: 2px solid #0ff;
            border-radius: 10px; overflow: hidden; transform: scaleX(-1);
            z-index: 100;
        }
        video { width: 100%; height: 100%; object-fit: cover; opacity: 0.4; }
        #ui {
            position: absolute; top: 20px; right: 20px; text-align: right;
            pointer-events: none; text-shadow: 0 0 10px #0ff;
        }
        /* Notification style */
        #msg { color: #f0f; font-weight: bold; font-size: 20px; }
    </style>
</head>
<body>

<div id="ui">
    <h1>JARVIS BLOCK SYSTEM</h1>
    <p id="msg">SYSTEM READY</p>
    <p>ACTION: PINCH FINGERS TO SPAWN</p>
</div>

<div id="video-wrap">
    <video id="input_video" autoplay playsinline></video>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
/** 1. THREE.JS SCENE **/
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Jarvis Grid
const grid = new THREE.GridHelper(100, 40, 0x00ffff, 0x002222);
grid.rotation.x = Math.PI / 2;
scene.add(grid);

const handGroup = new THREE.Group();
scene.add(handGroup);

// Blocks Storage
const blocks = [];
const boxGeo = new THREE.BoxBufferGeometry(2.5, 2.5, 2.5);
const boxMat = new THREE.MeshBasicMaterial({ color: 0x00ffff, wireframe: true, transparent: true, opacity: 0.7 });

camera.position.z = 20;

/** 2. HAND TRACKING & ENHANCED GESTURES **/
const videoElement = document.getElementById('input_video');
const msgEl = document.getElementById('msg');
let lastActionTime = 0;

function onResults(results) {
    handGroup.clear();
    
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        results.multiHandLandmarks.forEach((landmarks) => {
            // Coordinate transformation (optimized for laptop screen)
            const pts = landmarks.map(l => new THREE.Vector3((l.x - 0.5) * -50, (l.y - 0.5) * -35, (0.5 - l.z) * 20));

            // Skeleton Drawing
            const connections = [[0,1],[1,2],[2,3],[3,4],[0,5],[5,6],[6,7],[7,8],[5,9],[9,10],[10,11],[11,12],[9,13],[13,14],[14,15],[15,16],[13,17],[17,18],[18,19],[19,20],[0,17]];
            const lineMat = new THREE.LineBasicMaterial({ color: 0x00ffff });
            connections.forEach(([i, j]) => {
                const geo = new THREE.BufferGeometry().setFromPoints([pts[i], pts[j]]);
                handGroup.add(new THREE.Line(geo, lineMat));
            });

            // 1. PINCH LOGIC (Finger 4 & 8)
            const thumbTip = pts[4];
            const indexTip = pts[8];
            const pinchDist = thumbTip.distanceTo(indexTip);

            // Sensitivity increased (4.0 is very easy to trigger)
            if (pinchDist < 4.0 && (Date.now() - lastActionTime > 1500)) {
                spawnBlock(indexTip);
                lastActionTime = Date.now();
                msgEl.innerText = "BLOCK GENERATED!";
                setTimeout(() => msgEl.innerText = "READY", 1000);
            }

            // 2. GRAB & MOVE LOGIC (Muthi band karna)
            // Check if middle finger tip is lower than palm center
            const isGrab = pts[12].y < pts[9].y && pts[8].y < pts[5].y;
            
            if (isGrab && blocks.length > 0) {
                msgEl.innerText = "GRABBING BLOCK...";
                // Move the most recent block to follow the palm center
                blocks[blocks.length - 1].position.lerp(pts[9], 0.3);
                blocks[blocks.length - 1].material.color.set(0xff00ff); // Magenta color when grabbed
            } else if (blocks.length > 0) {
                blocks[blocks.length - 1].material.color.set(0x00ffff); // Back to Cyan
            }
        });
    }
}

function spawnBlock(position) {
    const mesh = new THREE.Mesh(boxGeo, boxMat.clone());
    mesh.position.copy(position);
    scene.add(mesh);
    blocks.push(mesh);
}

const hands = new Hands({ locateFile: (f) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${f}` });
hands.setOptions({ maxNumHands: 1, modelComplexity: 1, minDetectionConfidence: 0.6, minTrackingConfidence: 0.6 });
hands.onResults(onResults);

new Camera(videoElement, {
    onFrame: async () => { await hands.send({image: videoElement}); },
    width: 640, height: 480
}).start();

/** 3. ANIMATION **/
function animate() {
    requestAnimationFrame(animate);
    // Blocks ko rotate karein taake Jarvis feel aaye
    blocks.forEach(b => {
        b.rotation.x += 0.01;
        b.rotation.y += 0.01;
    });
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
