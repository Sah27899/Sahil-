# Sahil-
<!DOCTYPE html>
<html lang="hi">
<head>
  <meta charset="UTF-8">
  <title>Ball Dodging Pro+ Game</title>
  <style>
    * { margin:0; padding:0; box-sizing:border-box; }
    body { overflow:hidden; font-family:sans-serif; background:#111; color:#fff; }
    #startScreen, #gameOverScreen {
      position:absolute; top:0; left:0; right:0; bottom:0;
      display:flex; flex-direction:column; justify-content:center; align-items:center;
      background:rgba(0,0,0,0.8); z-index:2;
    }
    button { padding:12px 24px; font-size:18px; border:none; border-radius:8px;
             cursor:pointer; margin-top:10px; }
    #colorPicker button { width:40px; height:40px; border-radius:50%; margin:0 5px; }
    #scoreBoard {
      position:absolute; top:10px; left:10px; font-size:20px; z-index:1;
    }
    canvas { display:block; }
  </style>
</head>
<body>

<div id="startScreen">
  <h1>Ball Dodging Pro+</h1>
  <p>Choose Your Ball Color:</p>
  <div id="colorPicker">
    <button data-color="yellow" style="background:yellow;"></button>
    <button data-color="cyan"   style="background:cyan;"></button>
    <button data-color="magenta"style="background:magenta;"></button>
    <button data-color="lime"   style="background:lime;"></button>
  </div>
  <button id="startButton">Start Game</button>
</div>

<div id="gameOverScreen" style="display:none;">
  <h1>Game Over</h1>
  <p id="finalScore">Score: 0</p>
  <p id="bestScore">High Score: 0</p>
  <button id="restartButton">Play Again</button>
</div>

<div id="scoreBoard">Score: 0 | Shield: OFF</div>
<canvas id="gameCanvas"></canvas>

