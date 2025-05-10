<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Mobile Camera Recorder</title>
    <style>
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
    -webkit-tap-highlight-color: transparent;
}

body {
    min-height: 100vh;
    min-height: -webkit-fill-available;
    background: linear-gradient(135deg, #1a1a1a, #2d3436);
    font-family: system-ui, -apple-system, 'Segoe UI', sans-serif;
    color: #fff;
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 12px;
    overflow: hidden;
}

html {
    height: -webkit-fill-available;
}

.container {
    width: 100%;
    height: 100vh;
    height: -webkit-fill-available;
    display: flex;
    flex-direction: column;
    background: rgba(255, 255, 255, 0.05);
    backdrop-filter: blur(10px);
    -webkit-backdrop-filter: blur(10px);
    padding: 12px;
}

h1 {
    font-size: clamp(1.2rem, 4vw, 1.5rem);
    font-weight: 600;
    text-align: center;
    margin: 8px 0;
    background: linear-gradient(to right, #00c6fb, #005bea);
    -webkit-background-clip: text;
    background-clip: text;
    -webkit-text-fill-color: transparent;
}

.video-container {
    position: relative;
    width: 100%;
    flex: 1;
    overflow: hidden;
    background: #000;
    margin: 8px 0;
}

#videoElement {
    width: 100%;
    height: 100%;
    object-fit: cover;
    transform: scaleX(-1);
    -webkit-transform: scaleX(-1);
}

.button-container {
    display: flex;
    gap: 12px;
    justify-content: center;
    padding: 12px 0;
}

button {
    min-width: 120px;
    min-height: 44px;
    padding: 12px 20px;
    border: none;
    border-radius: 50px;
    font-size: 16px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.3s ease;
    color: white;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
}

#startButton {
    background: linear-gradient(45deg, #00c6fb, #005bea);
    box-shadow: 0 4px 15px rgba(0, 198, 251, 0.3);
}

#stopButton {
    background: linear-gradient(45deg, #ff416c, #ff4b2b);
    box-shadow: 0 4px 15px rgba(255, 65, 108, 0.3);
}

button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

#errorMsg {
    position: fixed;
    bottom: 20px;
    left: 50%;
    transform: translateX(-50%);
    background: rgba(255, 65, 108, 0.9);
    padding: 12px 20px;
    border-radius: 8px;
    display: none;
    color: white;
    font-size: 14px;
    max-width: 90%;
    text-align: center;
    z-index: 1000;
}
.face-oval0, .face-oval1, .face-oval2, .face-oval3, .face-oval4, .face-oval5 {
    position: absolute;
    border: 3px solid rgba(0, 198, 251, 0.7);
    border-radius: 50%;
    pointer-events: none;
    box-shadow: 0 0 0 2000px rgba(0, 0, 0, 0.3);
    animation: pulse 2s infinite;
    opacity: 0;
    visibility: hidden;
    transition: all 0.5s ease-in-out;
    /* Base size will be set dynamically via JavaScript */
}

@keyframes pulse {
    0% { border-color: rgba(0, 198, 251, 0.7); }
    50% { border-color: rgba(0, 198, 251, 0.3); }
    100% { border-color: rgba(0, 198, 251, 0.7); }
}

.face-oval-visible {
    opacity: 1;
    visibility: visible;
}
     </style>
</head>
<body>
    <!-- Previous HTML remains the same -->
    <div class="container">
        <h1>Mobile Camera</h1>
        <div class="video-container">
            <video id="videoElement" autoplay playsinline></video>
            <div class="face-oval0"></div>
            <div class="face-oval1"></div>
            <div class="face-oval2"></div>
            <div class="face-oval3"></div>
            <div class="face-oval4"></div>
            <div class="face-oval5"></div>
        </div>
        <div class="button-container">
            <button id="startButton">Start</button>
            <button id="stopButton" disabled>Stop</button>
        </div>
        <div id="errorMsg"></div>
    </div>
    <script>
// Previous variables remain the same
const videoElement = document.getElementById('videoElement');
const startButton = document.getElementById('startButton');
const stopButton = document.getElementById('stopButton');
const errorMsg = document.getElementById('errorMsg');
const faceOvals = document.querySelectorAll('[class^="face-oval"]');
let stream = null;
let ovalTimeouts = [];

function updateOvalSizes() {
    const container = document.querySelector('.video-container');
    const containerRect = container.getBoundingClientRect();
    
    // Calculate oval size based on container dimensions
    // Make oval height about 60% of the smaller container dimension
    const baseSize = Math.min(containerRect.width, containerRect.height) * 1;
    const ovalWidth = baseSize * 0.75;  // Make width 75% of height for oval shape
    const ovalHeight = baseSize;

    // Define oval positions relative to container size
    const positions = [
        { top: '33%', left: '33%' },
        { top: '33%', left: '57%' },
        { top: '45%', left: '40%' },
        { top: '50%', left: '50%' },
        { top: '57%', left: '33%' },
        { top: '57%', left: '57%' }
    ];

    faceOvals.forEach((oval, index) => {
        oval.style.width = `${ovalWidth}px`;
        oval.style.height = `${ovalHeight}px`;
        oval.style.top = positions[index].top;
        oval.style.left = positions[index].left;
        oval.style.transform = 'translate(-50%, -50%)';
    });
}

