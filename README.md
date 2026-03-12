<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Jarvis Interface Pro - Full Skeletal</title>
    <style>
        body { margin: 0; background: #000; overflow: hidden; color: #0ff; font-family: 'Courier New', monospace; font-weight: bold; }
        
        /* Camera Box (Top Left) */
        #video-wrap {
            position: absolute; top: 15px; left: 15px;
            width: 160px; height: 120px; border: 1px solid #0ff;
            border-radius: 8px; overflow: hidden; transform: scaleX(-1);
            z-index: 100; opacity: 0.5;
        }
        video { width: 100%; height: 100%; object-fit: cover; }
        canvas#hand_canvas { position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index: 101; }

        /* Status UI (Center Top) */
        #ui {
            position: absolute; top: 10px; width: 100%; text-align: center;
            pointer-events: none; letter-spacing: 3px;
            text-shadow: 0 0 15px #0ff;
        }
        #hud-circles {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            pointer-events: none; opacity: 0.2;
        }
    </style>
</head>
<body>

<div id="ui"><h2>SKELETAL TRACKING ACTIVE ⚡</h2></div>

<div id="video-wrap">
    <video id="input_video" autoplay playsinline></video>
    <canvas id="hand_canvas"></canvas>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
/** 1. THREE.JS JAVIS HOLOGRAPHIC SETUP **/
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.setPixelRatio(window.devicePixelRatio);
document.body.appendChild(renderer.domElement);

// Javis Glow Material
const pointsMat = new THREE.PointsMaterial({ color: 0x00ffff, size: 0.15, transparent: true, blending: THREE.AdditiveBlending });
const lineMat = new THREE.LineBasicMaterial({ color: 0x00ffff, linewidth: 2, transparent: true, opacity: 0.8, blending: THREE.AdditiveBlending });

// Dono hathon ke liye points aur lines groups
const handsData = [ { pointsGroup: new THREE.Group(), linesGroup: new THREE.Group() }, { pointsGroup: new THREE.Group(), linesGroup: new THREE.Group() } ];
handsData.forEach(hd => { scene.add(hd.pointsGroup); scene.add(hd.linesGroup); });

// Standard connection mapping (index finger, thumb etc)
const connections = [[0,1],[1,2],[2,3],[3,4],[0,5],[5,6],[6,7],[7,8],[5,9],[9,10],[10,11],[11,12],[9,13],[13,14],[14,15],[15,16],[13,17],[17,18],[18,19],[19,20],[0,17]];

camera.position.z = 15;

/** 2. HAND TRACKING (Laptop Pro Mode) **/
const videoElement = document.getElementById('input_video');
const canvasElement = document.getElementById('hand_canvas');
const ctx = canvasElement.getContext('2d');

let isActive = false;
let globalScale = 1;

const hands = new Hands({ locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}` });
hands.setOptions({ maxNumHands: 2, modelComplexity: 1, minDetectionConfidence: 0.7 });

hands.onResults((results) => {
    ctx.clearRect(0, 0, canvasElement.width, canvasElement.height);
    
    // Hide all before updating
    handsData.forEach(hd => { hd.pointsGroup.visible = false; hd.linesGroup.visible = false; hd.pointsGroup.clear(); hd.linesGroup.clear(); });
    isActive = results.multiHandLandmarks ? results.multiHandLandmarks.length > 0 : false;

    if (results.multiHandLandmarks) {
        results.multiHandLandmarks.forEach((landmarks, index) => {
            const hd = handsData[index];
            if (!hd) return;
            hd.pointsGroup.visible = true;
            hd.linesGroup.visible = true;

            // Map standard points (index, middle etc base)
            const mappedPoints = landmarks.map(lm => {
                return new THREE.Vector3((lm.x - 0.5) * -45, (lm.y - 0.5) * -35, (0.5 - lm.z) * 15);
            });

            // Update Points (Ungliyon ke joints)
            const pointsGeo = new THREE.BufferGeometry().setFromPoints(mappedPoints);
            const ptsMesh = new THREE.Points(pointsGeo, pointsMat);
            hd.pointsGroup.add(ptsMesh);

            // Update Connections (Ungliyon ke lines)
            connections.forEach(([i, j]) => {
                const points = [mappedPoints[i], mappedPoints[j]];
                const lineGeo = new THREE.BufferGeometry().setFromPoints(points);
                const lineMesh = new THREE.Line(lineGeo, lineMat);
                hd.linesGroup.add(lineMesh);
            });

            // Draw Camera Box lines
            ctx.strokeStyle = "#00ffff"; ctx.lineWidth = 2;
            connections.forEach(([i, j]) => { ctx.beginPath(); ctx.moveTo(landmarks[i].x * 160, landmarks[i].y * 120); ctx.lineTo(landmarks[j].x * 160, landmarks[j].y * 120); ctx.stroke(); });
        });

        // Zoom/Scale logic based on two-hand distance
        if (results.multiHandLandmarks.length === 2) {
            const h1 = results.multiHandLandmarks[0][9];
            const h2 = results.multiHandLandmarks[1][9];
            const dist = Math.sqrt(Math.pow(h1.x - h2.x, 2) + Math.pow(h1.y - h2.y, 2));
            globalScale = THREE.MathUtils.lerp(globalScale, dist * 5, 0.1);
        } else {
            globalScale = THREE.MathUtils.lerp(globalScale, 1, 0.1); // Reset
        }
    }
});

new Camera(videoElement, {
    onFrame: async () => { await hands.send({image: videoElement}); },
    width: 640, height: 480
}).start();

/** 3. ANIMATION LOOP (Jarvis Cyclone Flow) **/
function animate() {
    requestAnimationFrame(animate);

    handsData.forEach(hd => {
        // Skeletal lines rotation (cyclone) style
        hd.linesGroup.rotation.z += 0.003;
        // Points thoda unstable move karein (hologram glitch effect)
        hd.pointsGroup.children.forEach(pt => { if (isActive) { pt.position.x += (Math.random()-0.5)*0.05; pt.position.y += (Math.random()-0.5)*0.05; } });
    });

    // Zoom update
    handsData.forEach(hd => { hd.pointsGroup.scale.set(globalScale, globalScale, globalScale); hd.linesGroup.scale.set(globalScale, globalScale, globalScale); });

    renderer.render(scene, camera);
}
animate();

window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight; camera.updateProjectionMatrix(); renderer.setSize(window.innerWidth, window.innerHeight);
});
</script>
</body>
</html>
