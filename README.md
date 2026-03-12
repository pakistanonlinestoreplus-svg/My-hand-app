# My-hand-app
// Simplified logic for an Android Particle Controller
handLandmarkerResult.landmarks().firstOrNull()?.let { landmarks ->
    val middleFingerBase = landmarks[9] // Landmark 9 is the palm center
    
    // Convert normalized (0.0 - 1.0) coordinates to World Space
    val handX = (middleFingerBase.x() - 0.5f) * screenWidth
    val handY = (middleFingerBase.y() - 0.5f) * screenHeight
    
    // Update the attractor point in your C++/OpenGL renderer
    myRenderer.updateAttractor(handX, handY, middleFingerBase.z())
}
