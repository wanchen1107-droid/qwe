<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>小恐龍進化 App</title>
    <style>
        * {
            /* 禁用手機長按選單與選取文字 */
            -webkit-touch-callout: none;
            -webkit-user-select: none;
            user-select: none;
            box-sizing: border-box;
        }

        body { 
            margin: 0; padding: 0; 
            display: flex; justify-content: center; align-items: center; 
            height: 100vh; background-color: #e0f7fa; 
            font-family: 'Arial', sans-serif; 
            overflow: hidden;
            touch-action: manipulation; /* 優化觸控延遲 */
        }

        #game-container { 
            position: relative; 
            width: 95vw; /* 改用比例，適應手機寬度 */
            max-width: 800px; 
            height: 250px; 
            background-color: white; 
            border-bottom: 2px solid #535353; 
            overflow: hidden; 
            border-radius: 8px; 
            box-shadow: 0 10px 20px rgba(0,0,0,0.1); 
        }
        
        /* --- 恐龍樣式 --- */
        #dino { 
            position: absolute; 
            bottom: 0; 
            left: 50px; 
            width: 50px; 
            height: 50px; 
            background-image: url('image_67211c.png'); 
            background-size: contain;
            background-repeat: no-repeat;
            z-index: 10; 
            transition: filter 0.3s; 
        }
        
        #dino.evo-gold { filter: sepia(1) saturate(5) hue-rotate(10deg) drop-shadow(0 0 5px #FFD700); }
        #dino.evo-flame { filter: saturate(2) hue-rotate(-10deg) drop-shadow(0 0 8px #ff4757); animation: flamePulse 0.5s infinite alternate; }
        
        @keyframes flamePulse { from { transform: scale(1); } to { transform: scale(1.08); } }

        /* 特效與障礙物 */
        .flame-particle { position: absolute; width: 6px; height: 6px; background: #ffa502; border-radius: 50%; animation: flameFly 0.6s linear forwards; }
        @keyframes flameFly { 0% { opacity: 1; transform: translate(0,0); } 100% { opacity: 0; transform: translate(-30px, -20px); } }

        .obstacle { position: absolute; bottom: 0; z-index: 5; background-color: #535353; border-radius: 4px; }
        .bird { bottom: 65px; border-radius: 50% 50% 0 0; }
        .cloud { position: absolute; border-radius: 50px; z-index: 1; opacity: 0.8; animation: cloudDrift linear infinite; }
        @keyframes cloudDrift { from { left: 110%; } to { left: -150px; } }

        /* UI */
        #ui-layer { position: absolute; top: 10px; width: 100%; display: flex; justify-content: flex-end; padding: 0 20px; z-index: 15; color: #535353; font-weight: bold; }
        #evo-status { position: absolute; top: 10px; left: 20px; font-weight: bold; z-index: 15; }
        #message { 
            position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); 
            text-align: center; display: none; z-index: 20; 
            background: rgba(255,255,255,0.95); padding: 25px; border-radius: 15px; border: 3px solid #535353; width: 80%;
        }
    </style>
</head>
<body>

    <div id="game-container">
        <div id="evo-status">狀態: 幼年體</div>
        <div id="ui-layer"><div id="score">SCORE: 00000</div></div>
        <div id="dino"></div>
        <div id="message">
            <h1 style="color: #ff4757; margin: 0;">GAME OVER</h1>
            <p id="final-score" style="font-size: 20px; font-weight: bold;"></p>
            <p>點擊螢幕重新挑戰</p>
        </div>
    </div>

