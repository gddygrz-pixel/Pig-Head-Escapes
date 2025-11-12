[Uploading index.html…]()
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>🐷 小猪下楼大冒险</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            background: linear-gradient(180deg, #1a1a2e 0%, #0f0f1e 100%);
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: flex-start;
            min-height: 100vh;
            font-family: Arial, sans-serif;
            color: #fff;
            overflow: hidden;
            touch-action: none;
            padding: 5px 0;
        }
        h1 {
            font-size: 20px;
            margin: 5px 0;
            text-shadow: 0 0 10px rgba(255, 182, 193, 0.8);
        }
        #gameCanvas {
            border: 3px solid #ffb6c1;
            border-radius: 8px;
            box-shadow: 0 0 30px rgba(255, 182, 193, 0.4);
            background: #000;
            max-width: 100%;
            height: auto;
            touch-action: none;
        }
        .stats {
            background: rgba(0, 0, 0, 0.9);
            padding: 8px 15px;
            border-radius: 8px;
            border: 2px solid #ffb6c1;
            width: 90%;
            max-width: 360px;
            text-align: center;
            margin: 5px 0;
        }
        .stat-row {
            display: flex;
            justify-content: space-around;
            margin: 4px 0;
            font-size: 14px;
            font-weight: bold;
        }
        .tips {
            margin: 5px 0;
            font-size: 12px;
            color: #aaa;
            max-width: 90%;
            text-align: center;
            background: rgba(0, 0, 0, 0.6);
            padding: 8px;
            border-radius: 5px;
        }
        .game-over {
            position: fixed;
            top: 50%;
            left: 50%;
            transform: translate(-50%, -50%);
            background: rgba(0, 0, 0, 0.95);
            padding: 30px 20px;
            border-radius: 16px;
            border: 3px solid #ffb6c1;
            text-align: center;
            display: none;
            z-index: 100;
            width: 90%;
            max-width: 300px;
        }
        .game-over h2 {
            font-size: 24px;
            margin-bottom: 15px;
        }
        .restart-btn {
            margin-top: 15px;
            padding: 12px 30px;
            background: #ffb6c1;
            border: none;
            border-radius: 8px;
            color: #000;
            font-size: 18px;
            font-weight: bold;
            cursor: pointer;
        }
        /* 触摸区域指示 */
        .touch-indicator {
            position: fixed;
            bottom: 10px;
            left: 50%;
            transform: translateX(-50%);
            background: rgba(0, 0, 0, 0.7);
            padding: 8px 15px;
            border-radius: 20px;
            font-size: 12px;
            color: #fff;
            z-index: 50;
            pointer-events: none;
        }
        @media (max-width: 400px) {
            h1 { font-size: 18px; }
            .stat-row { font-size: 12px; }
        }
    </style>