<script>
(() => {
  const canvas = document.getElementById("gameCanvas");
  const ctx    = canvas.getContext("2d");
  const startS = document.getElementById("startScreen");
  const overS  = document.getElementById("gameOverScreen");
  const startB = document.getElementById("startButton");
  const restartB = document.getElementById("restartButton");
  const scoreB = document.getElementById("scoreBoard");
  const finalScore = document.getElementById("finalScore");
  const bestScore  = document.getElementById("bestScore");
  const colorBtns  = document.querySelectorAll("#colorPicker button");

  let width, height;
  function resize() {
    width = canvas.width = window.innerWidth;
    height= canvas.height= window.innerHeight;
  }
  window.addEventListener("resize", resize);
  resize();

  // Game state
  let ball, obstacles, enemies, powerups, score, highScore, shield;
  let obstacleInterval, scoreInterval, enemyInterval;

  // Choose skin
  let chosenColor = "yellow";
  colorBtns.forEach(btn => {
    btn.onclick = () => {
      chosenColor = btn.dataset.color;
      colorBtns.forEach(b=> b.style.opacity=0.5);
      btn.style.opacity = 1;
    };
  });

  startB.onclick = startGame;
  restartB.onclick = () => location.reload();

  function startGame() {
    startS.style.display = "none";
    highScore = parseInt(localStorage.getItem("highScore")) || 0;

    // Init
    ball = { x: width/2, y: height-60, r:20, color:chosenColor, speed:10 };
    obstacles = []; enemies = []; powerups = [];
    score = 0; shield = false;

    obstacleInterval = setInterval(createObstacle, 1200);
    enemyInterval    = setInterval(createEnemy, 3000);
    scoreInterval    = setInterval(() => {
      score++;
      // spawn powerup occasionally
      if (score % 400 === 0) createPowerup();
      updateScoreBoard();
    }, 100);

    requestAnimationFrame(gameLoop);
  }

  function createObstacle() {
    obstacles.push({
      x: Math.random() * (width-40),
      y: -20, w:40, h:20,
      speed: 3 + Math.floor(score/300)*0.5
    });
  }

  function createEnemy() {
    enemies.push({
      x: Math.random() * (width-30),
      y: Math.random()*height/2,
      dx: (Math.random()*2-1)*2,
      dy: (Math.random()*2-1)*2,
      r:15
    });
  }

  function createPowerup() {
    powerups.push({
      x: Math.random()*(width-30),
      y: -30, size:30, speed:2
    });
  }

  function drawBackground() {
    if (score < 500) ctx.fillStyle = "#1e3c72";
    else if (score <1000) ctx.fillStyle = "#6a0dad";
    else ctx.fillStyle = "#8b0000";
    ctx.fillRect(0,0,width,height);
  }

  function drawBall() {
    ctx.beginPath();
    ctx.arc(ball.x, ball.y, ball.r, 0,2*Math.PI);
    ctx.fillStyle = ball.color; ctx.fill();
    ctx.closePath();
    if (shield) {
      ctx.beginPath();
      ctx.arc(ball.x, ball.y, ball.r+8, 0,2*Math.PI);
      ctx.strokeStyle = "white"; ctx.lineWidth=3; ctx.stroke();
      ctx.closePath();
    }
  }

  function drawObstacles() {
    ctx.fillStyle = "black";
    obstacles.forEach(o => ctx.fillRect(o.x,o.y,o.w,o.h));
  }

  function drawEnemies() {
    ctx.fillStyle = "red";
    enemies.forEach(e => {
      ctx.beginPath();
      ctx.arc(e.x,e.y,e.r,0,2*Math.PI);
      ctx.fill(); ctx.closePath();
    });
  }

  function drawPowerups() {
    ctx.fillStyle = "gold";
    powerups.forEach(p => ctx.fillRect(p.x,p.y,p.size,p.size));
  }

  function updateEntities() {
    obstacles.forEach(o => o.y += o.speed);
    enemies.forEach(e => { e.x+=e.dx; e.y+=e.dy;
      // bounce
      if (e.x<0||e.x>width) e.dx*=-1;
      if (e.y<0||e.y>height) e.dy*=-1;
    });
    powerups.forEach(p => p.y += p.speed);

    obstacles = obstacles.filter(o => o.y<height+20);
    powerups  = powerups.filter(p => p.y<height+30);
  }

  function checkCollision() {
    // obstacles
    for (let o of obstacles) {
      if (ball.x+ball.r>o.x && ball.x-ball.r<o.x+o.w &&
          ball.y+ball.r>o.y && ball.y-ball.r<o.y+o.h) {
        if (shield) { shield=false; return; }
        return endGame();
      }
    }
    // enemies
    for (let e of enemies) {
      const dx = ball.x - e.x, dy = ball.y - e.y;
      if (Math.hypot(dx,dy) < ball.r + e.r) {
        if (shield) { shield=false; return; }
        return endGame();
      }
    }
    // powerups
    powerups.forEach((p,i) => {
      if (ball.x+ball.r>p.x && ball.x-ball.r<p.x+p.size &&
          ball.y+ball.r>p.y && ball.y-ball.r<p.y+p.size) {
        shield = true;
        powerups.splice(i,1);
      }
    });
  }

  function updateScoreBoard() {
    scoreB.innerText = 
      `Score: ${score} | Shield: ${shield? "ON":"OFF"}`;
  }

  function endGame() {
    clearInterval(obstacleInterval);
    clearInterval(enemyInterval);
    clearInterval(scoreInterval);
    // high score
    if (score > highScore) {
      highScore = score;
      localStorage.setItem("highScore", highScore);
    }
    finalScore.innerText = `Score: ${score}`;
    bestScore.innerText  = `High Score: ${highScore}`;
    overS.style.display = "flex";
  }

  function gameLoop() {
    drawBackground();
    updateEntities();
    drawBall();
    drawObstacles();
    drawEnemies();
    drawPowerups();
    checkCollision();
    if (overS.style.display==="none") requestAnimationFrame(gameLoop);
  }

  // controls
  document.addEventListener("keydown", e => {
    if (e.key==="ArrowLeft") ball.x -= ball.speed;
    else if (e.key==="ArrowRight") ball.x += ball.speed;
  });
})();
</script>
