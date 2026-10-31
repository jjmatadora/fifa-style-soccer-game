const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

const ui = {
  scoreHome: document.getElementById('score-home'),
  scoreAway: document.getElementById('score-away'),
  timer: document.getElementById('timer'),
  status: document.getElementById('status'),
};

const FIELD = {
  width: canvas.width,
  height: canvas.height,
  cx: canvas.width / 2,
  cy: canvas.height / 2,
  goalHeight: 150,
  boxWidth: 110,
  boxHeight: 220,
};

const keys = {};
window.addEventListener('keydown', (event) => {
  const key = event.key.toLowerCase();
  keys[key] = true;
  if (key === ' ') event.preventDefault();
});
window.addEventListener('keyup', (event) => {
  keys[event.key.toLowerCase()] = false;
});

const game = {
  timer: 90,
  lastTime: 0,
  score: { home: 0, away: 0 },
};

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

function distance(a, b) {
  return Math.hypot(a.x - b.x, a.y - b.y);
}

function setStatus(text) {
  ui.status.textContent = text;
}

function syncHud() {
  ui.scoreHome.textContent = game.score.home;
  ui.scoreAway.textContent = game.score.away;

  const minutes = String(Math.floor(game.timer / 60)).padStart(2, '0');
  const seconds = String(Math.floor(game.timer % 60)).padStart(2, '0');
  ui.timer.textContent = `${minutes}:${seconds}`;
}

function createPlayer(team, name, x, y, primary, secondary, controlled = false) {
  return {
    team,
    name,
    x,
    y,
    radius: 16,
    primary,
    secondary,
    controlled,
    facingX: 1,
    facingY: 0,
    speed: 240,
    sprint: 320,
  };
}

const homePlayers = [
  createPlayer('home', 'Vega', 180, 270, '#70d4ff', '#2d7ddb', true),
  createPlayer('home', 'Costa', 330, 160, '#70d4ff', '#2d7ddb'),
  createPlayer('home', 'Silva', 330, 400, '#70d4ff', '#2d7ddb'),
  createPlayer('home', 'Torres', 470, 220, '#70d4ff', '#2d7ddb'),
  createPlayer('home', 'Ramos', 470, 340, '#70d4ff', '#2d7ddb'),
];

const awayPlayers = [
  createPlayer('away', 'Kane', FIELD.width - 180, 270, '#ff9a9a', '#d34141'),
  createPlayer('away', 'Stone', FIELD.width - 330, 160, '#ff9a9a', '#d34141'),
  createPlayer('away', 'Lopez', FIELD.width - 330, 400, '#ff9a9a', '#d34141'),
  createPlayer('away', 'Gomez', FIELD.width - 470, 220, '#ff9a9a', '#d34141'),
  createPlayer('away', 'Brooks', FIELD.width - 470, 340, '#ff9a9a', '#d34141'),
];

const ball = {
  x: FIELD.cx,
  y: FIELD.cy,
  radius: 10,
  vx: 0,
  vy: 0,
  owner: null,
  trail: [],
};

function resetBall() {
  ball.x = FIELD.cx;
  ball.y = FIELD.cy;
  ball.vx = 0;
  ball.vy = 0;
  ball.owner = null;
  ball.trail = [];
}

function scoreGoal(team) {
  if (team === 'home') {
    game.score.home += 1;
    setStatus('Home score!');
  } else {
    game.score.away += 1;
    setStatus('Away score!');
  }
  syncHud();
  resetBall();
}

function goalCheck() {
  const inGoalY = ball.y > FIELD.cy - FIELD.goalHeight / 2 && ball.y < FIELD.cy + FIELD.goalHeight / 2;

  if (ball.x <= 0 && inGoalY) {
    scoreGoal('away');
    return;
  }

  if (ball.x >= FIELD.width && inGoalY) {
    scoreGoal('home');
    return;
  }

  if (ball.x <= 0) {
    ball.x = ball.radius;
    ball.vx *= -0.9;
  }
  if (ball.x >= FIELD.width) {
    ball.x = FIELD.width - ball.radius;
    ball.vx *= -0.9;
  }
  if (ball.y <= 0) {
    ball.y = ball.radius;
    ball.vy *= -0.9;
  }
  if (ball.y >= FIELD.height) {
    ball.y = FIELD.height - ball.radius;
    ball.vy *= -0.9;
  }
}

