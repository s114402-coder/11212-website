<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <title>吉伊卡哇彈幕戰：海賴勇者 vs 飛鼠小桃</title>
    <style>
        body { margin: 0; background: #222; display: flex; justify-content: center; align-items: center; height: 100vh; color: white; font-family: 'Arial', sans-serif; overflow: hidden; }
        #game-container { position: relative; border: 4px solid #ffccdd; background: #000; box-shadow: 0 0 20px rgba(255, 192, 203, 0.5); }
        canvas { display: block; }
        .boss-ui { position: absolute; top: 10px; left: 50%; transform: translateX(-50%); width: 80%; text-align: center; }
        .hp-bar-bg { width: 100%; height: 8px; background: rgba(255,255,255,0.2); border-radius: 4px; }
        #hpBar { width: 100%; height: 100%; background: linear-gradient(90deg, #ffde59, #ff914d); border-radius: 4px; transition: width 0.3s; }
        .player-ui { position: absolute; bottom: 5px; right: 10px; text-align: right; }
        #livesContainer { color: #ff3366; font-size: 1.8em; text-shadow: 0 0 10px #ff3366; }
        .info-ui { position: absolute; bottom: 5px; left: 10px; font-size: 12px; color: #ffccdd; }
        
        .cd-container { width: 100px; height: 6px; background: #333; margin: 5px 0; border-radius: 3px; overflow: hidden; }
        #cdBar { width: 100%; height: 100%; background: #00ccff; }
        .hint-text { font-size: 10px; color: #aaa; margin-top: 2px; }

        #startBtn { 
            position: absolute; 
            top: 50%; 
            left: 50%; 
            transform: translate(-50%, -50%); 
            padding: 20px 40px; 
            font-size: 24px; 
            cursor: pointer; 
            background: #ffccdd; 
            border: none; 
            border-radius: 15px; 
            color: #333; 
            font-weight: bold; 
            box-shadow: 0 0 15px rgba(255,255,255,0.5);
            z-index: 10;
        }
    </style>
</head>
<body>

<div id="game-container">
    <div class="boss-ui">
        <div style="font-size:14px; margin-bottom:4px;">對戰對象：海賴勇者 (師傅)</div>
        <div class="hp-bar-bg"><div id="hpBar"></div></div>
        <div id="attackPattern" style="font-size:12px; color:#ffd700; margin-top:5px;">哈哈哈!!!</div>
    </div>
    <div class="info-ui">
        <div>Score: <span id="score">0</span></div>
        <div id="powerLevel" style="color:#0ff;">彈道：Lv.1</div>
        <div style="margin-top:8px;">集氣彈 (Ctrl):</div>
        <div class="cd-container"><div id="cdBar"></div></div>
        <div class="hint-text">Shift: 精確閃避 (顯示判定點)</div>
    </div>
    <div class="player-ui">
        <div id="livesContainer"></div>
    </div>
    <canvas id="gameCanvas"></canvas>
    <button id="startBtn">點擊開始對戰</button>
</div>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const startBtn = document.getElementById('startBtn');
const hpBar = document.getElementById('hpBar');
const cdBar = document.getElementById('cdBar');
const livesContainer = document.getElementById('livesContainer');
const attackLabel = document.getElementById('attackPattern');

canvas.width = 450;
canvas.height = 650;

const bossImg = new Image();
bossImg.src = 'boss.webp';
const playerImg = new Image();
playerImg.src = 'player.jpg';

let audioCtx, masterGain;
let score = 0;
let frameCount = 0;
let keys = {};
let isGameOver = false;
let gameStarted = false;

const player = { 
    x: 225, y: 550, speed: 4.5, focusSpeed: 2, radius: 2, lives: 5, invincible: 0, 
    bullets: [],
    bigBullets: [],
    skillCD: 0,
    maxCD: 600,
    notifiedReady: true 
};
// Boss HP 提升以應對 3 倍傷害
const boss = { x: 225, y: 120, targetX: 225, targetY: 120, hp: 12000, maxHp: 12000, bullets: [] };

function initAudio() {
    audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    masterGain = audioCtx.createGain();
    masterGain.gain.value = 0.3;
    masterGain.connect(audioCtx.destination);
    playBGM();
}

function playSound(freq, type, duration, vol) {
    if (!audioCtx) return;
    const osc = audioCtx.createOscillator();
    const g = audioCtx.createGain();
    osc.type = type;
    osc.frequency.setValueAtTime(freq, audioCtx.currentTime);
    g.gain.setValueAtTime(vol, audioCtx.currentTime);
    g.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + duration);
    osc.connect(g);
    g.connect(masterGain);
    osc.start();
    osc.stop(audioCtx.currentTime + duration);
}

function playBGM() {
    const notes = [523, 587, 659, 698, 784, 0, 784, 698, 659, 587, 523];
    let i = 0;
    setInterval(() => {
        if (!isGameOver && notes[i] !== 0) playSound(notes[i], 'triangle', 0.2, 0.08);
        i = (i + 1) % notes.length;
    }, 250);
}

function update() {
    if (!gameStarted || isGameOver) return;
    frameCount++;

    let isFocus = keys['ShiftLeft'] || keys['ShiftRight'];
    let moveSpeed = isFocus ? player.focusSpeed : player.speed;
    if (keys['ArrowUp'] && player.y > 20) player.y -= moveSpeed;
    if (keys['ArrowDown'] && player.y < canvas.height - 20) player.y += moveSpeed;
    if (keys['ArrowLeft'] && player.x > 20) player.x -= moveSpeed;
    if (keys['ArrowRight'] && player.x < canvas.width - 20) player.x += moveSpeed;

    if (player.invincible > 0) player.invincible--;

    // 玩家射擊 (發射頻率不變，傷害提升)
    if (frameCount % 6 === 0) {
        player.bullets.push({ x: player.x, y: player.y - 10, vx: 0, vy: -12 });
        if (frameCount > 1800) {
            player.bullets.push({ x: player.x, y: player.y, vx: -3, vy: -11 });
            player.bullets.push({ x: player.x, y: player.y, vx: 3, vy: -11 });
            document.getElementById('powerLevel').innerText = "彈道：Lv.2";
        }
        if (frameCount > 3600) {
            player.bullets.push({ x: player.x, y: player.y, vx: -6, vy: -10 });
            player.bullets.push({ x: player.x, y: player.y, vx: 6, vy: -10 });
            document.getElementById('powerLevel').innerText = "彈道：MAX";
        }
    }

    if (player.skillCD > 0) {
        player.skillCD--;
        player.notifiedReady = false;
    } else {
        if (!player.notifiedReady) {
            playSound(1200, 'sine', 0.15, 0.2); 
            player.notifiedReady = true;
        }
        if (keys['ControlLeft'] || keys['ControlRight']) {
            player.bigBullets.push({ x: player.x, y: player.y - 20, vy: -7 });
            player.skillCD = player.maxCD;
            playSound(100, 'square', 0.8, 0.4); 
        }
    }

    // 更新普通子彈 (傷害改為 12)
    player.bullets.forEach((b, i) => {
        b.x += b.vx; b.y += b.vy;
        if (b.y < -10) { player.bullets.splice(i, 1); return; }
        if (Math.hypot(b.x - boss.x, b.y - boss.y) < 45) {
            boss.hp -= 12; // 3倍傷害
            score += 30; // 分數也相應增加
            playSound(800, 'square', 0.05, 0.05);
            player.bullets.splice(i, 1);
        }
    });

    // 更新集氣彈 (傷害改為 360)
    player.bigBullets.forEach((b, i) => {
        b.y += b.vy;
        if (b.y < -50) { player.bigBullets.splice(i, 1); return; }
        if (Math.hypot(b.x - boss.x, b.y - boss.y) < 60) {
            boss.hp -= 360; // 3倍傷害
            score += 1000;
            playSound(400, 'sine', 0.2, 0.3);
            player.bigBullets.splice(i, 1);
        }
    });

    // Boss 邏輯
    let moveFreq = boss.hp > 12000 ? 120 : 60;
    if (frameCount % moveFreq === 0) {
        boss.targetX = 100 + Math.random() * (canvas.width - 200);
        boss.targetY = 80 + Math.random() * 80;
    }
    boss.x += (boss.targetX - boss.x) * 0.05;
    boss.y += (boss.targetY - boss.y) * 0.05;

    let patternIndex = Math.floor(frameCount / 1800) % 2;
    if (frameCount % 70 === 0) {
        let angle = Math.atan2(player.y - boss.y, player.x - boss.x);
        createBossBullet(boss.x, boss.y, angle, 4.5, "#ff3333", 6);
    }
    if (patternIndex === 0) { 
        attackLabel.innerText = "哈哈哈哈哈！！！！！";
        if (frameCount % 15 === 0) {
            for (let i = 0; i < 12; i++) {
                let angle = (Math.PI * 2 / 12) * i + (frameCount * 0.03);
                createBossBullet(boss.x, boss.y, angle, 2.5, "#fff", 4);
            }
        }
    } else { 
        attackLabel.innerText = "試煉現在才開始";
        if (frameCount % 8 === 0) {
            boss.bullets.push({ x: Math.random() * canvas.width, y: -10, vx: 0, vy: 4, color: "#ffff00", r: 5 });
        }
    }

    boss.bullets.forEach((b, i) => {
        b.x += b.vx; b.y += b.vy;
        if (b.y > canvas.height + 20 || b.x < -20 || b.x > canvas.width + 20) {
            boss.bullets.splice(i, 1);
        } else if (player.invincible === 0 && Math.hypot(b.x - player.x, b.y - player.y) < player.radius + b.r) {
            player.lives--; player.invincible = 100;
            playSound(150, 'sawtooth', 0.3, 0.2);
            if (player.lives <= 0) { isGameOver = true; alert("嗚嗚...小桃被師傅打敗了！"); location.reload(); }
        }
    });

    hpBar.style.width = (boss.hp / boss.maxHp * 100) + "%";
    cdBar.style.width = (1 - (player.skillCD / player.maxCD)) * 100 + "%";
    document.getElementById('score').innerText = score;
    let hearts = ""; for(let i=0; i<player.lives; i++) hearts += "❤";
    livesContainer.innerText = hearts;
    if (boss.hp <= 0) { isGameOver = true; alert("太強了！小桃成功完成了師傅的特訓！"); location.reload(); }
}

function createBossBullet(x, y, angle, speed, color, r) {
    boss.bullets.push({ x, y, vx: Math.cos(angle)*speed, vy: Math.sin(angle)*speed, color, r });
}

function draw() {
    ctx.fillStyle = "#1a1a1a";
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    ctx.save();
    ctx.translate(boss.x, boss.y);
    ctx.rotate(Math.sin(frameCount * 0.05) * 0.1);
    ctx.drawImage(bossImg, -40, -40, 80, 80);
    ctx.restore();

    if (player.invincible % 6 < 3) {
        ctx.save();
        ctx.translate(player.x, player.y);
        ctx.drawImage(playerImg, -25, -25, 50, 50);
        if (keys['ShiftLeft'] || keys['ShiftRight']) {
            ctx.fillStyle = "red";
            ctx.beginPath(); ctx.arc(0, 0, 3, 0, Math.PI*2); ctx.fill();
            ctx.strokeStyle = "white"; ctx.lineWidth = 1; ctx.stroke();
        }
        ctx.restore();
    }

    ctx.fillStyle = "#fff";
    player.bullets.forEach(b => { ctx.fillRect(b.x - 3, b.y, 6, 12); });
    player.bigBullets.forEach(b => {
        ctx.fillStyle = "#00ccff";
        ctx.shadowBlur = 15; ctx.shadowColor = "#00ccff";
        ctx.beginPath(); ctx.arc(b.x, b.y, 18, 0, Math.PI*2); ctx.fill();
        ctx.shadowBlur = 0;
    });

    boss.bullets.forEach(b => {
        ctx.fillStyle = b.color;
        ctx.beginPath(); ctx.arc(b.x, b.y, b.r, 0, Math.PI*2); ctx.fill();
    });
}

function loop() {
    update();
    draw();
    requestAnimationFrame(loop);
}

startBtn.onclick = () => {
    gameStarted = true;
    startBtn.style.display = 'none';
    initAudio();
    loop();
};

window.addEventListener('keydown', e => keys[e.code] = true);
window.addEventListener('keyup', e => keys[e.code] = false);
</script>
</body>
</html>
