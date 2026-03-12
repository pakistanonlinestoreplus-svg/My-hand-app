<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Jarvis OS - Precise Clone</title>
    <style>
        body { margin: 0; background: #000; overflow: hidden; color: #0ff; font-family: 'Segoe UI', sans-serif; }
        
        /* Camera Layout as seen in Video */
        #video-wrap {
            position: absolute; top: 20px; left: 20px;
            width: 240px; height: 180px; border: 1px solid #0ff;
            border-radius: 8px; overflow: hidden; transform: scaleX(-1);
            z-index: 100; box-shadow: 0 0 20px rgba(0,255,255,0.4);
        }
        video { width: 100%; height: 100%; object-fit: cover; }
        canvas#output_canvas {
            position: absolute; top: 0; left: 0; width: 100vw; height: 100vh;
            z-index: 50; pointer-events: none;
        }
        #ui {
            position: absolute; top: 25px; right: 25px; text-align: right;
            text-shadow: 0 0 10px #0ff; z-index: 110;
        }
    </style>
</head>
<body>

<div id="ui">
    <h1 style="margin:0; font-size: 24px;">SYSTEM ANALYSIS</h1>
    <p id="status" style="color:#0f0;">SENSORS: ACTIVE</p>
    <p id="gesture-info">WAITING FOR INPUT...</p>
</div>

<div id="video-wrap">
    <video id="input_video" autoplay playsinline></video>
</div>

<canvas id="output_canvas"></canvas>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js"></script>

<script>
/** 1. CLONE SETUP **/
const videoElement = document.getElementById('input_video');
const canvasElement = document.getElementById('output_canvas');
const ctx = canvasElement.getContext('2d');
const gestureInfo = document.getElementById('gesture-info');

// Three.js for 3D Blocks (Same as Video)
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth/window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ canvas: canvasElement, antialias: true, alpha: true });
renderer.setSize(window.innerWidth, window.innerHeight);

const grid = new THREE.GridHelper(100, 40, 0x00ffff, 0x002222);
grid.rotation.x = Math.PI / 2;
scene.add(grid);

camera.position.z = 30;

const blocks = [];
const boxGeo = new THREE.BoxGeometry(4, 4, 4);
const boxMat = new THREE.MeshBasicMaterial({ color: 0x00ffff, wireframe: true, transparent: true, opacity: 0.7 });

let lastPinchTime = 0;
let grabbedBlock = null;

/** 2. HAND TRACKING (Video exact skeleton) **/
const hands = new Hands({
    locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`
});

hands.setOptions({
    maxNumHands: 2,
    modelComplexity: 1,
    minDetectionConfidence: 0.7,
    minTrackingConfidence: 0.7
});

hands.onResults((results) => {
    // Canvas clear
    ctx.save();
    ctx.clearRect(0, 0, canvasElement.width, canvasElement.height);

    if (results.multiHandLandmarks) {
        results.multiHandLandmarks.forEach((landmarks, index) => {
            // Draw Skeleton Lines (Wahi jo video mein hath par aati hain)
            drawConnectors(ctx, landmarks, HAND_CONNECTIONS, {color: '#00ffff', lineWidth: 4});
            drawLandmarks(ctx, landmarks, {color: '#00ffff', lineWidth: 1, radius: 3});

            // 3D Mapping for Blocks
            const palm = landmarks[9];
            const worldX = (palm.x - 0.5) * -60;
            const worldY = (palm.y - 0.5) * -45;
            const worldZ = (0.5 - palm.z) * 20;

            // PINCH DETECTION (Thumb + Index)
            const thumb = landmarks[4];
            const indexF = landmarks[8];
            const dist = Math.sqrt(Math.pow(thumb.x-indexF.x,2) + Math.pow(thumb.y-indexF.y,2));

            if (dist < 0.05 && (Date.now() - lastPinchTime > 1500)) {
                spawnBlock(worldX, worldY);
                lastPinchTime = Date.now();
                gestureInfo.innerText = "OBJECT CREATED";
            }

            // GRAB DETECTION (Fist)
            const isGrab = landmarks[8].y > landmarks[6].y && landmarks[12].y > landmarks[10].y;
            if (isGrab) {
                if (!grabbedBlock && blocks.length > 0) {
                    blocks.forEach(b => {
                        if (b.position.distanceTo(new THREE.Vector3(worldX, worldY, worldZ)) < 10) grabbedBlock = b;
                    });
                }
                if (grabbedBlock) {
                    grabbedBlock.position.lerp(new THREE.Vector3(worldX, worldY, worldZ), 0.3);
                    grabbedBlock.rotation.y += 0.1;
                    gestureInfo.innerText = "MANIPULATING OBJECT";
                }
            } else {
                grabbedBlock = null;
            }
        });
    }
    ctx.restore();
    renderer.render(scene, camera);
});

function spawnBlock(x, y) {
    const mesh = new THREE.Mesh(boxGeo, boxMat.clone());
    mesh.position.set(x, y, 0);
    scene.add(mesh);
    blocks.push(mesh);
}

/** 3. CAMERA START **/
const cameraUtils = new Camera(videoElement, {
    onFrame: async () => {
        canvasElement.width = window.innerWidth;
        canvasElement.height = window.innerHeight;
        await hands.send({image: videoElement});
    },
    width: 1280,
    height: 720
});
cameraUtils.start();

window.addEventListener('resize', () => {
    renderer.setSize(window.innerWidth, window.innerHeight);
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
});
</script>
</body>
</html>