function applyInput(player, delta) {
  let moveX = 0;
  let moveY = 0;

  if (keys.w) moveY -= 1;
  if (keys.s) moveY += 1;
  if (keys.a) moveX -= 1;
  if (keys.d) moveX += 1;

  if (moveX !== 0 || moveY !== 0) {
    const len = Math.hypot(moveX, moveY) || 1;
    const speed = keys.shift ? player.sprint : player.speed;
    player.x += (moveX / len) * speed * delta;
    player.y += (moveY / len) * speed * delta;
    player.facingX = moveX / len;
    player.facingY = moveY / len;
  }

  player.x = clamp(player.x, 18, FIELD.width - 18);
  player.y = clamp(player.y, 18, FIELD.height - 18);
}

function updateControlledPlayer(delta) {
  const player = homePlayers[0];
  applyInput(player, delta);

  if (ball.owner === player) {
    const offset = 20;
    ball.x = player.x + player.facingX * offset;
    ball.y = player.y + player.facingY * offset;
  }

  if (ball.owner === null) {
    if (distance(player, ball) < player.radius + ball.radius + 10) {
      ball.owner = player;
      setStatus(`${player.name} has the ball`);
    }
  }

  if (ball.owner === player) {
    if (keys[' ']) {
      const targetX = FIELD.width;
      const targetY = FIELD.cy;
      const dx = targetX - player.x;
      const dy = targetY - player.y;
      const length = Math.hypot(dx, dy) || 1;
      ball.owner = null;
      ball.vx = (dx / length) * 780;
      ball.vy = (dy / length) * 780;
      setStatus(`${player.name} shoots!`);
    }

    if (keys.e) {
      const teammates = homePlayers.filter((p) => p !== player);
      const target = teammates.reduce((best, candidate) => {
        if (!best) return candidate;
        return distance(player, candidate) < distance(player, best) ? candidate : best;
      }, null);

      if (target) {
        const dx = target.x - player.x;
        const dy = target.y - player.y;
        const length = Math.hypot(dx, dy) || 1;
        ball.owner = null;
        ball.vx = (dx / length) * 640;
        ball.vy = (dy / length) * 640;
        ball.x = player.x + (dx / length) * 18;
        ball.y = player.y + (dy / length) * 18;
        setStatus(`${player.name} passes to ${target.name}`);
      }
    }
  }
}

function moveAIPlayer(player, targetX, targetY, delta) {
  const dx = targetX - player.x;
  const dy = targetY - player.y;
  const len = Math.hypot(dx, dy) || 1;

  if (len > 0.5) {
    player.x += (dx / len) * player.speed * delta * 0.92;
    player.y += (dy / len) * player.speed * delta * 0.92;
    player.facingX = dx / len;
    player.facingY = dy / len;
  }

  player.x = clamp(player.x, 18, FIELD.width - 18);
  player.y = clamp(player.y, 18, FIELD.height - 18);
}

function updateAI(delta) {
  const homeTarget = (player, index) => {
    if (ball.owner === player) {
      return { x: player.x + player.facingX * 40, y: player.y + player.facingY * 40 };
    }

    if (ball.owner && ball.owner.team !== 'home') {
      return { x: Math.max(90, ball.x - 80 + index * 20), y: 90 + index * 120 };
    }

    if (ball.owner && ball.owner.team === 'home') {
      return { x: ball.x + (index - 2) * 35, y: ball.y + (index % 2 === 0 ? -75 : 75) };
    }

    return { x: 120 + index * 60, y: 140 + (index % 2 === 0 ? 60 : 240) };
  };

  const awayTarget = (player, index) => {
    if (ball.owner === player) {
      return { x: player.x + player.facingX * 40, y: player.y + player.facingY * 40 };
    }

    if (ball.owner && ball.owner.team !== 'away') {
      return { x: Math.min(FIELD.width - 90, ball.x + 80 - index * 10), y: 90 + index * 120 };
    }

    if (ball.owner && ball.owner.team === 'away') {
      return { x: ball.x - (index - 2) * 35, y: ball.y + (index % 2 === 0 ? -75 : 75) };
    }

    return { x: FIELD.width - 120 - index * 60, y: 140 + (index % 2 === 0 ? 60 : 240) };
  };

  homePlayers.forEach((player, index) => {
    if (player.controlled) return;
    const target = homeTarget(player, index);
    moveAIPlayer(player, target.x, target.y, delta);
  });

  awayPlayers.forEach((player, index) => {
    const target = awayTarget(player, index);
    moveAIPlayer(player, target.x, target.y, delta);
  });
}

