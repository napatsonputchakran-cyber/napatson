<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AR English: Fill in the Blank</title>
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- MediaPipe Hands & Camera Utils -->
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/control_utils/control_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js" crossorigin="anonymous"></script>

    <style>
        body, html { 
            margin: 0; padding: 0; width: 100%; height: 100%; 
            overflow: hidden; font-family: 'Sarabun', sans-serif; 
            background-color: #0f172a; /* Slate 900 */
        }
        
        /* Video mirrored for natural movement */
        #videoElement {
            transform: scaleX(-1);
            position: absolute;
            top: 0;
            left: 0;
            width: 100vw;
            height: 100vh;
            object-fit: cover;
            z-index: 1;
            opacity: 0.7; /* Make background slightly dark to pop UI */
        }
        
        /* Canvas is NOT mirrored via CSS to keep text readable. 
           Coordinates are flipped in JS instead. */
        #gameCanvas { 
            position: absolute;
            top: 0;
            left: 0;
            width: 100vw;
            height: 100vh;
            z-index: 10; 
        }
        
        /* Glassmorphism Styles */
        .glass {
            background: rgba(255, 255, 255, 0.15);
            box-shadow: 0 8px 32px 0 rgba(0, 0, 0, 0.3);
            backdrop-filter: blur(12px);
            -webkit-backdrop-filter: blur(12px);
            border: 1px solid rgba(255, 255, 255, 0.2);
            border-radius: 1.5rem;
        }

        #uiLayer { position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index: 20; pointer-events: none; }
        .pointer-events-auto { pointer-events: auto; }
    </style>
</head>
<body>

    <!-- Camera Feed -->
    <video id="videoElement" autoplay playsinline></video>
    
    <!-- Game Canvas -->
    <canvas id="gameCanvas"></canvas>

    <!-- HTML UI Layer -->
    <div id="uiLayer" class="flex flex-col items-center justify-center">
        
        <!-- Start Screen -->
        <div id="startScreen" class="glass p-10 flex flex-col items-center pointer-events-auto transition-opacity duration-500">
            <h1 class="text-4xl md:text-6xl font-extrabold text-white mb-2 drop-shadow-lg text-center tracking-wide">
                AR Fill in the Blank
            </h1>
            <p class="text-gray-200 text-lg mb-8 text-center font-light">
                ฝึกแต่งประโยคภาษาอังกฤษด้วยมือคุณ (หรือใช้เมาส์ก็ได้)
            </p>
            <button id="startBtn" class="bg-blue-500 hover:bg-blue-600 text-white font-bold py-4 px-12 rounded-full text-2xl shadow-[0_0_20px_rgba(59,130,246,0.6)] transform transition hover:scale-105 active:scale-95">
                เข้าสู่เกม (Start)
            </button>
        </div>

        <!-- Hand Tracking Warning -->
        <div id="handWarning" class="glass absolute top-6 bg-red-500/60 border-red-400/50 px-6 py-3 hidden">
            <p class="text-white font-bold flex items-center text-lg shadow-black">
                <span class="animate-pulse mr-3 text-2xl">⚠️</span> กำลังหามือของคุณ... (ใช้เมาส์แทนได้)
            </p>
        </div>

        <!-- HUD -->
        <div id="hud" class="absolute top-5 left-5 right-5 flex justify-between items-start hidden">
            <div class="glass px-6 py-4">
                <p class="text-gray-300 text-sm font-semibold uppercase tracking-wider">Level <span id="levelDisplay">1</span>/3</p>
                <h2 id="hintDisplay" class="text-yellow-300 text-xl md:text-2xl font-bold mt-1 shadow-black drop-shadow-md">คำแปลใบ้</h2>
            </div>
            <div class="glass px-6 py-4 flex items-center">
                <span class="text-4xl mr-3 drop-shadow-md">🏆</span>
                <div>
                    <p class="text-gray-300 text-xs font-semibold uppercase tracking-wider">Score</p>
                    <p id="scoreDisplay" class="text-white text-3xl font-black drop-shadow-md">0</p>
                </div>
            </div>
        </div>

        <!-- Game Over Screen -->
        <div id="gameOverScreen" class="glass p-12 flex flex-col items-center hidden pointer-events-auto">
            <h1 class="text-6xl font-black text-transparent bg-clip-text bg-gradient-to-r from-green-400 to-blue-500 mb-4 drop-shadow-lg">
                All Clear! 🎉
            </h1>
            <p class="text-white text-2xl mb-8">คะแนนรวมของคุณ: <span id="finalScore" class="text-yellow-400 font-bold text-4xl"></span></p>
            <button onclick="location.reload()" class="bg-green-500 hover:bg-green-600 text-white font-bold py-3 px-10 rounded-full text-xl shadow-[0_0_20px_rgba(34,197,94,0.6)] hover:scale-105 transition transform">
                เล่นอีกครั้ง
            </button>
        </div>

    </div>

