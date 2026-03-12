<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Jarvis High-Speed 3D Editor</title>
    <style>
        body { margin: 0; background: #000; overflow: hidden; color: #0ff; font-family: 'Courier New', monospace; }
        #video-wrap {
            position: absolute; bottom: 20px; left: 20px;
            width: 180px; height: 135px; border: 1px solid #0ff;
            border-radius: 10px; overflow: hidden; transform: scaleX(-1);
            z-index: 100; opacity: 0.4;
        }
        video { width: 100%; height: 100%; object-fit: cover; }
        #hud {
            position: absolute; top: 20px; left: 20px; pointer-events: none;
            text-shadow: 0 0 10px #0ff;
        }
    </style>
</head>
<body>

<div id="hud">
    <h1 style="margin:0;">JARVIS EDITOR v4.0</h1>
    <p id="info">GESTURE: PINCH=CREATE | GRAB=CONTROL | FLICK=THROW</p>
</div>

<div id="video-wrap">
    <video id="input_video" autoplay playsinline></video>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.0/build/three.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js"></script>

<script>
/** 1. THREE.JS PHYSICS SETUP **/
const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const light = new THREE.PointLight(0x00ffff, 2, 100);
scene.add(light);
scene.add(new THREE.AmbientLight(0x111111));

const blocks = [];
const boxGeo = new THREE.BoxGeometry(3, 3, 3);
camera.position.z = 35;

/** 2. HAND TRACKING & MOVEMENT ENGINE **/
const videoElement = document.getElementById('input_video');
let grabbedBlock = null;
let lastPinch = 0;
let handVelocity = new THREE.Vector3();
let lastHandPos = new THREE.Vector3();

function onResults(results) {
    if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
        const lm = results.multiHandLandmarks[0];
        
        // Accurate 3D Mapping
        const currentPos = new THREE.Vector3((lm[9].x - 0.5) * -60, (lm[9].y - 0.5) * -45, (0.5 - lm[9].z) * 20);
        
        // Calculate Speed/Velocity for "Throwing"
        handVelocity.subVectors(currentPos, lastHandPos);
        lastHandPos.copy(currentPos);

        const thumb = new THREE.Vector3((lm[4].x - 0.5) * -60, (lm[4].y - 0.5) * -45, (0.5 - lm[4].z) * 20);
        const index = new THREE.Vector3((lm[8].x - 0.5) * -60, (lm[8].y - 0.5) * -45, (0.5 - lm[8].z) * 20);
        
        // PINCH TO CREATE (Index + Thumb)
        const d = thumb.distanceTo(index);
        if (d < 3.5 && Date.now() - lastPinch > 800) {
            spawnBlock(index);
            lastPinch = Date.now();
        }

        // GRAB LOGIC (Check if palm is closed)
        const isGrab = lm[8].y > lm[6].y && lm[12].y > lm[10].y;
        
        if (isGrab) {
            if (!grabbedBlock && blocks.length > 0) {
                // Find closest block
                blocks.forEach(b => { if(b.position.distanceTo(currentPos) < 10) grabbedBlock = b; });
            }
            if (grabbedBlock) {
                grabbedBlock.position.lerp(currentPos, 0.4); // Instant follow
                grabbedBlock.velocity.set(0,0,0); // Stop physics while holding
                grabbedBlock.material.opacity = 1;
            }
        } else {
            if (grabbedBlock) {
                // RELEASE & THROW: Block gets the hand's speed
                grabbedBlock.velocity.copy(handVelocity).multiplyScalar(1.5);
                grabbedBlock.material.opacity = 0.6;
                grabbedBlock = null;
            }
        }
    }
}

function spawnBlock(pos) {
    const mat = new THREE.MeshPhongMaterial({ color: 0x00ffff, wireframe: true, transparent: true, opacity: 0.6 });
    const b = new THREE.Mesh(boxGeo, mat);
    b.position.copy(pos);
    b.velocity = new THREE.Vector3(0,0,0); // Custom property for physics
    scene.add(b);
    blocks.push(b);
}

const hands = new Hands({ locateFile: (f) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${f}` });
hands.setOptions({ maxNumHands: 1, modelComplexity: 1, minDetectionConfidence: 0.75 });
hands.onResults(onResults);

new Camera(videoElement, {
    onFrame: async () => { await hands.send({image: videoElement}); },
    width: 640, height: 480
}).start();

/** 3. PHYSICS ANIMATION LOOP **/
function animate() {
    requestAnimationFrame(animate);
    
    blocks.forEach(b => {
        if (b !== grabbedBlock) {
            // Constant Floating Rotation
            b.rotation.x += 0.01;
            b.rotation.y += 0.01;
            
            // Physics: Apply Velocity (Throwing effect)
            b.position.add(b.velocity);
            b.velocity.multiplyScalar(0.98); // Air friction (slows down)
            
            // Boundary: Screen se bahar na jaye
            if (Math.abs(b.position.x) > 40) b.velocity.x *= -1;
            if (Math.abs(b.position.y) > 30) b.velocity.y *= -1;
        }
    });

    renderer.render(scene, camera);
}
animate();
</script>
</body>
</html>
