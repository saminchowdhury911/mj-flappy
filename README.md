<!DOCTYPE html>
<html>
<head>
    <title>Flappy MJ: The Legend Edition</title>
    <style>
        body { margin: 0; background: #000; display: flex; justify-content: center; align-items: center; height: 100vh; overflow: hidden; color: white; font-family: 'Arial', sans-serif; }
        canvas { border: 3px solid #333; box-shadow: 0 0 50px rgba(0, 255, 255, 0.3); cursor: pointer; }
    </style>
</head>
<body>
    <canvas id="gameCanvas"></canvas>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        canvas.width = 360;
        canvas.height = 640;

        // --- 1. ASSETS ---
        const playerImg = new Image(); playerImg.src = 'mj.png'; 
        const pipeImg = new Image(); pipeImg.src = 'pipe.png'; 
        const jumpSound = new Audio('jump.mp3'); 
        const bgMusic = new Audio('music bgm.mp3'); 
        const deathSound = new Audio('death.mp3'); 
        bgMusic.loop = true;

        // --- 2. STATES & VARIABLES ---
        let gameState = 'MENU';
        let birdY, velocity, score, pipes, particles;
        let shakeDuration = 0; // Tracks the shake effect
        const gravity = 0.22;
        const jump = -4.8;
        const pipeWidth = 60;
        const pipeGap = 170;
        let highScore = localStorage.getItem('mjHighScore') || 0;

        function init() {
            birdY = 300; velocity = 0; score = 0;
            pipes = []; particles = [];
            shakeDuration = 0;
        }

        function createParticle(x, y) {
            particles.push({
                x: x, y: y,
                size: Math.random() * 5 + 2,
                speedX: (Math.random() - 0.5) * 2 - 2,
                speedY: (Math.random() - 0.5) * 2,
                life: 1.0,
                color: `hsl(${Math.random() * 60 + 180}, 100%, 70%)`
            });
        }

        function draw() {
            ctx.save(); // Save the clean state before shaking

            // --- SCREEN SHAKE LOGIC ---
            if (shakeDuration > 0) {
                let shakeX = (Math.random() - 0.5) * 10; // Shakes left/right
                let shakeY = (Math.random() - 0.5) * 10; // Shakes up/down
                ctx.translate(shakeX, shakeY);
                shakeDuration--;
            }

            ctx.fillStyle = "#0a0a1a";
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            if (gameState === 'MENU') {
                ctx.textAlign = "center";
                ctx.fillStyle = "white";
                ctx.font = "bold 35px Arial";
                ctx.fillText("FLAPPY MJ", canvas.width/2, 120);
                if(playerImg.complete) {
                    ctx.shadowBlur = 30; ctx.shadowColor = "cyan";
                    ctx.drawImage(playerImg, canvas.width/2 - 60, 180, 120, 120);
                    ctx.shadowBlur = 0;
                }
                ctx.fillStyle = "#f1c40f"; 
                ctx.font = "bold 22px Arial";
                ctx.fillText("BEST SCORE: " + highScore, canvas.width/2, 360);
                ctx.fillStyle = "#00ffff";
                ctx.fillText("TAP TO START", canvas.width/2, 420);

            } else if (gameState === 'PLAYING') {
                velocity += gravity;
                birdY += velocity;
                if (Math.random() > 0.4) createParticle(60, birdY + 20);

                if (pipes.length === 0 || pipes[pipes.length - 1].x < canvas.width - 220) {
                    let topHeight = Math.random() * (canvas.height / 3) + 50;
                    pipes.push({ x: canvas.width, y: topHeight, passed: false });
                }

                for (let i = pipes.length - 1; i >= 0; i--) {
                    pipes[i].x -= 2.5;
                    if (pipeImg.complete && pipeImg.naturalHeight !== 0) {
                        ctx.drawImage(pipeImg, pipes[i].x, 0, pipeWidth, pipes[i].y);
                        ctx.drawImage(pipeImg, pipes[i].x, pipes[i].y + pipeGap, pipeWidth, canvas.height);
                    } else {
                        ctx.fillStyle = "#2ecc71";
                        ctx.fillRect(pipes[i].x, 0, pipeWidth, pipes[i].y);
                        ctx.fillRect(pipes[i].x, pipes[i].y + pipeGap, pipeWidth, canvas.height);
                    }

                    if (50 + 30 > pipes[i].x && 50 < pipes[i].x + pipeWidth &&
                        (birdY < pipes[i].y || birdY + 30 > pipes[i].y + pipeGap)) {
                        endGame();
                    }
                    if (pipes[i].x + pipeWidth < 50 && !pipes[i].passed) {
                        score++; pipes[i].passed = true;
                    }
                    if (pipes[i].x < -pipeWidth) pipes.splice(i, 1);
                }
                if (birdY + 30 > canvas.height || birdY < 0) endGame();

                ctx.fillStyle = "white";
                ctx.font = "bold 40px Arial";
                ctx.textAlign = "center";
                ctx.fillText(score, canvas.width / 2, 60);

            } else if (gameState === 'GAMEOVER') {
                ctx.fillStyle = "rgba(0,0,0,0.8)";
                ctx.fillRect(0, 0, canvas.width, canvas.height);
                ctx.textAlign = "center";
                ctx.fillStyle = "white";
                ctx.font = "bold 40px Arial";
                ctx.fillText("HEE-HEE!", canvas.width/2, canvas.height/2 - 80);
                ctx.font = "24px Arial";
                ctx.fillText("Score: " + score, canvas.width/2, canvas.height/2 - 20);
                ctx.fillStyle = "#f1c40f";
                ctx.fillText("Best: " + highScore, canvas.width/2, canvas.height/2 + 20);
                ctx.fillStyle = "#00ffff";
                ctx.fillText("Click to Try Again", canvas.width/2, canvas.height/2 + 80);
            }

            // Draw Particles
            for (let i = particles.length - 1; i >= 0; i--) {
                let p = particles[i];
                p.x += p.speedX; p.y += p.speedY; p.life -= 0.02;
                if (p.life <= 0) { particles.splice(i, 1); continue; }
                ctx.globalAlpha = p.life;
                ctx.fillStyle = p.color;
                ctx.beginPath(); ctx.arc(p.x, p.y, p.size, 0, Math.PI * 2); ctx.fill();
            }
            ctx.globalAlpha = 1.0;

            if (gameState !== 'MENU') {
                ctx.save();
                ctx.translate(50 + 20, birdY + 20);
                ctx.rotate(velocity * 0.08);
                if(playerImg.complete) ctx.drawImage(playerImg, -20, -20, 40, 40);
                else { ctx.fillStyle = "white"; ctx.fillRect(-15, -15, 30, 30); }
                ctx.restore();
            }

            ctx.restore(); // Restore from the shake translate
            requestAnimationFrame(draw);
        }

        function endGame() {
            if (gameState !== 'GAMEOVER') {
                gameState = 'GAMEOVER';
                shakeDuration = 20; // Trigger 20 frames of shaking
                bgMusic.pause();
                deathSound.currentTime = 0;
                deathSound.play().catch(e => {});
                if (score > highScore) {
                    highScore = score;
                    localStorage.setItem('mjHighScore', highScore);
                }
            }
        }

        const handleAction = () => {
            if (gameState === 'MENU' || gameState === 'GAMEOVER') {
                init(); gameState = 'PLAYING';
                bgMusic.currentTime = 0; bgMusic.play().catch(e => {});
            } else {
                velocity = jump;
                jumpSound.currentTime = 0; jumpSound.play().catch(e => {});
            }
        };

        window.addEventListener('keydown', (e) => { if(e.code === 'Space') handleAction(); });
        canvas.addEventListener('mousedown', handleAction);
        canvas.addEventListener('touchstart', (e) => { e.preventDefault(); handleAction(); });

        init();
        draw();
    </script>
</body>
</html>
