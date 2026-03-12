<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Jarvis Pro Controller - Full Functions</title>
    <style>
        body { margin: 0; background: #000; overflow: hidden; color: #0ff; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; }
        #video-wrap {
            position: absolute; bottom: 20px; right: 20px;
            width: 220px; height: 165px; border: 1px solid #0ff;
            border-radius: 12px; overflow: hidden; transform: scaleX(-1);
            z-index: 100; box-shadow: 0 0 15px rgba(0,255,255,0.5);
        }
        video { width: 100%; height: 100%; object-fit: cover; opacity: 0.4; }
        #hud {
            position: absolute; top: 20px; left: 20px; pointer-events: none;
            text-shadow: 0 0 10px #0ff;
        }
        .data-stream { font-size: 12px; color: #0f0; }
    </style>
</head>
<body>

<div id="hud">
    <h1 style="margin:0;">JARVIS GESTURE v3.0</h1>
    <div class="data-stream" id="stats">STATUS: CONNECTING...</div>
    <div id="gesture-hint" style="color:#f0f; margin-top:10px;">ACTION: PINCH TO CREATE | GRAB TO SCALE</div>
</div>

<div id="video-wrap">
    <video id="input_video" autoplay playsinline></video>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
/** 1. THREE.JS ENGINE SETUP **/
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

// Environment
const grid = new THREE.GridHelper(100, 40, 0x004444, 0x001111);
grid.rotation.x = Math.PI / 2;
scene.add(grid);

const handGroup = new THREE.Group();
scene.add(handGroup);

camera.position.z = 25;

// Block Management
const blocks = [];
const boxGeo = new THREE.BoxGeometry(3, 3, 3);
const boxMat = new THREE.MeshStandardMaterial({ 
    color: 0x00ffff, wireframe: true, emissive: 0x00ffff, emissiveIntensity: 0.5 
});
const light = new THREE.PointLight(0x00ffff, 1, 100);
scene.add(light);

/** 2. HAND TRACKING & VIDEO LOGIC **/
const videoElement = document.getElementById('input_video');
const statsEl = document.getElementById('stats');
let lastPinchTime = 0;

function onResults(results) {
    handGroup.clear();
    let handsCount = results.multiHandLandmarks ? results.multiHandLandmarks.length : 0;
    statsEl.innerText = `STATUS: ONLINE | HANDS: ${handsCount}`;

    if (results.multiHandLandmarks) {
        results.multiHandLandmarks.forEach((landmarks, hIndex) => {
            const pts = landmarks.map(l => new THREE.Vector3((l.x - 0.5) * -60, (l.y - 0.5) * -45, (0.5 - l.z) * 20));

            // Draw Skeleton
            const connections = [[0,1],[1,2],[2,3],[3,4],[0,5],[5,6],[6,7],[7,8],[5,9],[9,10],[10,11],[11,12],[9,13],[13,14],[14,15],[15,16],[13,17],[17,18],[18,19],[19,20],[0,17]];
            connections.forEach(([i, j]) => {
                const geo = new THREE.BufferGeometry().setFromPoints([pts[i], pts[j]]);
                const line = new THREE.Line(geo, new THREE.LineBasicMaterial({color: 0x00ffff, opacity: 0.5, transparent: true}));
                handGroup.add(line);
            });

            // GESTURES
            const thumb = pts[4], index = pts[8], middle = pts[12], palm = pts[9];
            const pinchDist = thumb.distanceTo(index);
            const isGrab = middle.y < pts[9].y && index.y < pts[5].y;

            // 1. PINCH TO CREATE
            if (pinchDist < 3.5 && (Date.now() - lastPinchTime > 2000)) {
                spawnBlock(index);
                lastPinchTime = Date.now();
            }

            // 2. GRAB TO MANIPULATE
            if (isGrab && blocks.length > 0) {
                let activeBlock = blocks[blocks.length - 1];
                activeBlock.position.lerp(palm, 0.2); // Follow hand
                
                // Rotation based on hand tilt
                activeBlock.rotation.y += 0.05; 
                activeBlock.material.color.set(0xff00ff); // Highlight
                
                // Dynamic Scaling (Distance from camera)
                let scaleVal = 1 + (pts[0].z * -0.1); 
                activeBlock.scale.set(scaleVal, scaleVal, scaleVal);
            } else if (blocks.length > 0) {
                blocks[blocks.length - 1].material.color.set(0x00ffff);
            }
        });
    }
}

function spawnBlock(pos) {
    const b = new THREE.Mesh(boxGeo, boxMat.clone());
    b.position.copy(pos);
    scene.add(b);
    blocks.push(b);
}

const hands = new Hands({ locateFile: (f) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${f}` });
hands.setOptions({ maxNumHands: 2, modelComplexity: 1, minDetectionConfidence: 0.7, minTrackingConfidence: 0.7 });
hands.onResults(onResults);

new Camera(videoElement, {
    onFrame: async () => { await hands.send({image: videoElement}); },
    width: 640, height: 480
}).start();

/** 3. ANIMATION **/
function animate() {
    requestAnimationFrame(animate);
    
    // Idle rotation for all blocks
    blocks.forEach(b => {
        b.rotation.x += 0.005;
        b.rotation.z += 0.005;
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