function stealBall() {
  const allPlayers = [...homePlayers, ...awayPlayers];

  for (const player of allPlayers) {
    if (ball.owner === player) continue;
    if (distance(player, ball) < player.radius + ball.radius + 6) {
      ball.owner = player;
      ball.vx = 0;
      ball.vy = 0;
      setStatus(`${player.name} takes possession`);
      return;
    }
  }
}

function updateBall(delta) {
  if (ball.owner) {
    const owner = ball.owner;
    const offset = 22;
    ball.x = owner.x + owner.facingX * offset;
    ball.y = owner.y + owner.facingY * offset;
    ball.vx = 0;
    ball.vy = 0;
    return;
  }

  ball.x += ball.vx * delta;
  ball.y += ball.vy * delta;
  ball.vx *= 0.987;
  ball.vy *= 0.987;

  if (Math.abs(ball.vx) < 2) ball.vx = 0;
  if (Math.abs(ball.vy) < 2) ball.vy = 0;

  ball.trail.push({ x: ball.x, y: ball.y });
  if (ball.trail.length > 8) ball.trail.shift();

  goalCheck();
}

function updateMatch(delta) {
  game.timer = Math.max(0, game.timer - delta);
  syncHud();

  if (game.timer === 0) {
    setStatus('Full time');
  }

  updateControlledPlayer(delta);
  updateAI(delta);
  stealBall();
  updateBall(delta);
}

function drawCrowd() {
  const crowdColors = ['#d8f2ff', '#d9ffcc', '#fff9b1', '#fec7d5', '#d3c9ff'];

  for (let row = 0; row < 16; row += 1) {
    for (let col = 0; col < 24; col += 1) {
      const x = 24 + col * 38;
      const y = 20 + row * 18;
      ctx.fillStyle = crowdColors[(row + col) % crowdColors.length];
      ctx.fillRect(x, y, 12, 12);
    }
  }

  for (let row = 0; row < 16; row += 1) {
    for (let col = 0; col < 24; col += 1) {
      const x = 24 + col * 38;
      const y = FIELD.height - 20 - row * 18;
      ctx.fillStyle = crowdColors[(row * 2 + col) % crowdColors.length];
      ctx.fillRect(x, y, 12, 12);
    }
  }
}

function drawPitch() {
  ctx.clearRect(0, 0, FIELD.width, FIELD.height);

  const fieldGradient = ctx.createLinearGradient(0, 0, 0, FIELD.height);
  fieldGradient.addColorStop(0, '#58c16a');
  fieldGradient.addColorStop(0.5, '#3aa152');
  fieldGradient.addColorStop(1, '#2e8d46');
  ctx.fillStyle = fieldGradient;
  ctx.fillRect(0, 0, FIELD.width, FIELD.height);

  ctx.fillStyle = 'rgba(255,255,255,0.04)';
  for (let x = 0; x < FIELD.width; x += 64) {
    ctx.fillRect(x, 0, 30, FIELD.height);
  }

  drawCrowd();

  ctx.strokeStyle = 'rgba(255,255,255,0.88)';
  ctx.lineWidth = 3;
  ctx.strokeRect(14, 14, FIELD.width - 28, FIELD.height - 28);
  ctx.beginPath();
  ctx.moveTo(FIELD.cx, 14);
  ctx.lineTo(FIELD.cx, FIELD.height - 14);
  ctx.stroke();

  ctx.beginPath();
  ctx.arc(FIELD.cx, FIELD.cy, 72, 0, Math.PI * 2);
  ctx.stroke();

  ctx.beginPath();
  ctx.arc(FIELD.cx, FIELD.cy, 5, 0, Math.PI * 2);
  ctx.fillStyle = 'rgba(255,255,255,0.9)';
  ctx.fill();

  ctx.strokeRect(14, FIELD.cy - FIELD.goalHeight / 2, 28, FIELD.goalHeight);
  ctx.strokeRect(FIELD.width - 42, FIELD.cy - FIELD.goalHeight / 2, 28, FIELD.goalHeight);
  ctx.strokeRect(14, FIELD.cy - FIELD.boxHeight / 2, FIELD.boxWidth, FIELD.boxHeight);
  ctx.strokeRect(FIELD.width - FIELD.boxWidth - 14, FIELD.cy - FIELD.boxHeight / 2, FIELD.boxWidth, FIELD.boxHeight);

  ctx.beginPath();
  ctx.arc(100, FIELD.cy, 30, 0.5 * Math.PI, 1.5 * Math.PI);
  ctx.stroke();
  ctx.beginPath();
  ctx.arc(FIELD.width - 100, FIELD.cy, 30, 1.5 * Math.PI, 0.5 * Math.PI);
  ctx.stroke();
}

