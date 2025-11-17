<!DOCTYPE html>
<html lang="fa" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Stork Oracle Run</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background-color: #0d1117; /* رنگ پس‌زمینه تیره */
            color: #ffffff;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            overflow: hidden; /* جلوگیری از اسکرول */
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            touch-action: none; /* برای موبایل */
        }

        #game-container {
            position: relative;
            width: 100%;
            max-width: 400px;
            height: 100%;
            max-height: 600px;
            border: 2px solid #00ff9d; /* سبز نئونی Stork */
            background: linear-gradient(180deg, #161b22 0%, #0d1117 100%);
            overflow: hidden;
        }

        canvas {
            display: block;
            width: 100%;
            height: 100%;
        }

        #ui-layer {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            padding: 15px;
            box-sizing: border-box;
            display: flex;
            justify-content: space-between;
            font-weight: bold;
            font-size: 18px;
            z-index: 10;
            text-shadow: 0 0 5px #000;
        }

        #score-board { color: #00ff9d; }
        #health-board { color: #ff4d4d; }

        #start-screen, #game-over-screen {
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
            z-index: 20;
            text-align: center;
        }

        h1 {
            color: #00ff9d;
            font-size: 32px;
            margin-bottom: 10px;
            text-shadow: 0 0 10px #00ff9d;
        }

        p { font-size: 14px; color: #ccc; margin-bottom: 20px; padding: 0 20px; }

        button {
            background: #00ff9d;
            color: #000;
            border: none;
            padding: 15px 30px;
            font-size: 18px;
            font-weight: bold;
            cursor: pointer;
            border-radius: 50px;
            box-shadow: 0 0 15px rgba(0, 255, 157, 0.5);
            transition: transform 0.2s;
        }

        button:active { transform: scale(0.95); }

        .hidden { display: none !important; }
        
        /* کنترل‌های موبایل */
        .controls-hint {
            position: absolute;
            bottom: 10px;
            width: 100%;
            text-align: center;
            font-size: 12px;
            color: rgba(255,255,255,0.3);
            pointer-events: none;
        }
    </style>
</head>
<body>

    <div id="game-container">
        <div id="ui-layer">
            <span id="score-board">Data: 0</span>
            <span id="health-board">HP: 3</span>
        </div>

        <div id="start-screen">
            <h1>🦢 Stork Oracle</h1>
            <p>داده‌های سریع (⚡) را جمع کن و از گس‌های گران (⛽) فرار کن!</p>
            <p>با کامپیوتر: دکمه‌های چپ/راست<br>با موبایل: لمس سمت چپ/راست صفحه</p>
            <button onclick="startGame()">شروع بازی</button>
        </div>

        <div id="game-over-screen" class="hidden">
            <h1 style="color: #ff4d4d;">ارتباط قطع شد!</h1>
            <p>امتیاز نهایی: <span id="final-score">0</span></p>
            <button onclick="startGame()">تلاش مجدد</button>
        </div>

        <canvas id="gameCanvas"></canvas>
        
        <div class="controls-hint">برای حرکت کلیک کنید یا ضربه بزنید</div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const scoreEl = document.getElementById('score-board');
        const healthEl = document.getElementById('health-board');
        const finalScoreEl = document.getElementById('final-score');
        const startScreen = document.getElementById('start-screen');
        const gameOverScreen = document.getElementById('game-over-screen');

        // تنظیم اندازه Canvas
        function resize() {
            canvas.width = canvas.parentElement.clientWidth;
            canvas.height = canvas.parentElement.clientHeight;
        }
        window.addEventListener('resize', resize);
        resize();

        // متغیرهای بازی
        let gameRunning = false;
        let score = 0;
        let health = 3;
        let speedMultiplier = 1;
        let frames = 0;

        // بازیکن (Stork)
        const player = {
            x: 0,
            y: 0,
            width: 50,
            height: 50,
            speed: 7,
            emoji: '🦢' // شکلک لک‌لک
        };

        // آیتم‌ها
        let items = [];
        
        // ورودی‌ها
        let moveLeft = false;
        let moveRight = false;

        // کنترل کیبورد
        window.addEventListener('keydown', (e) => {
            if (e.key === 'ArrowLeft') moveLeft = true;
            if (e.key === 'ArrowRight') moveRight = true;
        });
        window.addEventListener('keyup', (e) => {
            if (e.key === 'ArrowLeft') moveLeft = false;
            if (e.key === 'ArrowRight') moveRight = false;
        });

        // کنترل لمسی (موبایل)
        canvas.addEventListener('touchstart', handleTouch, {passive: false});
        canvas.addEventListener('touchmove', handleTouch, {passive: false});
        canvas.addEventListener('touchend', () => { moveLeft = false; moveRight = false; });

        function handleTouch(e) {
            e.preventDefault();
            const touchX = e.touches[0].clientX;
            const rect = canvas.getBoundingClientRect();
            const relativeX = touchX - rect.left;

            if (relativeX < canvas.width / 2) {
                moveLeft = true;
                moveRight = false;
            } else {
                moveLeft = false;
                moveRight = true;
            }
        }

        // کلاس آیتم (داده یا گس)
        class Item {
            constructor() {
                this.size = 40;
                this.x = Math.random() * (canvas.width - this.size);
                this.y = -this.size;
                this.speed = (Math.random() * 2 + 2) * speedMultiplier;
                // 70% شانس داده خوب، 30% شانس گس بد
                this.isGood = Math.random() > 0.3; 
                this.emoji = this.isGood ? '⚡' : '⛽'; // داده سریع یا گس
            }

            update() {
                this.y += this.speed;
            }

            draw() {
                ctx.font = "30px Arial";
                ctx.textAlign = "center";
                ctx.fillText(this.emoji, this.x + this.size/2, this.y + this.size/2 + 10);
            }
        }

        function initGame() {
            player.x = canvas.width / 2 - player.width / 2;
            player.y = canvas.height - player.height - 20;
            items = [];
            score = 0;
            health = 3;
            speedMultiplier = 1;
            frames = 0;
            scoreEl.innerText = `Data: ${score}`;
            healthEl.innerText = `HP: ${health}`;
        }

        function startGame() {
            initGame();
            gameRunning = true;
            startScreen.classList.add('hidden');
            gameOverScreen.classList.add('hidden');
            requestAnimationFrame(gameLoop);
        }

        function gameOver() {
            gameRunning = false;
            finalScoreEl.innerText = score;
            gameOverScreen.classList.remove('hidden');
        }

        function gameLoop() {
            if (!gameRunning) return;

            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // حرکت بازیکن
            if (moveLeft && player.x > 0) player.x -= player.speed;
            if (moveRight && player.x < canvas.width - player.width) player.x += player.speed;

            // رسم بازیکن
            ctx.font = "40px Arial";
            ctx.textAlign = "center";
            ctx.fillText(player.emoji, player.x + player.width/2, player.y + player.height/2 + 15);

            // مدیریت آیتم‌ها
            if (frames % 60 === 0) { // هر ثانیه (تقریبا) یک آیتم جدید
                items.push(new Item());
                // افزایش سختی بازی به مرور
                if (frames % 600 === 0) speedMultiplier += 0.1; 
            }

            for (let i = 0; i < items.length; i++) {
                let item = items[i];
                item.update();
                item.draw();

                // تشخیص برخورد
                if (
                    player.x < item.x + item.size &&
                    player.x + player.width > item.x &&
                    player.y < item.y + item.size &&
                    player.y + player.height > item.y
                ) {
                    // برخورد رخ داد
                    if (item.isGood) {
                        score += 10;
                        scoreEl.innerText = `Data: ${score}`;
                    } else {
                        health -= 1;
                        healthEl.innerText = `HP: ${health}`;
                        // افکت لرزش صفحه (ساده)
                        canvas.style.transform = "translate(5px, 0)";
                        setTimeout(() => canvas.style.transform = "none", 100);
                    }
                    items.splice(i, 1);
                    i--;
                    
                    if (health <= 0) {
                        gameOver();
                        return;
                    }
                }
                // حذف آیتم‌های خارج شده از صفحه
                else if (item.y > canvas.height) {
                    items.splice(i, 1);
                    i--;
                }
            }

            frames++;
            requestAnimationFrame(gameLoop);
        }
    </script>
</body>
</html>
