<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Ping-Pong против ИИ</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        body {
            background: #0a0a1a;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            font-family: 'Courier New', monospace;
        }
        .game-wrapper {
            background: #141430;
            padding: 30px;
            border-radius: 30px;
            box-shadow: 0 0 60px rgba(0, 200, 255, 0.15);
            text-align: center;
        }
        canvas {
            background: #0d0d22;
            border: 2px solid #00ccff;
            border-radius: 16px;
            display: block;
            box-shadow: 0 0 40px rgba(0, 200, 255, 0.2);
            cursor: none;
        }
        .info-panel {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-top: 18px;
            color: #aab;
            font-size: 22px;
            font-weight: bold;
        }
        .score {
            color: #00ccff;
            text-shadow: 0 0 12px rgba(0, 204, 255, 0.5);
        }
        .score span {
            font-size: 28px;
            padding: 0 10px;
        }
        .controls {
            display: flex;
            gap: 15px;
            align-items: center;
        }
        button {
            background: #00ccff;
            color: #0a0a1a;
            border: none;
            padding: 10px 28px;
            border-radius: 40px;
            font-size: 18px;
            font-weight: bold;
            cursor: pointer;
            transition: 0.25s;
            letter-spacing: 1px;
            box-shadow: 0 0 20px rgba(0, 204, 255, 0.3);
        }
        button:hover {
            background: #33ddff;
            transform: scale(1.05);
            box-shadow: 0 0 40px #00ccff;
        }
        .hint {
            color: #556;
            font-size: 14px;
            margin-top: 10px;
        }
        .hint kbd {
            background: #1a1a3e;
            padding: 4px 12px;
            border-radius: 8px;
            color: #88ddff;
            font-weight: bold;
        }
    </style>
</head>
<body>
<div class="game-wrapper">
    <canvas id="gameCanvas" width="800" height="450"></canvas>
    <div class="info-panel">
        <div class="score">👤 <span id="playerScore">0</span></div>
        <div class="score">🤖 <span id="aiScore">0</span></div>
        <div class="controls">
            <button id="restartBtn">⟳ Рестарт</button>
        </div>
    </div>
    <div class="hint">
        <kbd>↑</kbd> <kbd>↓</kbd> — движение  &nbsp;|&nbsp; <kbd>Пробел</kbd> — пауза
    </div>
</div>

