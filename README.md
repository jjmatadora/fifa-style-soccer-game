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
  centerX: canvas.width / 2,
  centerY: canvas.height / 2,
  goalHeight: 150,
  penaltyBoxWidth: 110,
  penaltyBoxHeight: 220,
};

const keys = {};
window.addEventListener('keydown', (event) => {
  const key = event.key.toLowerCase();
  keys[key] = true;

  if (key === ' ') {
    event.preventDefault();
  }
});

window.addEventListener('keyup', (event) => {
  keys[event.key.toLowerCase()] = false;
});

const game = {
  timer: 90,
  score: { home: 0, away: 0 },
  stateMessage: 'Kickoff',
  lastTime: 0,
};

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

function distance(a, b) {
  return Math.hypot(a.x - b.x, a.y - b.y);
}

function setStatus(text) {
  game.stateMessage = text;
  ui.status.textContent = text;
}

function syncHud() {
  ui.scoreHome.textContent = game.score.home;
  ui.scoreAway.textContent = game.score.away;

  const minutes = String(Math.floor(game.timer / 60)).padStart(2, '0');
  const seconds = String(Math.floor(game.timer % 60)).padStart(2, '0');
  ui.timer.textContent = `${minutes}:${seconds}`;
}

function createPlayer(team, name, x, y, color, controlled = false) {
  return {
    team,
    name,
    x,
    y,
    radius: 16,
    color,
    controlled,
    speed: 250,
    sprintSpeed: 330,
    facingX: 1,
    facingY: 0,
    isKeeper: false,
  };
}

const homePlayers = [
  createPlayer('home', 'A. Vega', 180, 270, '#79d2ff', true),
  createPlayer('home', 'R. Costa', 310, 150, '#79d2ff'),
  createPlayer('home', 'D. Silva', 310, 390, '#79d2ff'),
  createPlayer('home', 'M. Torres', 470, 210, '#79d2ff'),
  createPlayer('home', 'L. Ramos', 470, 330, '#79d2ff'),
];

const awayPlayers = [
  createPlayer('away', 'J. Kane', FIELD.width - 180, 270, '#ff9e9e'),
  createPlayer('away', 'N. Stone', FIELD.width - 310, 150, '#ff9e9e'),
  createPlayer('away', 'I. Lopez', FIELD.width - 310, 390, '#ff9e9e'),
  createPlayer('away', 'P. Gomez', FIELD.width - 470, 210, '#ff9e9e'),
  createPlayer('away', 'T. Brooks', FIELD.width - 470, 330, '#ff9e9e'),
];

const ball = {
  x: FIELD.centerX,
  y: FIELD.centerY,
  radius: 10,
  vx: 0,
  vy: 0,
  owner: null,
};

function resetBall(owner = null) {
  ball.x = FIELD.centerX;
  ball.y = FIELD.centerY;
  ball.vx = 0;
  ball.vy = 0;
  ball.owner = owner;
}

function nearestTeammate(player, players) {
  let best = null;
  let bestDistance = Infinity;

  for (const teammate of players) {
    if (teammate === player) continue;
    const d = distance(player, teammate);
    if (d < bestDistance) {
      bestDistance = d;
      best = teammate;
    }
  }

  return best;
}

function nearestOpponent(player, players) {
  let best = null;
  let bestDistance = Infinity;

  for (const opponent of players) {
    const d = distance(player, opponent);
    if (d < bestDistance) {
      bestDistance = d;
      best = opponent;
    }
  }

  return best;
}