</head>
<body>
    <h1>🐷 小猪下楼大冒险 🐷</h1>
    <canvas id="gameCanvas" width="360" height="560"></canvas>
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
    <div class="tips">
        💡 触摸屏幕左边←向左 | 触摸右边→向右<br>
        ⚠️ 屏幕会持续下降,不断往下掉!
    </div>
    <div class="touch-indicator" id="touchIndicator">👆 触摸屏幕控制移动</div>
    
    <div class="game-over" id="gameOver">
        <h2>💀 游戏结束!</h2>
        <div style="margin: 15px 0; font-size: 16px;">
            <p>下到第 <span id="finalFloor" style="color: #ffd700;">0</span> 层</p>
            <p>用时 <span id="finalTime" style="color: #00ff00;">0.0</span> 秒</p>
        </div>
        <button class="restart-btn" onclick="restartGame()">🔄 重新挑战</button>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const touchIndicator = document.getElementById('touchIndicator');
        
        // 根据屏幕大小调整画布
        function resizeCanvas() {
            const maxWidth = Math.min(window.innerWidth - 20, 360);
            canvas.style.width = maxWidth + 'px';
            canvas.style.height = (maxWidth * 560 / 360) + 'px';
        }
        resizeCanvas();
        window.addEventListener('resize', resizeCanvas);
        
        // 隐藏触摸提示
        setTimeout(() => {
            touchIndicator.style.opacity = '0';
            touchIndicator.style.transition = 'opacity 1s';
        }, 3000);
        
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
            platformSpacing: 120,
            touchActive: false
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
                
                if (this.y > game.camera.y + 400) {
                    game.camera.y = this.y - 400;
                }
                
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
                
                ctx.fillStyle = '#ffb3d9';
                ctx.fillRect(this.x, screenY, this.width, this.height);
                
                ctx.fillStyle = '#000';
                ctx.beginPath();
                ctx.arc(this.x + 8, screenY + 10, 3, 0, Math.PI * 2);
                ctx.arc(this.x + 22, screenY + 10, 3, 0, Math.PI * 2);
                ctx.fill();
                
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
                
                game.camera.y += game.screenFallSpeed;
                
                updatePlatformGeneration();
                game.player.update();
                
                game.platforms.forEach(p => p.draw());
                game.obstacles.forEach(o => o.draw());
                game.player.draw();
                
                // 触摸区域视觉提示
                if (game.touchActive) {
                    ctx.fillStyle = game.keys.left ? 'rgba(255, 0, 0, 0.1)' : 'rgba(0, 255, 0, 0.1)';
                    const halfWidth = canvas.width / 2;
                    if (game.keys.left) {
                        ctx.fillRect(0, 0, halfWidth, canvas.height);
                    } else {
                        ctx.fillRect(halfWidth, 0, halfWidth, canvas.height);
                    }
                }
                
                const gradient = ctx.createLinearGradient(0, 0, 0, 80);
                gradient.addColorStop(0, 'rgba(255, 0, 0, 0.8)');
                gradient.addColorStop(0.6, 'rgba(255, 0, 0, 0.4)');
                gradient.addColorStop(1, 'rgba(255, 0, 0, 0)');
                ctx.fillStyle = gradient;
                ctx.fillRect(0, 0, canvas.width, 80);
                
                ctx.strokeStyle = 'rgba(255, 0, 0, 0.9)';
                ctx.lineWidth = 3;
                ctx.setLineDash([12, 6]);
                ctx.beginPath();
                ctx.moveTo(0, 0);
                ctx.lineTo(canvas.width, 0);
                ctx.stroke();
                ctx.setLineDash([]);
                
                const playerScreenY = game.player.y - game.camera.y;
                if (playerScreenY < 120) {
                    const warningAlpha = 1 - playerScreenY / 120;
                    ctx.fillStyle = `rgba(255, 255, 0, ${warningAlpha})`;
                    ctx.font = 'bold 16px Arial';
                    ctx.textAlign = 'center';
                    
                    if (Math.floor(Date.now() / 300) % 2 === 0) {
                        ctx.fillText('⚠️ 危险! ⚠️', canvas.width / 2, 35);
                    }
                }
                
                updateUI();
            }
            requestAnimationFrame(gameLoop);
        }
        
        // 键盘控制(电脑)
        window.addEventListener('keydown', (e) => {
            if (e.key === 'ArrowLeft' || e.key === 'a') game.keys.left = true;
            if (e.key === 'ArrowRight' || e.key === 'd') game.keys.right = true;
        });
        
        window.addEventListener('keyup', (e) => {
            if (e.key === 'ArrowLeft' || e.key === 'a') game.keys.left = false;
            if (e.key === 'ArrowRight' || e.key === 'd') game.keys.right = false;
        });
        
        // 触摸控制(手机) - 新逻辑
        function handleTouch(e) {
            e.preventDefault();
            
            if (e.touches.length > 0) {
                const touch = e.touches[0];
                const rect = canvas.getBoundingClientRect();
                const touchX = touch.clientX - rect.left;
                const canvasWidth = rect.width;
                const halfWidth = canvasWidth / 2;
                
                game.touchActive = true;
                
                // 触摸左半边 = 向左走
                if (touchX < halfWidth) {
                    game.keys.left = true;
                    game.keys.right = false;
                } 
                // 触摸右半边 = 向右走
                else {
                    game.keys.right = true;
                    game.keys.left = false;
                }
            }
        }
        
        function handleTouchEnd(e) {
            e.preventDefault();
            game.touchActive = false;
            game.keys.left = false;
            game.keys.right = false;
        }
        
        canvas.addEventListener('touchstart', handleTouch, { passive: false });
        canvas.addEventListener('touchmove', handleTouch, { passive: false });
        canvas.addEventListener('touchend', handleTouchEnd, { passive: false });
        canvas.addEventListener('touchcancel', handleTouchEnd, { passive: false });
        
        // 鼠标控制(测试用)
        canvas.addEventListener('mousedown', (e) => {
            const rect = canvas.getBoundingClientRect();
            const mouseX = e.clientX - rect.left;
            const halfWidth = rect.width / 2;
            
            game.touchActive = true;
            
            if (mouseX < halfWidth) {
                game.keys.left = true;
                game.keys.right = false;
            } else {
                game.keys.right = true;
                game.keys.left = false;
            }
        });
        
        canvas.addEventListener('mousemove', (e) => {
            if (game.touchActive) {
                const rect = canvas.getBoundingClientRect();
                const mouseX = e.clientX - rect.left;
                const halfWidth = rect.width / 2;
                
                if (mouseX < halfWidth) {
                    game.keys.left = true;
                    game.keys.right = false;
                } else {
                    game.keys.right = true;
                    game.keys.left = false;
                }
            }
        });
        
        canvas.addEventListener('mouseup', () => {
            game.touchActive = false;
            game.keys.left = false;
            game.keys.right = false;
        });
        
        canvas.addEventListener('mouseleave', () => {
            game.touchActive = false;
            game.keys.left = false;
            game.keys.right = false;
        });
        
        init();
        gameLoop();
    </script>
</body>
</html>