<script>
    const container = document.getElementById('game-container');
    const dinoElement = document.getElementById('dino');
    const scoreElement = document.getElementById('score');
    const evoStatusElement = document.getElementById('evo-status');
    const messageElement = document.getElementById('message');
    const finalScoreElement = document.getElementById('final-score');

    let isGameRunning = true;
    let currentEvo = 'normal';
    let score = 0;
    let gameSpeed = 6;
    let baseGravity = 0.8;
    
    let dino = { x: 50, y: 0, vy: 0, width: 40, height: 45, isJumping: false };
    let obstacles = [];
    let cloudTimer = 0;
    let obstacleTimer = 0;
    let flameEffectTimer = 0;

    // --- 控制邏輯：同時支援鍵盤與觸控 ---
    function handleJump() {
        if (!isGameRunning) {
            resetGame();
        } else if (!dino.isJumping) { 
            dino.vy = -14; // 手機版稍微調低一點點跳躍高度
            dino.isJumping = true; 
        }
    }

    document.addEventListener('keydown', (e) => {
        if (e.code === 'Space' || e.code === 'ArrowUp') handleJump();
    });

    // 觸控事件
    container.addEventListener('touchstart', (e) => {
        handleJump();
        e.preventDefault(); // 防止縮放
    }, {passive: false});

    function gameLoop() {
        if (!isGameRunning) return;

        updateDino();
        updateObstacles();
        updateClouds();
        handleEvolution();
        checkCollisions();
        
        score += 0.15;
        gameSpeed = 6 + (score / 250); // 難度提升略微放緩
        
        scoreElement.innerText = 'SCORE: ' + Math.floor(score).toString().padStart(5, '0');
        requestAnimationFrame(gameLoop);
    }

    function updateDino() {
        dino.vy += baseGravity;
        dino.y -= dino.vy;
        if (dino.y <= 0) { dino.y = 0; dino.vy = 0; dino.isJumping = false; }
        dinoElement.style.bottom = dino.y + 'px';

        if (currentEvo === 'flame' && ++flameEffectTimer > 4) {
            createFlameParticle();
            flameEffectTimer = 0;
        }
    }

    function handleEvolution() {
        let s = Math.floor(score);
        if (s >= 1000 && currentEvo !== 'flame') {
            currentEvo = 'flame';
            dinoElement.className = 'evo-flame';
            evoStatusElement.innerText = '狀態: 火焰進化';
            evoStatusElement.style.color = '#ff4757';
        } else if (s >= 500 && s < 1000 && currentEvo !== 'gold') {
            currentEvo = 'gold';
            dinoElement.className = 'evo-gold';
            evoStatusElement.innerText = '狀態: 金色覺醒';
            evoStatusElement.style.color = '#FFB900';
        }
    }

    function createFlameParticle() {
        const p = document.createElement('div');
        p.className = 'flame-particle';
        p.style.left = (dino.x + 10) + 'px';
        p.style.bottom = (dino.y + 15) + 'px';
        container.appendChild(p);
        setTimeout(() => { if(p.parentNode) container.removeChild(p); }, 600);
    }

    function updateObstacles() {
        for (let i = obstacles.length - 1; i >= 0; i--) {
            obstacles[i].x -= gameSpeed;
            obstacles[i].element.style.left = obstacles[i].x + 'px';
            if (obstacles[i].x + obstacles[i].width < -50) {
                container.removeChild(obstacles[i].element);
                obstacles.splice(i, 1);
            }
        }
        if (++obstacleTimer > Math.max(50, 110 - gameSpeed * 2)) {
            createObstacle();
            obstacleTimer = 0;
        }
    }

    function createObstacle() {
        let rand = Math.random();
        let config = { type: 'cactus', width: 25, height: 45, y: 0 };
        if (rand > 0.8) config = { type: 'bird', width: 35, height: 25, y: 65 };
        const el = document.createElement('div');
        el.className = `obstacle ${config.type === 'bird' ? 'bird' : ''}`;
        el.style.left = '100%';
        el.style.bottom = config.y + 'px';
        el.style.width = config.width + 'px';
        el.style.height = config.height + 'px';
        container.appendChild(el);
        obstacles.push({ element: el, x: container.offsetWidth, y: config.y, width: config.width, height: config.height });
    }

    function updateClouds() {
        if (++cloudTimer > 120) {
            const el = document.createElement('div');
            el.className = 'cloud';
            const w = 60 + Math.random() * 40;
            el.style.width = w + 'px';
            el.style.height = (w * 0.5) + 'px';
            el.style.background = '#eee';
            el.style.top = (30 + Math.random() * 50) + 'px';
            el.style.animationDuration = (15 + Math.random() * 10) + 's';
            container.appendChild(el);
            setTimeout(() => { if(el.parentNode) container.removeChild(el); }, 25000);
            cloudTimer = 0;
        }
    }

    function checkCollisions() {
        // 進化後依然會死，挑戰高分！
        let dRect = { left: dino.x + 15, right: dino.x + dino.width - 15, top: dino.y + dino.height - 10, bottom: dino.y };
        for (let o of obstacles) {
            let oRect = { left: o.x + 5, right: o.x + o.width - 5, top: o.y + o.height, bottom: o.y };
            if (dRect.right > oRect.left && dRect.left < oRect.right && dRect.top > oRect.bottom && dRect.bottom < oRect.top) {
                gameOver();
            }
        }
    }

    function gameOver() { 
        isGameRunning = false; 
        finalScoreElement.innerText = "最終得分: " + Math.floor(score);
        messageElement.style.display = 'block'; 
    }

    function resetGame() {
        isGameRunning = true; currentEvo = 'normal'; score = 0; gameSpeed = 6; 
        dino.y = 0; dino.vy = 0; obstacleTimer = 0;
        messageElement.style.display = 'none';
        dinoElement.className = '';
        evoStatusElement.innerText = '狀態: 幼年體';
        evoStatusElement.style.color = '#535353';
        obstacles.forEach(o => container.removeChild(o.element));
        obstacles = [];
        gameLoop();
    }

    gameLoop();
</script>
</body>
</html>