function drawBall() {
  for (let i = 0; i < ball.trail.length; i += 1) {
    const p = ball.trail[i];
    const alpha = (i + 1) / ball.trail.length;
    ctx.beginPath();
    ctx.fillStyle = `rgba(255,255,255,${alpha * 0.28})`;
    ctx.arc(p.x, p.y, ball.radius * (0.4 + alpha * 0.8), 0, Math.PI * 2);
    ctx.fill();
  }

  ctx.beginPath();
  ctx.shadowColor = 'rgba(255, 255, 255, 0.7)';
  ctx.shadowBlur = 18;
  ctx.fillStyle = '#f5f5f5';
  ctx.arc(ball.x, ball.y, ball.radius, 0, Math.PI * 2);
  ctx.fill();
  ctx.shadowBlur = 0;

  ctx.beginPath();
  ctx.lineWidth = 2;
  ctx.strokeStyle = '#d2d2d2';
  ctx.arc(ball.x, ball.y, ball.radius, 0, Math.PI * 2);
  ctx.stroke();
}

function drawPlayer(player) {
  ctx.save();
  ctx.beginPath();
  ctx.fillStyle = 'rgba(0,0,0,0.22)';
  ctx.ellipse(player.x, player.y + player.radius + 8, player.radius * 1.1, player.radius * 0.7, 0, 0, Math.PI * 2);
  ctx.fill();

  const grad = ctx.createLinearGradient(player.x - 10, player.y - 10, player.x + 10, player.y + 10);
  grad.addColorStop(0, player.primary);
  grad.addColorStop(1, player.secondary);

  ctx.beginPath();
  ctx.fillStyle = grad;
  ctx.arc(player.x, player.y, player.radius, 0, Math.PI * 2);
  ctx.fill();

  ctx.beginPath();
  ctx.fillStyle = '#111';
  ctx.arc(player.x, player.y, 6, 0, Math.PI * 2);
  ctx.fill();

  ctx.strokeStyle = 'rgba(255,255,255,0.6)';
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.moveTo(player.x, player.y);
  ctx.lineTo(player.x + player.facingX * 18, player.y + player.facingY * 18);
  ctx.stroke();

  ctx.font = 'bold 11px Arial';
  ctx.textAlign = 'center';
  ctx.fillStyle = '#f7f7f7';
  ctx.fillText(player.name.split(' ')[0][0] || 'P', player.x, player.y + 4);
  ctx.restore();
}

function render() {
  drawPitch();
  [...homePlayers, ...awayPlayers].forEach(drawPlayer);
  drawBall();
}

function gameLoop(timestamp) {
  const delta = Math.min((timestamp - game.lastTime) / 1000 || 0.016, 0.03);
  game.lastTime = timestamp;

  updateMatch(delta);
  render();
  requestAnimationFrame(gameLoop);
}

syncHud();
setStatus('Kickoff');
resetBall();
requestAnimationFrame(gameLoop);

