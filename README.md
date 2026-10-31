const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

const scoreHomeEl = document.getElementById('score-home');
const scoreAwayEl = document.getElementById('score-away');
const timerEl = document.getElementById('timer');

const FIELD = {
  width: canvas.width,
  height: canvas.height,
  goalHeight: 150,
  goalDepth: 27,
  penaltyBoxWidth: 110,
  penaltyBoxHeight: 220,
};

const keys = {};

window.addEventListener('keydown', (event) => {
  keys[event.key.toLowerCase()] = true;
});

window.addEventListener('keyup', (event) => {
  keys[event.key.toLowerCase()] = false;
});

const game = {
  lastTime: 0,
  seconds: 90,
  score: { home: 0, away: 0 },
  kickoff: true,
};

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

function distance(a, b) {
  return Math.hypot(a.x - b.x, a.y - b.y);
}

function resetBall() {
  ball.x = FIELD.width / 2;
  ball.y = FIELD.height / 2;
  ball.vx = 0;
  ball.vy = 0;
  game.kickoff = true;
}

const ball = {
  x: FIELD.width / 2,
  y: FIELD.height / 2,
  radius: 9,
  vx: 0,
  vy: 0,
};

function createPlayer(x, y, color, team, label, controlled = false) {
  return {
    x,
    y,
    radius: 16,
    color,
    team,
    label,
    controlled,
    vx: 0,
    vy: 0,
    speed: 230,
    sprint: 300,
  };
}

const homePlayers = [
  createPlayer(160, 270, '#7ec8ff', 'home', 'P1', true),
  createPlayer(300, 170, '#7ec8ff', 'home', 'P2'),
  createPlayer(300, 420, '#7ec8ff', 'home', 'P3'),
  createPlayer(460, 240, '#7ec8ff', 'home', 'P4'),
  createPlayer(460, 360, '#7ec8ff', 'home', 'P5'),
];

const awayPlayers = [
  createPlayer(FIELD.width - 160, 270, '#ff9a9a', 'away', 'A1'),
  createPlayer(FIELD.width - 300, 170, '#ff9a9a', 'away', 'A2'),
  createPlayer(FIELD.width - 300, 420, '#ff9a9a', 'away', 'A3'),
  createPlayer(FIELD.width - 460, 240, '#ff9a9a', 'away', 'A4'),
  createPlayer(FIELD.width - 460, 360, '#ff9a9a', 'away', 'A5'),
];

const teams = { home: homePlayers, away: awayPlayers };

function updateScoreboard() {
  scoreHomeEl.textContent = game.score.home;
  scoreAwayEl.textContent = game.score.away;
}

function updateTimer(delta) {
  game.seconds -= delta;
  if (game.seconds <= 0) {
    game.seconds = 0;
  }

  const minutes = String(Math.floor(game.seconds / 60)).padStart(2, '0');
  const seconds = String(Math.floor(game.seconds % 60)).padStart(2, '0');
  timerEl.textContent = `${minutes}:${seconds}`;
}

function handleInput(player, delta) {
  let moveX = 0;
  let moveY = 0;

  if (keys.w) moveY -= 1;
  if (keys.s) moveY += 1;
  if (keys.a) moveX -= 1;
  if (keys.d) moveX += 1;

  const normalized = Math.hypot(moveX, moveY) || 1;
  const speed = keys.shift ? player.sprint : player.speed;

  if (moveX || moveY) {
    player.x += (moveX / normalized) * speed * delta;
    player.y += (moveY / normalized) * speed * delta;
  }

  player.x = clamp(player.x, 18, FIELD.width - 18);
  player.y = clamp(player.y, 18, FIELD.height - 18);
}

