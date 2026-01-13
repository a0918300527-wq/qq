<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>星際躲避者 - Star Dodger</title>
    <style>
        :root {
            --bg-color: #0a0a12;
            --text-color: #ffffff;
            --accent-color: #00d4ff;
        }

        body {
            margin: 0;
            padding: 0;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            background-color: var(--bg-color);
            color: var(--text-color);
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            overflow: hidden;
            touch-action: none;
        }

        #game-container {
            position: relative;
            box-shadow: 0 0 50px rgba(0, 212, 255, 0.2);
            border: 2px solid #333;
            border-radius: 8px;
            overflow: hidden;
            background: black;
        }

        canvas {
            display: block;
            cursor: none;
        }

        #ui-layer {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            pointer-events: none;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
        }

        .hud {
            position: absolute;
            top: 20px;
            left: 20px;
            font-size: 24px;
            font-weight: bold;
            text-shadow: 2px 2px 4px rgba(0,0,0,0.5);
            pointer-events: none;
        }

        .menu {
            pointer-events: auto;
            background: rgba(10, 10, 18, 0.85);
            padding: 30px;
            border-radius: 15px;
            text-align: center;
            border: 1px solid var(--accent-color);
            backdrop-filter: blur(5px);
        }

        h1 {
            margin-top: 0;
            color: var(--accent-color);
            letter-spacing: 4px;
        }

        button {
            background: transparent;
            color: var(--accent-color);
            border: 2px solid var(--accent-color);
            padding: 12px 30px;
            font-size: 18px;
            font-weight: bold;
            border-radius: 50px;
            cursor: pointer;
            transition: all 0.3s;
            margin-top: 20px;
        }

        button:hover {
            background: var(--accent-color);
            color: #000;
            box-shadow: 0 0 20px var(--accent-color);
        }

        .instructions {
            margin-top: 15px;
            font-size: 14px;
            color: #aaa;
        }

        #game-over {
            display: none;
        }

        #start-screen {
            display: block;
        }
    </style>
