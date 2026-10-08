<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Lagos Danfo Rush: The Ultimate Hustle</title>
    <style>
        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
        }

        body {
            background-color: #121212;
            color: #ffffff;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            overflow: hidden;
        }

        #game-container {
            position: relative;
            width: 400px;
            height: 650px;
            background: #2b2b2b;
            border: 4px solid #f1c40f;
            border-radius: 12px;
            overflow: hidden;
            box-shadow: 0 10px 30px rgba(0,0,0,0.8);
        }

        canvas {
            display: block;
            background: #333;
        }

        /* UI Overlays */
        .screen {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background: rgba(0, 0, 0, 0.85);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            text-align: center;
            padding: 20px;
            z-index: 10;
        }

        .screen.hidden {
            display: none;
        }

        h1 {
            color: #f1c40f;
            font-size: 28px;
            margin-bottom: 10px;
            text-transform: uppercase;
            text-shadow: 2px 2px #e74c3c;
        }

        p {
            font-size: 14px;
            margin-bottom: 20px;
            color: #ecf0f1;
            line-height: 1.5;
        }

        .btn {
            background-color: #f1c40f;
            color: #2c3e50;
            border: none;
            padding: 12px 25px;
            font-size: 16px;
            font-weight: bold;
            border-radius: 25px;
            cursor: pointer;
            transition: 0.2s;
            box-shadow: 0 4px 10px rgba(241, 196, 15, 0.4);
            text-transform: uppercase;
        }

        .btn:hover {
            background-color: #f39c12;
            transform: scale(1.05);
        }

        /* HUD */
        #hud {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            display: flex;
            justify-content: space-between;
            padding: 10px 15px;
            background: rgba(0, 0, 0, 0.6);
            font-size: 14px;
            font-weight: bold;
            z-index: 5;
            border-bottom: 2px solid #f1c40f;
        }

        .hud-item {
            color: #f1c40f;
        }

        /* Controls hint for mobile */
        .controls-hint {
            position: absolute;
            bottom: 15px;
            width: 100%;
            text-align: center;
            font-size: 12px;
            color: #bdc3c7;
            pointer-events: none;
            z-index: 5;
            text-shadow: 1px 1px #000;
        }
    </style>