function moveTeamAI(teamPlayers, ballTarget, delta) {
  teamPlayers.forEach((player, idx) => {
    if (player.controlled) return;

    const offsetGoal = player.team === 'home' ? 0 : 1;
    const targetX = ballTarget.x + (idx - 2) * 18 * (player.team === 'home' ? 1 : -1);
    const targetY = ballTarget.y + (idx % 2 === 0 ? -70 : 70) + (player.team === 'home' ? 0 : 0);

    let moveX = 0;
    let moveY = 0;

    let desiredX = player.team === 'home' ? clamp(targetX, 70, FIELD.width - 160) : clamp(targetX, 160, FIELD.width - 70);
    let desiredY = clamp(targetY, 80, FIELD.height - 80);

    if (distance(player, ball) < 170) {
      desiredX = ball.x + (player.team === 'home' ? 22 : -22);
      desiredY = ball.y;
    }

    if (player.team === 'away') {
      desiredX = clamp(ball.x + 20 + (idx - 2) * 20, 150, FIELD.width - 60);
    }

    moveX = desiredX - player.x;
    moveY = desiredY - player.y;

    const len = Math.hypot(moveX, moveY) || 1;
    const moveSpeed = player.speed * 0.9;

    player.x += (moveX / len) * moveSpeed * delta;
    player.y += (moveY / len) * moveSpeed * delta;

    player.x = clamp(player.x, 18, FIELD.width - 18);
    player.y = clamp(player.y, 18, FIELD.height - 18);
  });
}

function handleBallCollision(player, delta) {
  const dx = ball.x - player.x;
  const dy = ball.y - player.y;
  const minDistance = player.radius + ball.radius + 2;

  if (Math.hypot(dx, dy) < minDistance) {
    const angle = Math.atan2(dy, dx);
    const targetX = player.x + Math.cos(angle) * minDistance;
    const targetY = player.y + Math.sin(angle) * minDistance;

    ball.x = targetX;
    ball.y = targetY;

    const kickStrength = 540;
    const pushX = Math.cos(angle) * kickStrength;
    const pushY = Math.sin(angle) * kickStrength;

    ball.vx += pushX * 0.65;
    ball.vy += pushY * 0.65;

    if (keys[' ']) {
      ball.vx += pushX * 0.8;
      ball.vy += pushY * 0.8;
    }
  }
}

function updateBall(delta) {
  ball.x += ball.vx * delta;
  ball.y += ball.vy * delta;

  ball.vx *= 0.988;
  ball.vy *= 0.988;

  if (Math.abs(ball.vx) < 0.2) ball.vx = 0;
  if (Math.abs(ball.vy) < 0.2) ball.vy = 0;

  if (ball.y - ball.radius < 0) {
    ball.y = ball.radius;
    ball.vy *= -0.85;
  }

  if (ball.y + ball.radius > FIELD.height) {
    ball.y = FIELD.height - ball.radius;
    ball.vy *= -0.85;
  }

  if (ball.x < 0) {
    if (ball.y > (FIELD.height - FIELD.goalHeight) / 2 && ball.y < (FIELD.height + FIELD.goalHeight) / 2) {
      game.score.away += 1;
      updateScoreboard();
      resetBall();
      return;
    }
    ball.x = 0;
    ball.vx *= -0.85;
  }

  if (ball.x > FIELD.width) {
    if (ball.y > (FIELD.height - FIELD.goalHeight) / 2 && ball.y < (FIELD.height + FIELD.goalHeight) / 2) {
      game.score.home += 1;
      updateScoreboard();
      resetBall();
      return;
    }
    ball.x = FIELD.width;
    ball.vx *= -0.85;
  }

  if (ball.x < 0 || ball.x > FIELD.width || ball.y < 0 || ball.y > FIELD.height) {
    ball.vx *= 0.9;
    ball.vy *= 0.9;
  }

  if (game.kickoff && Math.abs(ball.x - FIELD.width / 2) < 12 && Math.abs(ball.y - FIELD.height / 2) < 12) {
    game.kickoff = false;
  }
}

function shootBall(player) {
  const dx = ball.x - player.x;
  const dy = ball.y - player.y;
  const distanceToBall = Math.hypot(dx, dy);

  if (distanceToBall < 40 && (keys[' '] || keys.k)) {
    const angle = Math.atan2(dy, dx);
    ball.vx = Math.cos(angle) * 700;
    ball.vy = Math.sin(angle) * 700;
  }
}