</head>
<body>

    <div id="game-container">
        <canvas id="gameCanvas"></canvas>
        <div class="hud" id="scoreDisplay">分數: 0</div>
        
        <div id="ui-layer">
            <!-- 開始畫面 -->
            <div id="start-screen" class="menu">
                <h1>星際躲避者</h1>
                <p>穿梭在隕石群中生存下來</p>
                <button onclick="startGame()">開始遊戲</button>
                <div class="instructions">使用 滑鼠 或 手指 左右移動操控太空船</div>
            </div>

            <!-- 結束畫面 -->
            <div id="game-over" class="menu">
                <h1 style="color: #ff4d4d;">任務失敗</h1>
                <p id="finalScoreText">最終得分: 0</p>
                <button onclick="startGame()">再次嘗試</button>
            </div>
        </div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreDisplay = document.getElementById('scoreDisplay');
        const startScreen = document.getElementById('start-screen');
        const gameOverScreen = document.getElementById('game-over');
        const finalScoreText = document.getElementById('finalScoreText');

        // 遊戲設定
        let gameActive = false;
        let score = 0;
        let animationId;
        
        const player = {
            x: 0,
            y: 0,
            width: 40,
            height: 40,
            color: '#00d4ff',
            trail: []
        };

        let obstacles = [];
        let particles = [];
        let stars = [];
        let difficulty = 1;

        // 初始化背景星星
        function initStars() {
            stars = [];
            for (let i = 0; i < 100; i++) {
                stars.push({
                    x: Math.random() * canvas.width,
                    y: Math.random() * canvas.height,
                    size: Math.random() * 2,
                    speed: Math.random() * 0.5 + 0.2
                });
            }
        }

        function resize() {
            canvas.width = Math.min(window.innerWidth * 0.9, 600);
            canvas.height = Math.min(window.innerHeight * 0.8, 800);
            player.x = canvas.width / 2;
            player.y = canvas.height - 80;
            initStars();
        }

        window.addEventListener('resize', resize);
        resize();

        // 操作監聽
        const moveHandler = (e) => {
            if (!gameActive) return;
            let clientX;
            if (e.type === 'touchmove') {
                clientX = e.touches[0].clientX;
            } else {
                clientX = e.clientX;
            }
            
            const rect = canvas.getBoundingClientRect();
            const root = document.documentElement;
            const mouseX = clientX - rect.left;
            
            // 平滑移動
            player.x = Math.max(player.width/2, Math.min(canvas.width - player.width/2, mouseX));
        };

        canvas.addEventListener('mousemove', moveHandler);
        canvas.addEventListener('touchmove', (e) => {
            moveHandler(e);
            e.preventDefault();
        }, { passive: false });

        function createObstacle() {
            const size = Math.random() * 30 + 20;
            obstacles.push({
                x: Math.random() * (canvas.width - size),
                y: -size,
                size: size,
                speed: (Math.random() * 2 + 2) * difficulty,
                rotation: 0,
                rotSpeed: Math.random() * 0.1 - 0.05
            });
        }

        function createExplosion(x, y, color) {
            for (let i = 0; i < 20; i++) {
                particles.push({
                    x: x,
                    y: y,
                    vx: (Math.random() - 0.5) * 10,
                    vy: (Math.random() - 0.5) * 10,
                    life: 1.0,
                    color: color
                });
            }
        }

        function startGame() {
            gameActive = true;
            score = 0;
            difficulty = 1;
            obstacles = [];
            particles = [];
            player.trail = [];
            startScreen.style.display = 'none';
            gameOverScreen.style.display = 'none';
            scoreDisplay.textContent = `分數: ${score}`;
            
            cancelAnimationFrame(animationId);
            gameLoop();
        }

        function endGame() {
            gameActive = false;
            gameOverScreen.style.display = 'block';
            finalScoreText.textContent = `最終得分: ${Math.floor(score)}`;
            createExplosion(player.x, player.y, player.color);
        }

        function drawPlayer() {
            // 繪製尾跡
            player.trail.push({x: player.x, y: player.y});
            if (player.trail.length > 10) player.trail.shift();

            ctx.globalAlpha = 0.3;
            player.trail.forEach((t, i) => {
                ctx.fillStyle = player.color;
                const s = (i / player.trail.length) * player.width;
                ctx.beginPath();
                ctx.arc(t.x, t.y + 20, s/2, 0, Math.PI * 2);
                ctx.fill();
            });
            ctx.globalAlpha = 1.0;

            // 繪製太空船 (簡單的多邊形)
            ctx.fillStyle = player.color;
            ctx.beginPath();
            ctx.moveTo(player.x, player.y - 20); // 頂點
            ctx.lineTo(player.x - 20, player.y + 20); // 左底
            ctx.lineTo(player.x, player.y + 10); // 內凹底
            ctx.lineTo(player.x + 20, player.y + 20); // 右底
            ctx.closePath();
            ctx.fill();
            
            // 引擎火光
            ctx.fillStyle = '#ffaa00';
            ctx.beginPath();
            ctx.arc(player.x, player.y + 15, 5 + Math.random() * 5, 0, Math.PI * 2);
            ctx.fill();
        }

        function update() {
            if (!gameActive) return;

            score += 0.1;
            scoreDisplay.textContent = `分數: ${Math.floor(score)}`;
            difficulty = 1 + (score / 500);

            // 更新背景
            stars.forEach(star => {
                star.y += star.speed;
                if (star.y > canvas.height) star.y = 0;
            });

            // 生成障礙物
            if (Math.random() < 0.03 * difficulty) {
                createObstacle();
            }

            // 更新障礙物
            for (let i = obstacles.length - 1; i >= 0; i--) {
                const o = obstacles[i];
                o.y += o.speed;
                o.rotation += o.rotSpeed;

                // 碰撞檢測
                const dx = player.x - (o.x + o.size/2);
                const dy = player.y - (o.y + o.size/2);
                const distance = Math.sqrt(dx*dx + dy*dy);

                if (distance < (o.size/2 + 15)) {
                    endGame();
                }

                if (o.y > canvas.height) {
                    obstacles.splice(i, 1);
                }
            }

            // 更新粒子
            for (let i = particles.length - 1; i >= 0; i--) {
                const p = particles[i];
                p.x += p.vx;
                p.y += p.vy;
                p.life -= 0.02;
                if (p.life <= 0) particles.splice(i, 1);
            }
        }

        function draw() {
            // 清除畫布
            ctx.fillStyle = '#0a0a12';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // 畫星星
            ctx.fillStyle = 'white';
            stars.forEach(star => {
                ctx.beginPath();
                ctx.arc(star.x, star.y, star.size, 0, Math.PI * 2);
                ctx.fill();
            });

            // 畫障礙物
            obstacles.forEach(o => {
                ctx.save();
                ctx.translate(o.x + o.size/2, o.y + o.size/2);
                ctx.rotate(o.rotation);
                
                // 畫一個多邊形岩石
                ctx.fillStyle = '#666';
                ctx.strokeStyle = '#888';
                ctx.lineWidth = 2;
                ctx.beginPath();
                ctx.moveTo(0, -o.size/2);
                ctx.lineTo(o.size/2, -o.size/4);
                ctx.lineTo(o.size/2.2, o.size/2.2);
                ctx.lineTo(0, o.size/2);
                ctx.lineTo(-o.size/2, o.size/3);
                ctx.lineTo(-o.size/2.5, -o.size/2.5);
                ctx.closePath();
                ctx.fill();
                ctx.stroke();
                
                ctx.restore();
            });

            // 畫粒子
            particles.forEach(p => {
                ctx.globalAlpha = p.life;
                ctx.fillStyle = p.color;
                ctx.beginPath();
                ctx.arc(p.x, p.y, 3, 0, Math.PI * 2);
                ctx.fill();
            });
            ctx.globalAlpha = 1.0;

            if (gameActive) {
                drawPlayer();
            }
        }

        function gameLoop() {
            update();
            draw();
            animationId = requestAnimationFrame(gameLoop);
        }

        // 初始繪製
        draw();
    </script>
</body>
</html>
