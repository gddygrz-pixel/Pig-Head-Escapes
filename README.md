[Uploading index.html…]()
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>🐷 小猪下楼大冒险</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            background: linear-gradient(180deg, #1a1a2e 0%, #0f0f1e 100%);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            font-family: Arial, sans-serif;
            color: #fff;
            overflow: hidden;
            touch-action: none;
        }
        h1 {
            font-size: 24px;
            margin-bottom: 10px;
            text-shadow: 0 0 10px rgba(255, 182, 193, 0.8);
            animation: pulse 2s infinite;
        }
        @keyframes pulse {
            0%, 100% { transform: scale(1); }
            50% { transform: scale(1.05); }
        }
        #gameCanvas {
            border: 3px solid #ffb6c1;
            border-radius: 8px;
            box-shadow: 0 0 30px rgba(255, 182, 193, 0.4);
            background: #000;
        }
        .stats {
            background: rgba(0, 0, 0, 0.8);
            padding: 15px 20px;
            border-radius: 8px;
            border: 2px solid #ffb6c1;
            min-width: 280px;
            text-align: center;
            margin: 10px 0;
        }
        .stat-row {
            display: flex;
            justify-content: space-around;
            margin: 8px 0;
            font-size: 18px;
            font-weight: bold;
        }
        .controls {
            display: flex;
            gap: 20px;
        }
        .btn {
            width: 100px;
            height: 100px;
            border: 3px solid #ffb6c1;
            border-radius: 50%;
            background: rgba(255, 182, 193, 0.2);
            color: #fff;
            font-size: 32px;
            cursor: pointer;
            user-select: none;
            display: flex;
            align-items: center;
            justify-content: center;
            transition: all 0.1s;
        }
        .btn:active {
            background: rgba(255, 182, 193, 0.5);
            transform: scale(0.95);
        }
        .game-over {
            position: absolute;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(0, 0, 0, 0.95);
            padding: 40px;
            border-radius: 16px;
            border: 3px solid #ffb6c1;
            text-align: center;
            display: none;
            z-index: 100;
        }
        .restart-btn {
            margin-top: 20px;
            padding: 15px 40px;
            background: #ffb6c1;
            border: none;
            border-radius: 8px;
            color: #000;
            font-size: 20px;
            font-weight: bold;
            cursor: pointer;
        }
        .tips {
            margin-top: 10px;
            font-size: 12px;
            color: #aaa;
            max-width: 300px;
            text-align: center;
        }
    </style>