<script>
    // --- Level Data ---
    const levels = [
        {
            sentence: "I ___ an ___.",
            hint: "ฉัน กิน แอปเปิ้ล",
            answers: ["eat", "apple"],
            options: ["eat", "apple", "dog", "run"]
        },
        {
            sentence: "The ___ is ___.",
            hint: "รถยนต์ สีแดง",
            answers: ["car", "red"],
            options: ["car", "red", "cat", "blue", "fly"]
        },
        {
            sentence: "She ___ to ___.",
            hint: "เธอ ชอบ อ่านหนังสือ",
            answers: ["likes", "read"],
            options: ["likes", "read", "like", "swimming"]
        }
    ];

    // --- Global Game State ---
    let currentLevel = 0;
    let score = 0;
    let gameState = 'start'; // start, instruct, playing, gameover
    let lastHandTime = 0;

    // --- Input State (MediaPipe + Mouse) ---
    let cursor = { x: -100, y: -100 };
    let isPinching = false;
    let pinchJustReleased = false;
    let handLandmarks = null;
    let inputSource = 'none'; // 'none', 'mouse', 'hand'

    // --- Game Objects ---
    let cards = [];
    let slots = [];
    let sentenceParts = []; // Text parts to draw between slots
    let grabbedCard = null;
    let startButtonAR = null;

    // --- Canvas Setup ---
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    
    function resizeCanvas() {
        canvas.width = window.innerWidth;
        canvas.height = window.innerHeight;
    }
    window.addEventListener('resize', resizeCanvas);
    resizeCanvas();

    // --- Object Classes ---
    class Card {
        constructor(word, index, total) {
            this.word = word;
            this.id = index;
            // Dynamic width based on word length
            ctx.font = 'bold 24px Arial';
            this.width = Math.max(120, ctx.measureText(this.word).width + 60);
            this.height = 60;
            
            // Distribute around the screen (top half)
            let sectionWidth = canvas.width / total;
            this.x = (sectionWidth * index) + (sectionWidth / 2) - (this.width / 2);
            this.y = Math.random() * (canvas.height * 0.25) + 100; // Random height in top area
            
            this.isGrabbed = false;
            this.currentSlot = null;
        }

        draw(ctx) {
            ctx.save();
            ctx.translate(this.x, this.y);
            
            // 2.5D Effect (Glassmorphism card)
            ctx.shadowColor = 'rgba(0,0,0,0.4)';
            ctx.shadowBlur = this.isGrabbed ? 25 : 10;
            ctx.shadowOffsetY = this.isGrabbed ? 15 : 5;
            
            // Card Background
            ctx.fillStyle = this.isGrabbed ? 'rgba(255, 255, 255, 1)' : 'rgba(240, 249, 255, 0.9)';
            if(this.isGrabbed) ctx.scale(1.05, 1.05); // Slight pop effect

            ctx.beginPath();
            ctx.roundRect(0, 0, this.width, this.height, 12);
            ctx.fill();
            
            // Border
            ctx.strokeStyle = 'rgba(255, 255, 255, 0.8)';
            ctx.lineWidth = 2;
            ctx.stroke();

            // Text
            ctx.shadowColor = 'transparent';
            ctx.fillStyle = '#1e293b'; // Slate 800
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            ctx.font = 'bold 24px Arial';
            ctx.fillText(this.word, this.width/2, this.height/2);
            
            ctx.restore();
        }
    }

    class Slot {
        constructor(expectedWord) {
            this.x = 0; // Will be set by sentence layout
            this.y = 0;
            this.width = 140; // Fixed slot width
            this.height = 60;
            this.expectedWord = expectedWord;
            this.cardInside = null;
        }

        draw(ctx) {
            ctx.save();
            ctx.translate(this.x, this.y);
            
            // Inner shadow for empty slot
            if(!this.cardInside) {
                ctx.fillStyle = 'rgba(0, 0, 0, 0.3)';
                ctx.beginPath();
                ctx.roundRect(0, 0, this.width, this.height, 12);
                ctx.fill();
            }

            ctx.strokeStyle = 'rgba(255, 255, 255, 0.6)';
            ctx.lineWidth = 2;
            ctx.setLineDash([8, 8]);
            ctx.beginPath();
            ctx.roundRect(0, 0, this.width, this.height, 12);
            ctx.stroke();

            if (!this.cardInside) {
                ctx.fillStyle = 'rgba(255, 255, 255, 0.4)';
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                ctx.font = 'bold 16px Arial';
                ctx.fillText("Drop here", this.width/2, this.height/2);
            }
            ctx.restore();
        }
    }

    class ARButton {
        constructor(x, y, text) {
            this.x = x;
            this.y = y;
            this.width = 280;
            this.height = 90;
            this.text = text;
        }
        draw(ctx) {
            ctx.save();
            ctx.translate(this.x, this.y);
            
            // Glowing effect
            ctx.shadowColor = 'rgba(34, 197, 94, 0.8)';
            ctx.shadowBlur = 30;
            
            // Button Body
            ctx.fillStyle = '#22c55e'; // Tailwind Green 500
            ctx.beginPath();
            ctx.roundRect(0, 0, this.width, this.height, 45);
            ctx.fill();
            
            // Inner border
            ctx.strokeStyle = 'rgba(255,255,255,0.5)';
            ctx.lineWidth = 4;
            ctx.stroke();

            // Text
            ctx.shadowColor = 'transparent';
            ctx.fillStyle = 'white';
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            ctx.font = 'bold 32px Sarabun';
            ctx.fillText(this.text, this.width/2, this.height/2);
            ctx.restore();
        }
    }

    // --- Game Logic Functions ---
    function initLevel() {
        cards = [];
        slots = [];
        sentenceParts = [];
        grabbedCard = null;
        
        let levelData = levels[currentLevel];
        document.getElementById('hintDisplay').innerText = `💡 ${levelData.hint}`;
        document.getElementById('levelDisplay').innerText = currentLevel + 1;

        // 1. Create Cards (shuffle options)
        let options = [...levelData.options].sort(() => Math.random() - 0.5);
        options.forEach((word, i) => cards.push(new Card(word, i, options.length)));

        // 2. Parse Sentence and Create Slots
        // Example: "The ___ is ___." -> parts: ["The ", " is ", "."]
        let rawParts = levelData.sentence.split("___");
        
        levelData.answers.forEach((ans, i) => {
            slots.push(new Slot(ans));
        });

        // 3. Layout Sentence Center Bottom (Glass Frame)
        // We will calculate layout dynamically in drawLoop to handle responsive sizing,
        // but we store the string parts here.
        sentenceParts = rawParts;
    }

    function checkWinCondition() {
        let allCorrect = true;
        for (let slot of slots) {
            if (!slot.cardInside || slot.cardInside.word !== slot.expectedWord) {
                allCorrect = false;
                break;
            }
        }

        if (allCorrect) {
            // Lock UI temporarily
            gameState = 'transition';
            score += 100;
            document.getElementById('scoreDisplay').innerText = score;
            
            setTimeout(() => {
                currentLevel++;
                if (currentLevel < levels.length) {
                    gameState = 'playing';
                    initLevel();
                } else {
                    gameState = 'gameover';
                    document.getElementById('hud').classList.add('hidden');
                    document.getElementById('gameOverScreen').classList.remove('hidden');
                    document.getElementById('finalScore').innerText = score;
                }
            }, 1500); // 1.5 second celebration delay
        }
    }

    function isPointInRect(px, py, rx, ry, rw, rh) {
        return px >= rx && px <= rx + rw && py >= ry && py <= ry + rh;
    }

    // --- Fallback: Mouse / Touch Events ---
    function updateMouse(e) {
        inputSource = 'mouse';
        if (e.touches) {
            cursor.x = e.touches[0].clientX;
            cursor.y = e.touches[0].clientY;
        } else {
            cursor.x = e.clientX;
            cursor.y = e.clientY;
        }
    }
    
    window.addEventListener('mousemove', updateMouse);
    window.addEventListener('touchmove', updateMouse, {passive: false});
    
    window.addEventListener('mousedown', (e) => { updateMouse(e); isPinching = true; });
    window.addEventListener('touchstart', (e) => { updateMouse(e); isPinching = true; }, {passive: false});
    
    window.addEventListener('mouseup', () => { isPinching = false; pinchJustReleased = true; });
    window.addEventListener('touchend', () => { isPinching = false; pinchJustReleased = true; });

    // Prevent scrolling when touching canvas
    canvas.addEventListener('touchmove', e => e.preventDefault(), {passive: false});

    // --- DOM Event Listeners ---
    document.getElementById('startBtn').addEventListener('click', () => {
        document.getElementById('startScreen').classList.add('hidden');
        gameState = 'instruct';
        startButtonAR = new ARButton(canvas.width/2 - 140, canvas.height/2 - 45, "เริ่มเกม! (Pinch)");
    });

    // --- MediaPipe Setup ---
    const videoElement = document.getElementById('videoElement');
    
    function onResults(results) {
        handLandmarks = null;
        lastHandTime = performance.now();

        if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
            inputSource = 'hand';
            handLandmarks = results.multiHandLandmarks[0];
            
            // Get Index Tip (8) and Thumb Tip (4)
            const index = handLandmarks[8];
            const thumb = handLandmarks[4];

            // Canvas is NOT mirrored, but Video IS.
            // To make coordinates match the mirrored video, we flip X.
            cursor.x = (1 - index.x) * canvas.width; 
            cursor.y = index.y * canvas.height;

            // Calculate Pythagorean distance using screen pixels
            let thumbX = (1 - thumb.x) * canvas.width;
            let thumbY = thumb.y * canvas.height;
            let dx = cursor.x - thumbX;
            let dy = cursor.y - thumbY;
            
            // Math.sqrt(dx*dx + dy*dy) for accuracy
            let distance = Math.sqrt(dx * dx + dy * dy);
            
            // Threshold for pinch (~5% of screen width)
            let pinchThreshold = canvas.width * 0.05; 
            
            let wasPinching = isPinching;
            isPinching = distance < pinchThreshold;
            
            if (wasPinching && !isPinching) {
                pinchJustReleased = true;
            }
        }
    }

    const hands = new Hands({locateFile: (file) => {
        return `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`;
    }});
    hands.setOptions({
        maxNumHands: 1,
        modelComplexity: 1,
        minDetectionConfidence: 0.7,
        minTrackingConfidence: 0.7
    });
    hands.onResults(onResults);

    const camera = new Camera(videoElement, {
        onFrame: async () => { await hands.send({image: videoElement}); },
        width: 1280,
        height: 720
    });
    camera.start();

    // --- Main Draw & Game Loop (requestAnimationFrame) ---
    function drawLoop() {
        ctx.clearRect(0, 0, canvas.width, canvas.height);

        // UI Hand Warning
        if (inputSource === 'none' && (gameState === 'playing' || gameState === 'instruct') && performance.now() - lastHandTime > 2000) {
            document.getElementById('handWarning').classList.remove('hidden');
        } else {
            document.getElementById('handWarning').classList.add('hidden');
        }

        let hoveredObj = null;

        // --- INSTRUCTION STATE ---
        if (gameState === 'instruct') {
            if (startButtonAR) {
                // Keep button centered if resized
                startButtonAR.x = canvas.width/2 - startButtonAR.width/2;
                startButtonAR.y = canvas.height/2 - startButtonAR.height/2;
                
                startButtonAR.draw(ctx);
                
                if (isPointInRect(cursor.x, cursor.y, startButtonAR.x, startButtonAR.y, startButtonAR.width, startButtonAR.height)) {
                    hoveredObj = startButtonAR;
                    if (isPinching) {
                        gameState = 'playing';
                        document.getElementById('hud').classList.remove('hidden');
                        initLevel();
                    }
                }
            }
        } 
        // --- PLAYING / TRANSITION STATE ---
        else if (gameState === 'playing' || gameState === 'transition') {
            
            // 1. Calculate & Draw Sentence Frame (Glassmorphism on Canvas)
            ctx.font = 'bold 36px Arial';
            let frameY = canvas.height - 180;
            let padding = 40;
            let textY = frameY + 70; // Baseline for text
            
            // Measure total width needed
            let totalWidth = 0;
            sentenceParts.forEach((part, i) => {
                totalWidth += ctx.measureText(part).width;
                if (i < slots.length) totalWidth += slots[i].width + 20; // 20px gap
            });

            let startX = (canvas.width - totalWidth) / 2;
            
            // Draw Glass Frame Background
            ctx.save();
            ctx.fillStyle = 'rgba(15, 23, 42, 0.7)'; // Dark overlay for contrast
            ctx.beginPath();
            ctx.roundRect(Math.max(20, startX - padding), frameY, Math.min(canvas.width-40, totalWidth + padding*2), 120, 20);
            ctx.fill();
            ctx.strokeStyle = 'rgba(255, 255, 255, 0.2)';
            ctx.lineWidth = 2;
            ctx.stroke();
            ctx.restore();

            // 2. Render Sentence and Layout Slots
            let currentX = startX;
            ctx.fillStyle = 'white';
            ctx.textBaseline = 'middle';
            
            sentenceParts.forEach((part, i) => {
                // Draw text part
                ctx.fillText(part, currentX, textY);
                currentX += ctx.measureText(part).width;

                // Position and draw slot if exists
                if (i < slots.length) {
                    currentX += 10; // Left margin
                    slots[i].x = currentX;
     