function showOvalsSequentially() {
    clearOvalTimeouts();
    updateOvalSizes();
    
    let totalDelay = 0;
    
    faceOvals.forEach((oval, index) => {
        oval.classList.remove('face-oval-visible');
        
        const timeout = setTimeout(() => {
            if (index > 0) {
                faceOvals[index - 1].classList.remove('face-oval-visible');
            }
            oval.classList.add('face-oval-visible');
            
            if (index === faceOvals.length - 1) {
                setTimeout(() => {
                    oval.classList.remove('face-oval-visible');
                }, 4000);
            }
        }, totalDelay);
        
        totalDelay += index === 0 ? 5000 : 4000;
        ovalTimeouts.push(timeout);
    });
}

function clearOvalTimeouts() {
    ovalTimeouts.forEach(timeout => clearTimeout(timeout));
    ovalTimeouts = [];
    faceOvals.forEach(oval => {
        oval.classList.remove('face-oval-visible');
    });
}

async function startCamera() {
    try {
        if (stream) {
            stopCamera();
        }

        const pixelRatio = window.devicePixelRatio || 1;
        const screenWidth = screen.width * pixelRatio;
        const screenHeight = screen.height * pixelRatio;

        let constraints = {
            video: {
                facingMode: 'user',
                width: { ideal: screenWidth },
                height: { ideal: screenHeight }
            },
            audio: false
        };

        if (/Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent)) {
            const devices = await navigator.mediaDevices.enumerateDevices();
            const videoDevices = devices.filter(device => device.kind === 'videoinput');
            
            if (videoDevices.length > 0) {
                constraints.video.deviceId = videoDevices[0].deviceId;
                
                if (window.orientation === 0 || window.orientation === 180) {
                    constraints.video.height = { ideal: Math.max(screenHeight, screenWidth) };
                    constraints.video.width = { ideal: Math.min(screenHeight, screenWidth) };
                } else {
                    constraints.video.width = { ideal: Math.max(screenHeight, screenWidth) };
                    constraints.video.height = { ideal: Math.min(screenHeight, screenWidth) };
                }
            }
        }

        stream = await navigator.mediaDevices.getUserMedia(constraints);
        videoElement.srcObject = stream;
        
        handleOrientation();
        showOvalsSequentially();

        startButton.disabled = true;
        stopButton.disabled = false;
        errorMsg.style.display = 'none';

        // Auto-stop after 50 seconds
        setTimeout(() => {
            stopCamera();
        }, 50000);

    } catch (err) {
        console.error('Camera error:', err);
        let errorMessage = 'Camera access denied. Please check your settings.';
        
        if (err.name === 'NotFoundError') {
            errorMessage = 'No camera found.';
        } else if (err.name === 'NotAllowedError') {
            errorMessage = 'Please allow camera access.';
        } else if (err.name === 'NotReadableError') {
            errorMessage = 'Camera is in use by another app.';
        }
        
        errorMsg.textContent = errorMessage;
        errorMsg.style.display = 'block';
        startButton.disabled = false;
        stopButton.disabled = true;
        
        setTimeout(() => {
            errorMsg.style.display = 'none';
        }, 3000);
    }
}

function stopCamera() {
    if (stream) {
        stream.getTracks().forEach(track => track.stop());
        videoElement.srcObject = null;
        startButton.disabled = false;
        stopButton.disabled = true;
        stream = null;
        clearOvalTimeouts();
        videoElement.classList.remove('rotate-landscape', 'rotate-landscape-back');
    }
}

function handleOrientation() {
    if (stream) {
        const videoTrack = stream.getVideoTracks()[0];
        const settings = videoTrack.getSettings();
        
        if (window.orientation === 90 || window.orientation === -90) {
            if (settings.facingMode === 'user') {
                videoElement.classList.add('rotate-landscape');
                videoElement.classList.remove('rotate-landscape-back');
            } else {
                videoElement.classList.add('rotate-landscape-back');
                videoElement.classList.remove('rotate-landscape');
            }
        } else {
            videoElement.classList.remove('rotate-landscape', 'rotate-landscape-back');
        }
        
        // Update oval sizes after orientation change
        setTimeout(updateOvalSizes, 300);
    }
}

// Event Listeners
startButton.addEventListener('click', startCamera);
stopButton.addEventListener('click', stopCamera);

window.addEventListener('orientationchange', () => {
    setTimeout(() => {
        handleOrientation();
        if (stream) {
            clearOvalTimeouts();
            showOvalsSequentially();
        }
    }, 300);
});

window.addEventListener('resize', () => {
    if (stream) {
        updateOvalSizes();
    }
});

document.addEventListener('visibilitychange', () => {
    if (document.hidden && stream) {
        stopCamera();
    }
});

document.addEventListener('touchend', (e) => {
    e.preventDefault();
    e.target.click();
}, { passive: false });

window.addEventListener('beforeunload', stopCamera);
    </script>
</body>
</html>