function updatePlayers(delta) {
  const controlled = homePlayers[0];
  handleInput(controlled, delta);

  homePlayers.forEach((player) => {
    if (player !== controlled) {
      moveTeamAI(homePlayers, ball, delta);
      return;
    }
  });

  awayPlayers.forEach((player) => {
    moveTeamAI(awayPlayers, ball, delta);
  });

  homePlayers.forEach((player) => {
    if (player !== controlled) {
      handleBallCollision(player, delta);
    }
  });

  awayPlayers.forEach((player) => {
    handleBallCollision(player, delta);
  });

  handleBallCollision(controlled, delta);
  shootBall(controlled);

  homePlayers.forEach((player) => {
    if (player !== controlled) {
      const dx = ball.x - player.x;
      const dy = ball.y - player.y;
      const dyNorm = Math.hypot(dx, dy) || 1;
      if (Math.abs(dx) < 18 && Math.abs(dy) < 18) {
        ball.vx += (dx / dyNorm) * 10;
        ball.vy += (dy / dyNorm) * 10;
      }
    }
  });
}

function drawField() {
  ctx.clearRect(0, 0, FIELD.width, FIELD.height);

  ctx.fillStyle = '#3ba653';
  ctx.fillRect(0, 0, FIELD.width, FIELD.height);

  ctx.strokeStyle = 'rgba(255,255,255,0.9)';
  ctx.lineWidth = 3;
  ctx.strokeRect(14, 14, FIELD.width - 28, FIELD.height - 28);

  ctx.beginPath();
  ctx.moveTo(FIELD.width / 2, 14);
  ctx.lineTo(FIELD.width / 2, FIELD.height - 14);
  ctx.stroke();

  ctx.beginPath();
  ctx.arc(FIELD.width / 2, FIELD.height / 2, 70, 0, Math.PI * 2);
  ctx.stroke();

  ctx.strokeRect(14, (FIELD.height - FIELD.goalHeight) / 2, 28, FIELD.goalHeight);
  ctx.strokeRect(FIELD.width - 42, (FIELD.height - FIELD.goalHeight) / 2, 28, FIELD.goalHeight);

  ctx.strokeRect(14, (FIELD.height - FIELD.penaltyBoxHeight) / 2, FIELD.penaltyBoxWidth, FIELD.penaltyBoxHeight);
  ctx.strokeRect(FIELD.width - FIELD.penaltyBoxWidth - 14, (FIELD.height - FIELD.penaltyBoxHeight) / 2, FIELD.penaltyBoxWidth, FIELD.penaltyBoxHeight);

  ctx.beginPath();
  ctx.arc(FIELD.width / 2, FIELD.height / 2, 6, 0, Math.PI * 2);
  ctx.fillStyle = 'rgba(255,255,255,0.9)';
  ctx.fill();
}

function drawBall() {
  ctx.beginPath();
  ctx.fillStyle = '#f5f5f5';
  ctx.arc(ball.x, ball.y, ball.radius, 0, Math.PI * 2);
  ctx.fill();

  ctx.beginPath();
  ctx.lineWidth = 2;
  ctx.strokeStyle = '#ccc';
  ctx.arc(ball.x, ball.y, ball.radius, 0, Math.PI * 2);
  ctx.stroke();
}

function drawPlayer(player) {
  ctx.beginPath();
  ctx.fillStyle = player.color;
  ctx.arc(player.x, player.y, player.radius, 0, Math.PI * 2);
  ctx.fill();

  ctx.beginPath();
  ctx.fillStyle = '#111';
  ctx.arc(player.x, player.y, 6, 0, Math.PI * 2);
  ctx.fill();

  ctx.font = '12px Arial';
  ctx.fillStyle = '#ffffff';
  ctx.textAlign = 'center';
  ctx.fillText(player.label, player.x, player.y - 25);
}

function render() {
  drawField();

  [...homePlayers, ...awayPlayers].forEach(drawPlayer);
  drawBall();
}

function gameLoop(timestamp) {
  const delta = Math.min((timestamp - game.lastTime) / 1000 || 0.016, 0.03);
  game.lastTime = timestamp;

  updateTimer(delta);
  updatePlayers(delta);
  updateBall(delta);
  render();

  requestAnimationFrame(gameLoop);
}

updateScoreboard();
resetBall();
requestAnimationFrame(gameLoop);