<script>
    (function() {
        // ---------- НАСТРОЙКИ ----------
        const W = 800, H = 450;
        const PADDLE_W = 12, PADDLE_H = 80;
        const BALL_SIZE = 12;
        const PLAYER_SPEED = 6;
        const AI_SPEED = 4.2;       // ИИ чуть медленнее игрока — честно
        const BALL_SPEED_INIT = 5;
        const BALL_MAX_SPEED = 9;

        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const playerScoreSpan = document.getElementById('playerScore');
        const aiScoreSpan = document.getElementById('aiScore');

        // ---------- СОСТОЯНИЕ ----------
        let player = { x: 20, y: H/2 - PADDLE_H/2, w: PADDLE_W, h: PADDLE_H };
        let ai = { x: W - 20 - PADDLE_W, y: H/2 - PADDLE_H/2, w: PADDLE_W, h: PADDLE_H };

        let ball = {
            x: W/2, y: H/2,
            size: BALL_SIZE,
            vx: BALL_SPEED_INIT * (Math.random() > 0.5 ? 1 : -1),
            vy: (Math.random() * 2 - 1) * 2.5
        };

        let playerScore = 0;
        let aiScore = 0;
        let paused = false;


let gameOver = false;
        let animationId = null;

        // Клавиши
        let keys = {
            ArrowUp: false,
            ArrowDown: false
        };

        // ---------- СБРОС МЯЧА ----------
        function resetBall(direction) {
            ball.x = W/2;
            ball.y = H/2 + (Math.random() * 60 - 30);
            ball.vx = BALL_SPEED_INIT * direction;
            ball.vy = (Math.random() * 2 - 1) * 2.5;
            // Не даём улететь слишком полого
            if (Math.abs(ball.vy) < 0.8) ball.vy = 0.8 * (ball.vy > 0 ? 1 : -1);
        }

        // ---------- СБРОС ИГРЫ ----------
        function resetGame() {
            player.y = H/2 - PADDLE_H/2;
            ai.y = H/2 - PADDLE_H/2;
            playerScore = 0;
            aiScore = 0;
            gameOver = false;
            paused = false;
            updateScore();
            resetBall(1);
        }

        // ---------- ОБНОВЛЕНИЕ СЧЁТА ----------
        function updateScore() {
            playerScoreSpan.textContent = playerScore;
            aiScoreSpan.textContent = aiScore;
        }

        // ---------- ФИЗИКА + ИИ ----------
        function update() {
            if (paused || gameOver) return;

            // ---- Движение игрока ----
            if (keys.ArrowUp && player.y > 0) player.y -= PLAYER_SPEED;
            if (keys.ArrowDown && player.y + player.h < H) player.y += PLAYER_SPEED;

            // ---- Движение ИИ (умный!) ----
            // ИИ предсказывает, куда летит мяч, но с задержкой и погрешностью
            const aiCenter = ai.y + ai.h/2;
            const ballCenter = ball.y + ball.size/2;

            // Если мяч летит в сторону ИИ — активно двигаемся
            if (ball.vx > 0) {
                // Предсказываем, где будет мяч через 30 кадров (с учётом отскоков)
                let predictedY = ball.y;
                let tempVx = ball.vx;
                let tempVy = ball.vy;
                let tempX = ball.x;
                let tempY = ball.y;

                for (let i = 0; i < 30; i++) {
                    tempX += tempVx;
                    tempY += tempVy;
                    // Отскок от стен
                    if (tempY <= 0 || tempY + ball.size >= H) tempVy = -tempVy;
                    if (tempX + ball.size >= W) break; // если гол — не важно
                }
                // Смещаем цель к предсказанной позиции
                let targetY = tempY - ai.h/2 + ball.size/2;
                // Ограничиваем, чтобы не вылетело за поле
                targetY = Math.max(0, Math.min(H - ai.h, targetY));

                // Двигаемся к цели со скоростью AI_SPEED
                if (ai.y + ai.h/2 < targetY + ai.h/2) {
                    ai.y += AI_SPEED;
                } else {
                    ai.y -= AI_SPEED;
                }
            } else {
                // Мяч летит от ИИ — возвращаемся в центр
                const centerY = H/2 - ai.h/2;
                if (Math.abs(ai.y - centerY) > 2) {
                    if (ai.y < centerY) ai.y += AI_SPEED * 0.6;
                    else ai.y -= AI_SPEED * 0.6;
                }
            }

            // Не даём ИИ выйти за границы
            ai.y = Math.max(0, Math.min(H - ai.h, ai.y));

            // ---- Движение мяча ----
            ball.x += ball.vx;
            ball.y += ball.vy;

            // Отскок от верхней и нижней стен
            if (ball.y <= 0 || ball.y + ball.size >= H) {
                ball.vy = -ball.vy;
                ball.y = Math.max(0, Math.min(H - ball.size, ball.y));
            }

            // ---- Столкновение с ракеткой игрока ----
            if (ball.vx < 0 &&
                ball.x <= player.x + player.w &&
                ball.x + ball.size >= player.x &&
                ball.y + ball.size >= player.y &&
                ball.y <= player.y + player.h) {

                // Отскок с изменением угла в зависимости от места удара
                const hitPos = (ball.y + ball.size/2) - (player


.y + player.h/2);
                const normalized = hitPos / (player.h/2); // -1 .. 1
                const angle = normalized * Math.PI / 3.5; // макс 50 градусов
                const speed = Math.min(BALL_MAX_SPEED, Math.sqrt(ball.vx*ball.vx + ball.vy*ball.vy) * 1.05);
                ball.vx = Math.abs(speed * Math.cos(angle));
                ball.vy = speed * Math.sin(angle);
                ball.x = player.x + player.w + 1;
            }

            // ---- Столкновение с ракеткой ИИ ----
            if (ball.vx > 0 &&
                ball.x + ball.size >= ai.x &&
                ball.x <= ai.x + ai.w &&
                ball.y + ball.size >= ai.y &&
                ball.y <= ai.y + ai.h) {

                const hitPos = (ball.y + ball.size/2) - (ai.y + ai.h/2);
                const normalized = hitPos / (ai.h/2);
                const angle = normalized * Math.PI / 3.5;
                const speed = Math.min(BALL_MAX_SPEED, Math.sqrt(ball.vx*ball.vx + ball.vy*ball.vy) * 1.05);
                ball.vx = -Math.abs(speed * Math.cos(angle));
                ball.vy = speed * Math.sin(angle);
                ball.x = ai.x - ball.size - 1;
            }

            // ---- Голы ----
            if (ball.x + ball.size < 0) {
                // Гол ИИ
                aiScore++;
                updateScore();
                resetBall(1);
            } else if (ball.x > W) {
                // Гол игрока
                playerScore++;
                updateScore();
                resetBall(-1);
            }

            // Проверка на победу (10 очков)
            if (playerScore >= 10 || aiScore >= 10) {
                gameOver = true;
            }
        }

        // ---------- ОТРИСОВКА ----------
        function draw() {
            ctx.clearRect(0, 0, W, H);

            // Центральная линия
            ctx.setLineDash([12, 16]);
            ctx.strokeStyle = '#334466';
            ctx.lineWidth = 2;
            ctx.beginPath();
            ctx.moveTo(W/2, 0);
            ctx.lineTo(W/2, H);
            ctx.stroke();
            ctx.setLineDash([]);

            // Центральный круг
            ctx.strokeStyle = '#1a2a44';
            ctx.lineWidth = 1.5;
            ctx.beginPath();
            ctx.arc(W/2, H/2, 50, 0, Math.PI*2);
            ctx.stroke();

            // ---- Ракетки ----
            // Игрок (свечение)
            ctx.shadowColor = '#00ccff';
            ctx.shadowBlur = 20;
            ctx.fillStyle = '#00ccff';
            ctx.beginPath();
            ctx.roundRect(player.x, player.y, player.w, player.h, 6);
            ctx.fill();

            // ИИ (красное свечение)
            ctx.shadowColor = '#ff4477';
            ctx.shadowBlur = 20;
            ctx.fillStyle = '#ff4477';
            ctx.beginPath();
            ctx.roundRect(ai.x, ai.y, ai.w, ai.h, 6);
            ctx.fill();
            ctx.shadowBlur = 0;

            // ---- Мяч ----
            ctx.shadowColor = '#ffffff';
            ctx.shadowBlur = 25;
            ctx.fillStyle = '#ffffff';
            ctx.beginPath();
            ctx.arc(ball.x + ball.size/2, ball.y + ball.size/2, ball.size/2, 0, Math.PI*2);
            ctx.fill();

            // Блики на мяче
            ctx.shadowBlur = 0;
            ctx.fillStyle = 'rgba(255,255,255,0.3)';
            ctx.beginPath();
            ctx.arc(ball.x + ball.size/2 - 2, ball.y + ball.size/2 - 3, 3, 0, Math.PI*2);
            ctx.fill();

            // ---- Сообщения ----
            if (gameOver) {
                ctx.fillStyle = 'rgba(0,0,0,0.7)';
                ctx.fillRect(0, 0, W, H);
                ctx.fillStyle = '#ffffff';
                ctx.font = 'bold 48px "Courier New", monospace';
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                const msg = playerScore >= 10 ? '🏆 ТЫ ПОБЕДИЛ!' : '💀 ИИ ПОБЕДИЛ';
                ctx.fillText(msg, W/2, H/2 - 20);
                ctx.font = '20px "Courier New", monospace';


ctx.fillStyle = '#88ddff';
                ctx.fillText('Нажми "Рестарт"', W/2, H/2 + 50);
            } else if (paused) {
                ctx.fillStyle = 'rgba(0,0,0,0.5)';
                ctx.fillRect(0, 0, W, H);
                ctx.fillStyle = '#ffcc00';
                ctx.font = 'bold 40px "Courier New", monospace';
                ctx.textAlign = 'center';
                ctx.textBaseline = 'middle';
                ctx.fillText('⏸ ПАУЗА', W/2, H/2);
            }
        }

        // ---------- ХЕЛПЕР roundRect ----------
        CanvasRenderingContext2D.prototype.roundRect = function(x, y, w, h, r) {
            if (w < 2*r) r = w/2;
            if (h < 2*r) r = h/2;
            this.moveTo(x + r, y);
            this.lineTo(x + w - r, y);
            this.quadraticCurveTo(x + w, y, x + w, y + r);
            this.lineTo(x + w, y + h - r);
            this.quadraticCurveTo(x + w, y + h, x + w - r, y + h);
            this.lineTo(x + r, y + h);
            this.quadraticCurveTo(x, y + h, x, y + h - r);
            this.lineTo(x, y + r);
            this.quadraticCurveTo(x, y, x + r, y);
            this.closePath();
            return this;
        };

        // ---------- ИГРОВОЙ ЦИКЛ ----------
        function gameLoop() {
            update();
            draw();
            animationId = requestAnimationFrame(gameLoop);
        }

        // ---------- УПРАВЛЕНИЕ ----------
        function handleKey(e) {
            const key = e.key;
            if (key === ' ') {
                e.preventDefault();
                if (!gameOver) paused = !paused;
                return;
            }
            if (key === 'ArrowUp' || key === 'ArrowDown') {
                e.preventDefault();
                keys[key] = true;
            }
        }

        function handleKeyUp(e) {
            const key = e.key;
            if (key === 'ArrowUp' || key === 'ArrowDown') {
                e.preventDefault();
                keys[key] = false;
            }
        }

        // ---------- ПОДПИСКИ ----------
        window.addEventListener('keydown', handleKey);
        window.addEventListener('keyup', handleKeyUp);

        document.getElementById('restartBtn').addEventListener('click', () => {
            resetGame();
        });

        // ---------- СТАРТ ----------
        resetGame();
        gameLoop();
    })();
</script>
</body>
</html>