</head>
<body>
    <div id="gameContainer">
        <h1>🐷 小猪下楼大冒险 🐷</h1>
        <canvas id="gameCanvas" width="360" height="640"></canvas>
        <div class="stats">
            <div class="stat-row">
                <span>楼层: <span id="floor">0</span></span>
                <span>时间: <span id="time">0.0s</span></span>
            </div>
            <div class="stat-row">
                <span>速度: <span id="speed">1.0x</span></span>
                <span>下降: <span id="fallSpeed">0.3</span>/帧</span>
            </div>
        </div>
        <div class="controls">
            <div class="btn" id="leftBtn">←</div>
            <div class="btn" id="rightBtn">→</div>
        </div>
        <div class="tips">
            💡 屏幕会持续向下移动,必须不断下落!<br>
            ⚠️ 停在平台上太久会被吞噬!
        </div>
        <div class="game-over" id="gameOver">
            <h2>💀 游戏结束!</h2>
            <div style="margin: 20px 0; font-size: 18px;">
                <p>下到第 <span id="finalFloor" style="color: #ffd700;">0</span> 层</p>
                <p>用时 <span id="finalTime" style="color: #00ff00;">0.0</span> 秒</p>
            </div>
            <button class="restart-btn" onclick="restartGame()">🔄 重新挑战</button>
        </div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        
        const game = {
            player: null,
            platforms: [],
            obstacles: [],
            camera: { y: 0 },
            floor: 0,
            maxFloor: 0,
            speed: 1.0,
            startTime: 0,
            isGameOver: false,
            keys: { left: false, right: false },
            screenFallSpeed: 0.3,
            lastGeneratedFloor: 10,
            platformSpacing: 120
        };
        
        class Player {
            constructor() {
                this.x = canvas.width / 2;
                this.y = 100;
                this.width = 30;
                this.height = 30;
                this.vx = 0;
                this.vy = 0;
                this.speed = 5;
                this.gravity = 0.5;
            }
            
            update() {
                if (game.keys.left) this.vx = -this.speed;
                else if (game.keys.right) this.vx = this.speed;
                else this.vx *= 0.85;
                
                this.vy += this.gravity;
                this.x += this.vx;
                this.y += this.vy;
                
                if (this.x + this.width < 0) this.x = canvas.width;
                else if (this.x > canvas.width) this.x = -this.width;
                
                this.checkCollisions();
                
                // 摄像机只向下移动
                if (this.y > game.camera.y + 400) {
                    game.camera.y = this.y - 400;
                }
                
                // 检查是否被屏幕上方吞噬
                const playerScreenY = this.y - game.camera.y;
                if (playerScreenY < 0) {
                    this.gameOver();
                }
                
                const currentFloor = Math.floor(this.y / game.platformSpacing);
                if (currentFloor > game.floor) {
                    game.floor = currentFloor;
                    if (game.floor > game.maxFloor) game.maxFloor = game.floor;
                    game.speed = 1.0 + game.floor * 0.05;
                    this.gravity = 0.5 * game.speed;
                    game.screenFallSpeed = 0.3 + Math.sqrt(game.floor / 100) * 0.5;
                }
            }
            
            checkCollisions() {
                game.platforms.forEach(platform => {
                    if (this.vy > 0) {
                        const playerBottom = this.y + this.height;
                        const playerLastBottom = playerBottom - this.vy;
                        
                        if (playerLastBottom <= platform.y && 
                            playerBottom >= platform.y &&
                            this.x + this.width > platform.x &&
                            this.x < platform.x + platform.width) {
                            
                            const inGap = this.x + this.width/2 > platform.gapStart && 
                                         this.x + this.width/2 < platform.gapEnd;
                            
                            if (!inGap) {
                                this.y = platform.y - this.height;
                                this.vy = 0;
                            }
                        }
                    }
                });
                
                game.obstacles.forEach(obstacle => {
                    if (this.x < obstacle.x + obstacle.width &&
                        this.x + this.width > obstacle.x &&
                        this.y < obstacle.y + obstacle.height &&
                        this.y + this.height > obstacle.y) {
                        this.gameOver();
                    }
                });
            }
            
            gameOver() {
                if (!game.isGameOver) {
                    game.isGameOver = true;
                    showGameOver();
                }
            }
            
            draw() {
                const screenY = this.y - game.camera.y;
                
                // 粉色小猪
                ctx.fillStyle = '#ffb3d9';
                ctx.fillRect(this.x, screenY, this.width, this.height);
                
                // 眼睛
                ctx.fillStyle = '#000';
                ctx.beginPath();
                ctx.arc(this.x + 8, screenY + 10, 3, 0, Math.PI * 2);
                ctx.arc(this.x + 22, screenY + 10, 3, 0, Math.PI * 2);
                ctx.fill();
                
                // 猪鼻子
                ctx.fillStyle = '#ff85a1';
                ctx.fillRect(this.x + 10, screenY + 18, 10, 6);
                ctx.fillStyle = '#000';
                ctx.fillRect(this.x + 12, screenY + 20, 2, 2);
                ctx.fillRect(this.x + 16, screenY + 20, 2, 2);
            }
        }
        
        class Platform {
            constructor(y, floorNumber) {
                this.y = y;
                this.x = 0;
                this.width = canvas.width;
                this.height = 15;
                this.floorNumber = floorNumber;
                
                const gapSize = 60 + Math.random() * 40;
                this.gapStart = Math.random() * (canvas.width - gapSize);
                this.gapEnd = this.gapStart + gapSize;
                
                const hue = (floorNumber * 15) % 360;
                this.color = `hsl(${hue}, 70%, 50%)`;
            }
            
            draw() {
                const screenY = this.y - game.camera.y;
                if (screenY < -50 || screenY > canvas.height + 50) return;
                
                ctx.fillStyle = this.color;
                if (this.gapStart > 0) {
                    ctx.fillRect(this.x, screenY, this.gapStart, this.height);
                }
                if (this.gapEnd < this.width) {
                    ctx.fillRect(this.gapEnd, screenY, this.width - this.gapEnd, this.height);
                }
                
                if (this.floorNumber > 0 && this.gapStart > 50) {
                    ctx.fillStyle = '#fff';
                    ctx.font = 'bold 14px Arial';
                    ctx.fillText(`${this.floorNumber}F`, 10, screenY + 12);
                }
            }
        }
        
        class Obstacle {
            constructor(x, y, type) {
                this.x = x;
                this.y = y;
                this.type = type;
                this.width = type === 'spike' ? 30 : 40;
                this.height = type === 'spike' ? 20 : 40;
            }
            
            draw() {
                const screenY = this.y - game.camera.y;
                if (screenY < -50 || screenY > canvas.height + 50) return;
                
                if (this.type === 'spike') {
                    ctx.fillStyle = '#ffd43b';
                    ctx.beginPath();
                    ctx.moveTo(this.x, screenY + this.height);
                    ctx.lineTo(this.x + this.width/2, screenY);
                    ctx.lineTo(this.x + this.width, screenY + this.height);
                    ctx.closePath();
                    ctx.fill();
                } else {
                    ctx.fillStyle = '#e64980';
                    ctx.fillRect(this.x, screenY, this.width, this.height);
                }
            }
        }
        
        function generatePlatform(floorNumber) {
            const y = floorNumber * game.platformSpacing;
            game.platforms.push(new Platform(y, floorNumber));
            
            if (floorNumber >= 10 && Math.random() < 0.15) {
                const platform = game.platforms[game.platforms.length - 1];
                const obstacleType = Math.random() < 0.6 ? 'spike' : 'block';
                const obstacleWidth = obstacleType === 'spike' ? 30 : 40;
                
                let obstacleX;
                if (Math.random() < 0.5 && platform.gapStart > obstacleWidth + 20) {
                    obstacleX = Math.random() * (platform.gapStart - obstacleWidth - 10) + 10;
                } else if (platform.gapEnd < canvas.width - obstacleWidth - 20) {
                    obstacleX = platform.gapEnd + 10 + Math.random() * (canvas.width - platform.gapEnd - obstacleWidth - 20);
                } else {
                    return;
                }
                
                const obstacleY = y - (obstacleType === 'spike' ? 20 : 40);
                game.obstacles.push(new Obstacle(obstacleX, obstacleY, obstacleType));
            }
        }
        
        function updatePlatformGeneration() {
            const playerFloor = Math.floor(game.player.y / game.platformSpacing);
            while (game.lastGeneratedFloor < playerFloor + 10) {
                game.lastGeneratedFloor++;
                generatePlatform(game.lastGeneratedFloor);
            }
            
            game.platforms = game.platforms.filter(p => 
                Math.abs(p.floorNumber - playerFloor) < 20
            );
            game.obstacles = game.obstacles.filter(o => {
                const obstacleFloor = Math.floor(o.y / game.platformSpacing);
                return Math.abs(obstacleFloor - playerFloor) < 20;
            });
        }
        
        function init() {
            game.player = new Player();
            game.camera.y = 0;
            game.floor = 0;
            game.maxFloor = 0;
            game.speed = 1.0;
            game.startTime = Date.now();
            game.isGameOver = false;
            game.screenFallSpeed = 0.3;
            game.lastGeneratedFloor = 10;
            game.platforms = [];
            game.obstacles = [];
            
            for (let i = 0; i <= 10; i++) {
                generatePlatform(i);
            }
        }
        
        function updateUI() {
            document.getElementById('floor').textContent = game.maxFloor;
            document.getElementById('speed').textContent = game.speed.toFixed(1) + 'x';
            document.getElementById('fallSpeed').textContent = game.screenFallSpeed.toFixed(2);
            const elapsed = (Date.now() - game.startTime) / 1000;
            document.getElementById('time').textContent = elapsed.toFixed(1) + 's';
        }
        
        function showGameOver() {
            const gameTime = (Date.now() - game.startTime) / 1000;
            document.getElementById('finalFloor').textContent = game.maxFloor;
            document.getElementById('finalTime').textContent = gameTime.toFixed(1);
            document.getElementById('gameOver').style.display = 'block';
        }
        
        function restartGame() {
            document.getElementById('gameOver').style.display = 'none';
            init();
        }
        
        function gameLoop() {
            if (!game.isGameOver) {
                ctx.fillStyle = '#000';
                ctx.fillRect(0, 0, canvas.width, canvas.height);
                
                // 摄像机持续向下移动
                game.camera.y += game.screenFallSpeed;
                
                updatePlatformGeneration();
                game.player.update();
                
                game.platforms.forEach(p => p.draw());
                game.obstacles.forEach(o => o.draw());
                game.player.draw();
                
                // 危险区域
                const gradient = ctx.createLinearGradient(0, 0, 0, 100);
                gradient.addColorStop(0, 'rgba(255, 0, 0, 0.8)');
                gradient.addColorStop(0.6, 'rgba(255, 0, 0, 0.4)');
                gradient.addColorStop(1, 'rgba(255, 0, 0, 0)');
                ctx.fillStyle = gradient;
                ctx.fillRect(0, 0, canvas.width, 100);
                
                ctx.strokeStyle = 'rgba(255, 0, 0, 0.9)';
                ctx.lineWidth = 4;
                ctx.setLineDash([15, 8]);
                ctx.beginPath();
                ctx.moveTo(0, 0);
                ctx.lineTo(canvas.width, 0);
                ctx.stroke();
                ctx.setLineDash([]);
                
                // 警告
                const playerScreenY = game.player.y - game.camera.y;
                if (playerScreenY < 150) {
                    const warningAlpha = 1 - playerScreenY / 150;
                    ctx.fillStyle = `rgba(255, 255, 0, ${warningAlpha})`;
                    ctx.font = 'bold 20px Arial';
                    ctx.textAlign = 'center';
                    
                    if (Math.floor(Date.now() / 300) % 2 === 0) {
                        ctx.fillText('⚠️ 危险! 快下落! ⚠️', canvas.width / 2, 40);
                    }
                }
                
                updateUI();
            }
            requestAnimationFrame(gameLoop);
        }
        
        window.addEventListener('keydown', (e) => {
            if (e.key === 'ArrowLeft' || e.key === 'a') game.keys.left = true;
            if (e.key === 'ArrowRight' || e.key === 'd') game.keys.right = true;
        });
        
        window.addEventListener('keyup', (e) => {
            if (e.key === 'ArrowLeft' || e.key === 'a') game.keys.left = false;
            if (e.key === 'ArrowRight' || e.key === 'd') game.keys.right = false;
        });
        
        const leftBtn = document.getElementById('leftBtn');
        const rightBtn = document.getElementById('rightBtn');
        
        leftBtn.addEventListener('touchstart', (e) => { e.preventDefault(); game.keys.left = true; });
        leftBtn.addEventListener('touchend', (e) => { e.preventDefault(); game.keys.left = false; });
        leftBtn.addEventListener('mousedown', () => game.keys.left = true);
        leftBtn.addEventListener('mouseup', () => game.keys.left = false);
        
        rightBtn.addEventListener('touchstart', (e) => { e.preventDefault(); game.keys.right = true; });
        rightBtn.addEventListener('touchend', (e) => { e.preventDefault(); game.keys.right = false; });
        rightBtn.addEventListener('mousedown', () => game.keys.right = true);
        rightBtn.addEventListener('mouseup', () => game.keys.right = false);
        
        init();
        gameLoop();
    </script>
</body>
</html>