function isBallInGoalArea() {
  return ball.y > FIELD.centerY - FIELD.goalHeight / 2 && ball.y < FIELD.centerY + FIELD.goalHeight / 2;
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

function handleGoalCollisions() {
  const inGoalY = isBallInGoalArea();

  if (ball.x - ball.radius <= 0 && inGoalY) {
    scoreGoal('away');
    return;
  }

  if (ball.x + ball.radius >= FIELD.width && inGoalY) {
    scoreGoal('home');
    return;
  }

  if (ball.x - ball.radius <= 0) {
    ball.x = ball.radius;
    ball.vx *= -0.9;
  }

  if (ball.x + ball.radius >= FIELD.width) {
    ball.x = FIELD.width - ball.radius;
    ball.vx *= -0.9;
  }

  if (ball.y - ball.radius <= 0) {
    ball.y = ball.radius;
    ball.vy *= -0.9;
  }

  if (ball.y + ball.radius >= FIELD.height) {
    ball.y = FIELD.height - ball.radius;
    ball.vy *= -0.9;
  }
}

function applyPlayerInput(player, delta) {
  let moveX = 0;
  let moveY = 0;

  if (keys.w) moveY -= 1;
  if (keys.s) moveY += 1;
  if (keys.a) moveX -= 1;
  if (keys.d) moveX += 1;

  if (moveX !== 0 || moveY !== 0) {
    const length = Math.hypot(moveX, moveY) || 1;
    const speed = keys.shift ? player.sprintSpeed : player.speed;
    player.x += (moveX / length) * speed * delta;
    player.y += (moveY / length) * speed * delta;
    player.facingX = moveX / length;
    player.facingY = moveY / length;
  }

  player.x = clamp(player.x, 18, FIELD.width - 18);
  player.y = clamp(player.y, 18, FIELD.height - 18);
}

function tryTakePossession() {
  const allPlayers = [...homePlayers, ...awayPlayers];

  for (const player of allPlayers) {
    if (ball.owner === player) continue;

    const d = distance(player, ball);
    const minDistance = player.radius + ball.radius + 6;

    if (d < minDistance) {
      const currentOwner = ball.owner;
      if (currentOwner && currentOwner.team !== player.team) {
        const opponent = nearestOpponent(player, allPlayers);
        if (opponent && opponent === player) {
          ball.owner = player;
          ball.vx = 0;
          ball.vy = 0;
          setStatus(`${player.name} wins the ball`);
          return;
        }
      }

      if (!currentOwner || currentOwner.team === player.team) {
        ball.owner = player;
        ball.vx = 0;
        ball.vy = 0;
        setStatus(`${player.name} receives the ball`);
      }
    }
  }
}

function shootBall(player) {
  if (ball.owner !== player) return;

  if (!keys[' ']) return;

  const targetX = player.team === 'home' ? FIELD.width : 0;
  const targetY = FIELD.centerY;

  const dx = targetX - player.x;
  const dy = targetY - player.y;
  const length = Math.hypot(dx, dy) || 1;

  ball.owner = null;
  ball.x = player.x + (dx / length) * 18;
  ball.y = player.y + (dy / length) * 18;
  ball.vx = (dx / length) * 780;
  ball.vy = (dy / length) * 780;
  setStatus(`${player.name} shoots!`);
}

function passBall(player) {
  if (ball.owner !== player) return;
  if (!keys.e) return;

  const teammates = player.team === 'home' ? homePlayers : awayPlayers;
  const target = nearestTeammate(player, teammates);

  if (!target) return;

  const dx = target.x - player.x;
  const dy = target.y - player.y;
  const length = Math.hypot(dx, dy) || 1;

  ball.owner = null;
  ball.x = player.x + (dx / length) * 18;
  ball.y = player.y + (dy / length) * 18;
  ball.vx = (dx / length) * 640;
  ball.vy = (dy / length) * 640;
  setStatus(`${player.name} passes to ${target.name}`);
}

function keepBallAttached(player) {
  if (ball.owner !== player) return;

  const offset = 20;
  const dirX = player.facingX || 1;
  const dirY = player.facingY || 0;

  ball.x = player.x + dirX * offset;
  ball.y = player.y + dirY * offset;
  ball.vx = 0;
  ball.vy = 0;
}

function updateBallPhysics(delta) {
  if (ball.owner) {
    const owner = ball.owner;
    keepBallAttached(owner);
    return;
  }

  ball.x += ball.vx * delta;
  ball.y += ball.vy * delta;

  ball.vx *= 0.99;
  ball.vy *= 0.99;

  if (Math.abs(ball.vx) < 4) ball.vx = 0;
  if (Math.abs(ball.vy) < 4) ball.vy = 0;

  handleGoalCollisions();

  const allPlayers = [...homePlayers, ...awayPlayers];
  for (const player of allPlayers) {
    const d = distance(player, ball);
    if (d < player.radius + ball.radius + 4) {
      const nx = (ball.x - player.x) / (d || 1);
      const ny = (ball.y - player.y) / (d || 1);
      ball.x = player.x + nx * (player.radius + ball.radius + 4);
      ball.y = player.y + ny * (player.radius + ball.radius + 4);
      ball.vx *= 0.3;
      ball.vy *= 0.3;
    }
  }
}

function moveAIPlayer(player, delta, targetX, targetY) {
  const dx = targetX - player.x;
  const dy = targetY - player.y;
  const length = Math.hypot(dx, dy) || 1;

  if (length > 6) {
    const speed = player.speed * 0.9;
    player.x += (dx / length) * speed * delta;
    player.y += (dy / length) * speed * delta;
    player.facingX = dx / length;
    player.facingY = dy / length;
  }

  player.x = clamp(player.x, 18, FIELD.width - 18);
  player.y = clamp(player.y, 18, FIELD.height - 18);
}

function updateAI(delta) {
  const homeTargetX = ball.x - 120;
  const awayTargetX = ball.x + 120;

  homePlayers.forEach((player, index) => {
    if (player.controlled) return;

    let targetX = 120 + index * 50;
    let targetY = 140 + (index % 2 === 0 ? 0 : 1) * 180 + (index === 0 ? 90 : 0);
    if (ball.owner && ball.owner.team === 'home') {
      targetX = ball.x + (index - 2) * 30;
      targetY = ball.y + (index % 2 === 0 ? -80 : 80);
    }

    if (ball.owner && ball.owner.team === 'away') {
      targetX = 180 + index * 60;
      targetY = 140 + (index % 2 === 0 ? 0 : 1) * 180 + (index === 0 ? 90 : 0);
    }

    if (ball.owner && ball.owner.team !== player.team) {
      targetX = player.team === 'home' ? Math.max(90, ball.x - 80) : Math.min(FIELD.width - 90, ball.x + 80);
      targetY = ball.y + (index % 2 === 0 ? -90 : 90);
    }

    if (ball.owner !== player && distance(player, ball) < 140) {
      targetX = ball.x;
      targetY = ball.y;
    }

    moveAIPlayer(player, delta, clamp(targetX, 70, FIELD.width - 70), clamp(targetY, 70, FIELD.height - 70));
  });

  awayPlayers.forEach((player, index) => {
    if (player.controlled) return;

    let targetX = FIELD.width - 120 - index * 50;
    let targetY = 140 + (index % 2 === 0 ? 0 : 1) * 180 + (index === 0 ? 90 : 0);

    if (ball.owner && ball.owner.team === 'away') {
      targetX = ball.x - (index - 2) * 30;
      targetY = ball.y + (index % 2 === 0 ? -80 : 80);
    }

    if (ball.owner && ball.owner.team === 'home') {
      targetX = FIELD.width - 180 - index * 60;
      targetY = 140 + (index % 2 === 0 ? 0 : 1) * 180 + (index === 0 ? 90 : 0);
    }

    if (ball.owner && ball.owner.team !== player.team) {
      targetX = player.team === 'home' ? Math.max(90, ball.x - 80) : Math.min(FIELD.width - 90, ball.x + 80);
      targetY = ball.y + (index % 2 === 0 ? -90 : 90);
    }

    if (ball.owner !== player && distance(player, ball) < 140) {
      targetX = ball.x;
      targetY = ball.y;
    }

    moveAIPlayer(player, delta, clamp(targetX, 70, FIELD.width - 70), clamp(targetY, 70, FIELD.height - 70));
  });
}

function updateControlledPlayer(delta) {
  const player = homePlayers[0];
  applyPlayerInput(player, delta);

  if (ball.owner === player) {
    keepBallAttached(player);
  }

  if (ball.owner === null) {
    const d = distance(player, ball);
    if (d < player.radius + ball.radius + 8) {
      ball.owner = player;
      setStatus(`${player.name} controls the ball`);
    }
  }

  shootBall(player);
  passBall(player);
}

function updateMatch(delta) {
  game.timer = Math.max(0, game.timer - delta);
  if (game.timer === 0) {
    setStatus('Full time');
  }

  updateControlledPlayer(delta);
  updateAI(delta);
  tryTakePossession();
  updateBallPhysics(delta);
  syncHud();
}

function drawPitch() {
  ctx.clearRect(0, 0, FIELD.width, FIELD.height);

  ctx.fillStyle = '#3aa152';
  ctx.fillRect(0, 0, FIELD.width, FIELD.height);

  ctx.strokeStyle = 'rgba(255,255,255,0.8)';
  ctx.lineWidth = 3;
  ctx.strokeRect(14, 14, FIELD.width - 28, FIELD.height - 28);

  ctx.beginPath();
  ctx.moveTo(FIELD.centerX, 14);
  ctx.lineTo(FIELD.centerX, FIELD.height - 14);
  ctx.stroke();

  ctx.beginPath();
  ctx.arc(FIELD.centerX, FIELD.centerY, 72, 0, Math.PI * 2);
  ctx.stroke();

  ctx.beginPath();
  ctx.arc(FIELD.centerX, FIELD.centerY, 6, 0, Math.PI * 2);
  ctx.fillStyle = 'rgba(255,255,255,0.9)';
  ctx.fill();

  ctx.strokeRect(14, FIELD.centerY - FIELD.goalHeight / 2, 30, FIELD.goalHeight);
  ctx.strokeRect(FIELD.width - 44, FIELD.centerY - FIELD.goalHeight / 2, 30, FIELD.goalHeight);

  ctx.strokeRect(14, FIELD.centerY - FIELD.penaltyBoxHeight / 2, FIELD.penaltyBoxWidth, FIELD.penaltyBoxHeight);
  ctx.strokeRect(FIELD.width - FIELD.penaltyBoxWidth - 14, FIELD.centerY - FIELD.penaltyBoxHeight / 2, FIELD.penaltyBoxWidth, FIELD.penaltyBoxHeight);
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
  ctx.textAlign = 'center';
  ctx.fillStyle = '#fff';
  ctx.fillText(player.name, player.x, player.y - 24);
}

function drawBall() {
  ctx.beginPath();
  ctx.fillStyle = '#fefefe';
  ctx.arc(ball.x, ball.y, ball.radius, 0, Math.PI * 2);
  ctx.fill();

  ctx.beginPath();
  ctx.lineWidth = 2;
  ctx.strokeStyle = '#c7c7c7';
  ctx.arc(ball.x, ball.y, ball.radius, 0, Math.PI * 2);
  ctx.stroke();
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