</head>
<body>

    <div id="game-container">
        <!-- HUD -->
        <div id="hud">
            <div class="hud-item">FARE: ₦<span id="score">0</span></div>
            <div class="hud-item">PASSENGERS: <span id="passengers">0</span>/4</div>
            <div class="hud-item">FUEL: <span id="fuel">100</span>%</div>
        </div>

        <!-- Canvas Game Area -->
        <canvas id="gameCanvas" width="400" height="650"></canvas>

        <div class="controls-hint">Use ◄ ► Arrow Keys or Tap Left/Right to Switch Lanes</div>

        <!-- Start Screen -->
        <div id="start-screen" class="screen">
            <h1>Lagos Danfo Rush</h1>
            <p>Navigate the chaotic Lagos traffic! Dodge reckless okadas, avoid LASTMA, pick up passengers, and collect transport fares before your engine knocks!</p>
            <button class="btn" id="start-btn">Start Hustle</button>
        </div>

        <!-- Game Over Screen -->
        <div id="game-over-screen" class="screen hidden">
            <h1>Engine Knock! 🤦‍♂️</h1>
            <p id="game-over-reason">LASTMA caught you or you ran out of fuel!</p>
            <p style="font-size: 18px; color: #f1c40f; font-weight: bold;">Total Fare Collected: ₦<span id="final-score">0</span></p>
            <button class="btn" id="restart-btn">Try Again</button>
        </div>
    </div>

    <script>
        const canvas = document.getElementById("gameCanvas");
        const ctx = canvas.getContext("2d");

        // UI Elements
        const startScreen = document.getElementById("start-screen");
        const gameOverScreen = document.getElementById("game-over-screen");
        const startBtn = document.getElementById("start-btn");
        const restartBtn = document.getElementById("restart-btn");
        const scoreEl = document.getElementById("score");
        const passengersEl = document.getElementById("passengers");
        const fuelEl = document.getElementById("fuel");
        const finalScoreEl = document.getElementById("final-score");
        const gameOverReason = document.getElementById("game-over-reason");

        // Game Variables
        let gameRunning = false;
        let score = 0;
        let passengers = 0;
        let fuel = 100;
        let gameSpeed = 5;
        let roadOffset = 0;

        // Lanes X positions (3 lanes)
        const lanes = [85, 200, 315];
        let currentLane = 1; // Start in middle lane

        // Player (Danfo Bus)
        const player = {
            width: 50,
            height: 90,
            y: 520,
            targetX: lanes[1],
            x: lanes[1]
        };

        let obstacles = [];
        let items = []; // Passengers or Fuel
        let particles = [];

        // Input Handling
        document.addEventListener("keydown", (e) => {
            if (!gameRunning) return;
            if (e.key === "ArrowLeft" || e.key === "a") {
                if (currentLane > 0) {
                    currentLane--;
                }
            } else if (e.key === "ArrowRight" || e.key === "d") {
                if (currentLane < lanes.length - 1) {
                    currentLane++;
                }
            }
            player.targetX = lanes[currentLane];
        });

        // Touch support for mobile
        canvas.addEventListener("click", (e) => {
            if (!gameRunning) return;
            const rect = canvas.getBoundingClientRect();
            const clickX = e.clientX - rect.left;
            if (clickX < canvas.width / 2) {
                if (currentLane > 0) currentLane--;
            } else {
                if (currentLane < lanes.length - 1) currentLane++;
            }
            player.targetX = lanes[currentLane];
        });

        function startGame() {
            score = 0;
            passengers = 0;
            fuel = 100;
            gameSpeed = 5;
            currentLane = 1;
            player.x = lanes[currentLane];
            player.targetX = lanes[currentLane];
            obstacles = [];
            items = [];
            particles = [];
            
            startScreen.classList.add("hidden");
            gameOverScreen.classList.add("hidden");
            gameRunning = true;
            lastSpawnTime = Date.now();
            requestAnimationFrame(updateGame);
        }

        function endGame(reason) {
            gameRunning = false;
            finalScoreEl.innerText = score;
            gameOverReason.innerText = reason;
            gameOverScreen.classList.remove("hidden");
        }

        let lastSpawnTime = 0;
        function spawnEntities() {
            const now = Date.now();
            if (now - lastSpawnTime > 1300) {
                lastSpawnTime = now;
                const lane = Math.floor(Math.random() * lanes.length);
                const typeRand = Math.random();

                if (typeRand < 0.5) {
                    // Obstacle: Okada or LASTMA car or Pothole
                    const obsType = Math.random() < 0.5 ? 'okada' : (Math.random() < 0.5 ? 'lastma' : 'pothole');
                    obstacles.push({
                        lane: lane,
                        x: lanes[lane],
                        y: -100,
                        type: obsType,
                        width: obsType === 'okada' ? 30 : 50,
                        height: obsType === 'okada' ? 60 : 70
                    });
                } else {
                    // Item: Passenger or Fuel
                    const itemType = Math.random() < 0.7 ? 'passenger' : 'fuel';
                    items.push({
                        lane: lane,
                        x: lanes[lane],
                        y: -100,
                        type: itemType,
                        width: 35,
                        height: 35
                    });
                }
            }
        }

        function updateGame() {
            if (!gameRunning) return;

            // Smooth lane shifting
            player.x += (player.targetX - player.x) * 0.2;

            // Decrease fuel slowly
            fuel -= 0.03;
            if (fuel <= 0) {
                fuel = 0;
                endGame("Your Danfo ran out of fuel! No gas to enter traffic.");
                return;
            }

            // Increase speed gradually
            gameSpeed += 0.001;

            // Move road background
            roadOffset = (roadOffset + gameSpeed) % 40;

            // Spawn objects
            spawnEntities();

            // Update Obstacles
            for (let i = obstacles.length - 1; i >= 0; i--) {
                let obs = obstacles[i];
                obs.y += gameSpeed;

                // Collision detection with player
                if (
                    Math.abs(player.x - obs.x) < 35 &&
                    Math.abs(player.y - obs.y) < 60
                ) {
                    if (obs.type === 'pothole') {
                        fuel -= 15;
                        obstacles.splice(i, 1);
                    } else {
                        endGame(obs.type === 'lastma' ? "LASTMA impounded your Danfo!" : "You crashed into a reckless Okada!");
                        return;
                    }
                }

                // Remove off-screen obstacles
                if (obs.y > canvas.height + 100) {
                    obstacles.splice(i, 1);
                    score += 50; // Survival bonus
                }
            }

            // Update Items
            for (let i = items.length - 1; i >= 0; i--) {
                let item = items[i];
                item.y += gameSpeed;

                // Pickup detection
                if (
                    Math.abs(player.x - item.x) < 40 &&
                    Math.abs(player.y - item.y) < 60
                ) {
                    if (item.type === 'passenger') {
                        passengers++;
                        score += 500; // Fare collected
                        if (passengers >= 4) {
                            passengers = 0;
                            score += 1000; // Drop off bonus!
                        }
                    } else if (item.type === 'fuel') {
                        fuel = Math.min(100, fuel + 30);
                    }
                    items.splice(i, 1);
                }

                // Remove off-screen items
                if (item.y > canvas.height + 100) {
                    items.splice(i, 1);
                }
            }

            // Update HUD
            scoreEl.innerText = score;
            passengersEl.innerText = passengers;
            fuelEl.innerText = Math.floor(fuel);

            // Render
            draw();

            requestAnimationFrame(updateGame);
        }

        function draw() {
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // Draw Road Background
            ctx.fillStyle = "#222";
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // Road shoulders
            ctx.fillStyle = "#27ae60";
            ctx.fillRect(0, 0, 20, canvas.height);
            ctx.fillRect(canvas.width - 20, 0, 20, canvas.height);

            // Lane markers (dashed lines)
            ctx.strokeStyle = "#f1c40f";
            ctx.lineWidth = 4;
            ctx.setLineDash([20, 20]);
            ctx.lineDashOffset = -roadOffset;

            ctx.beginPath();
            ctx.moveTo(133, 0);
            ctx.lineTo(133, canvas.height);
            ctx.stroke();

            ctx.beginPath();
            ctx.moveTo(266, 0);
            ctx.lineTo(266, canvas.height);
            ctx.stroke();

            ctx.setLineDash([]); // Reset line dash

            // Draw Items (Passengers & Fuel)
            items.forEach(item => {
                ctx.save();
                ctx.translate(item.x, item.y);
                if (item.type === 'passenger') {
                    ctx.fillStyle = "#3498db";
                    ctx.beginPath();
                    ctx.arc(0, -5, 10, 0, Math.PI * 2); // Head
                    ctx.fill();
                    ctx.fillRect(-10, 5, 20, 20); // Body
                } else {
                    ctx.fillStyle = "#e74c3c";
                    ctx.fillRect(-12, -15, 24, 30);
                    ctx.fillStyle = "#fff";
                    ctx.font = "12px sans-serif";
                    ctx.fillText("⛽", -8, 4);
                }
                ctx.restore();
            });

            // Draw Obstacles
            obstacles.forEach(obs => {
                ctx.save();
                ctx.translate(obs.x, obs.y);
                if (obs.type === 'okada') {
                    // Okada bike
                    ctx.fillStyle = "#e67e22";
                    ctx.fillRect(-6, -25, 12, 50);
                    ctx.fillStyle = "#fff";
                    ctx.fillRect(-12, -15, 24, 10); // Rider
                } else if (obs.type === 'lastma') {
                    // LASTMA blue/yellow patrol car
                    ctx.fillStyle = "#3498db";
                    ctx.fillRect(-25, -35, 50, 70);
                    ctx.fillStyle = "#f1c40f";
                    ctx.fillRect(-25, -10, 50, 20);
                } else {
                    // Pothole
                    ctx.fillStyle = "#111";
                    ctx.beginPath();
                    ctx.ellipse(0, 0, 25, 15, 0, 0, Math.PI * 2);
                    ctx.fill();
                    ctx.strokeStyle = "#444";
                    ctx.lineWidth = 3;
                    ctx.stroke();
                }
                ctx.restore();
            });

            // Draw Player Danfo Bus (Yellow with black stripes)
            ctx.save();
            ctx.translate(player.x, player.y);
            
            // Bus body
            ctx.fillStyle = "#f1c40f";
            ctx.fillRect(-player.width / 2, -player.height / 2, player.width, player.height);

            // Black stripe / Windows
            ctx.fillStyle = "#2c3e50";
            ctx.fillRect(-player.width / 2, -player.height / 2 + 15, player.width, 25); // Windshield & windows
            ctx.fillRect(-player.width / 2 + 5, player.height / 2 - 20, player.width - 10, 10); // Headlights area

            // "ECOBANK" or "LAGOS" creative text vibe on roof
            ctx.fillStyle = "#000";
            ctx.font = "bold 9px sans-serif";
            ctx.fillText("DANFO", -18, -10);

            ctx.restore();
        }

        startBtn.addEventListener("click", startGame);
        restartBtn.addEventListener("click", startGame);
    </script>
</body>
</html>
