const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

const menuScreen = document.getElementById('menuScreen');
const gameShell = document.getElementById('gameShell');
const startMatchBtn = document.getElementById('startMatchBtn');
const homeTeamPicker = document.getElementById('homeTeamPicker');
const awayTeamPicker = document.getElementById('awayTeamPicker');
const homeTeamName = document.getElementById('homeTeamName');
const awayTeamName = document.getElementById('awayTeamName');

const ui = {
  scoreHome: document.getElementById('score-home'),
  scoreAway: document.getElementById('score-away'),
  timer: document.getElementById('timer'),
  status: document.getElementById('status'),
  homeLabel: document.getElementById('homeLabel'),
  awayLabel: document.getElementById('awayLabel'),
  possession: document.getElementById('possessionValue'),
  press: document.getElementById('pressValue'),
  stamina: document.getElementById('staminaValue'),
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

const teams = [
  { name: 'Manchester City', primary: '#6ec6ff', secondary: '#1d50a1' },
  { name: 'Real Madrid', primary: '#f6d365', secondary: '#d82131' },
  { name: 'Barcelona', primary: '#7bc1ff', secondary: '#a63a3a' },
  { name: 'Bayern', primary: '#d9d9d9', secondary: '#d71920' },
  { name: 'PSG', primary: '#3b82f6', secondary: '#1f3d8a' },
  { name: 'Juventus', primary: '#d3d5d9', secondary: '#111827' },
  { name: 'Chelsea', primary: '#4c7ef3', secondary: '#0b2c68' },
  { name: 'Inter', primary: '#d7ce7d', secondary: '#20432d' },
];

const state = {
  selectedHome: 0,
  selectedAway: 1,
  matchRunning: false,
  timer: 90,
  lastTime: 0,
  score: { home: 0, away: 0 },
};

const keys = {};
window.addEventListener('keydown', (event) => {
  const key = event.key.toLowerCase();
  keys[key] = true;
  if (key === ' ') event.preventDefault();
  if (key === 'p') resetMatch();
});
window.addEventListener('keyup', (event) => {
  keys[event.key.toLowerCase()] = false;
});

function clamp(value, min, max) {
  return Math.min(Math.max(value, min), max);
}

function distance(a, b) {
  return Math.hypot(a.x - b.x, a.y - b.y);
}

function updateTeamLabels() {
  const home = teams[state.selectedHome];
  const away = teams[state.selectedAway];
  homeTeamName.textContent = home.name;
  awayTeamName.textContent = away.name;
  ui.homeLabel.textContent = home.name;
  ui.awayLabel.textContent = away.name;
}

function renderTeamButtons() {
  homeTeamPicker.innerHTML = '';
  awayTeamPicker.innerHTML = '';

  teams.forEach((team, index) => {
    const homeBtn = document.createElement('button');
    homeBtn.className = `team-option ${index === state.selectedHome ? 'active' : ''}`;
    homeBtn.style.background = `linear-gradient(135deg, ${team.primary}, ${team.secondary})`;
    homeBtn.textContent = team.name;
    homeBtn.addEventListener('click', () => {
      state.selectedHome = index;
      renderTeamButtons();
      updateTeamLabels();
    });
    homeTeamPicker.appendChild(homeBtn);

    const awayBtn = document.createElement('button');
    awayBtn.className = `team-option ${index === state.selectedAway ? 'active' : ''}`;
    awayBtn.style.background = `linear-gradient(135deg, ${team.primary}, ${team.secondary})`;
    awayBtn.textContent = team.name;
    awayBtn.addEventListener('click', () => {
      state.selectedAway = index;
      renderTeamButtons();
      updateTeamLabels();
    });
    awayTeamPicker.appendChild(awayBtn);
  });
}

function buildPlayers() {
  const home = teams[state.selectedHome];
  const away = teams[state.selectedAway];

  const homePlayers = [
    createPlayer('home', 'Vega', 180, 270, home.primary, home.secondary, true),
    createPlayer('home', 'Costa', 330, 160, home.primary, home.secondary),
    createPlayer('home', 'Silva', 330, 400, home.primary, home.secondary),
    createPlayer('home', 'Torres', 470, 220, home.primary, home.secondary),
    createPlayer('home', 'Ramos', 470, 340, home.primary, home.secondary),
  ];

  const awayPlayers = [
    createPlayer('away', 'Kane', FIELD.width - 180, 270, away.primary, away.secondary),
    createPlayer('away', 'Stone', FIELD.width - 330, 160, away.primary, away.secondary),
    createPlayer('away', 'Lopez', FIELD.width - 330, 400, away.primary, away.secondary),
    createPlayer('away', 'Gomez', FIELD.width - 470, 220, away.primary, away.secondary),
    createPlayer('away', 'Brooks', FIELD.width - 470, 340, away.primary, away.secondary),
  ];

  return { homePlayers, awayPlayers };
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

const ball = {
  x: FIELD.cx,
  y: FIELD.cy,
  radius: 10,
  vx: 0,
  vy: 0,
  owner: null,
  trail: [],
};

let homePlayers = [];
let awayPlayers = [];

function resetBall() {
  ball.x = FIELD.cx;
  ball.y = FIELD.cy;
  ball.vx = 0;
  ball.vy = 0;
  ball.owner = null;
  ball.trail = [];
}

function syncHud() {
  ui.scoreHome.textContent = state.score.home;
  ui.scoreAway.textContent = state.score.away;

  const minutes = String(Math.floor(state.timer / 60)).padStart(2, '0');
  const seconds = String(Math.floor(state.timer % 60)).padStart(2, '0');
  ui.timer.textContent = `${minutes}:${seconds}`;

  const possession = Math.min(100, Math.max(0, 50 + (Math.random() * 20 - 10)));
  ui.possession.textContent = `${Math.round(possession)}%`;
  ui.press.textContent = possession > 55 ? 'High' : 'Balanced';
  ui.stamina.textContent = `${Math.round(100 - (90 - state.timer) * 0.8)}%`;
}

function setStatus(text) {
  ui.status.textContent = text;
}

function resetMatch() {
  const teamsData = buildPlayers();
  homePlayers = teamsData.homePlayers;
  awayPlayers = teamsData.awayPlayers;
  state.score.home = 0;
  state.score.away = 0;
  state.timer = 90;
  setStatus('Kickoff');
  resetBall();
  syncHud();
}

function startMatch() {
  menuScreen.classList.add('hidden');
  gameShell.classList.remove('hidden');
  state.matchRunning = true;
  resetMatch();
}

function scoreGoal(team) {
  if (team === 'home') {
    state.score.home += 1;
    setStatus('Home score!');
  } else {
    state.score.away += 1;
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

function shootBall(player) {
  if (ball.owner !== player) return;
  if (!keys[' ']) return;

  const targetX = player.team === 'home' ? FIELD.width : 0;
  const targetY = FIELD.cy;
  const dx = targetX - player.x;
  const dy = targetY - player.y;
  const length = Math.hypot(dx, dy) || 1;

  ball.owner = null;
  ball.vx = (dx / length) * 820;
  ball.vy = (dy / length) * 820;
  setStatus(`${player.name} shoots!`);
}

function passBall(player) {
  if (ball.owner !== player) return;
  if (!keys.e) return;

  const teammates = (player.team === 'home' ? homePlayers : awayPlayers).filter((p) => p !== player);
  const target = teammates.reduce((best, current) => {
    if (!best) return current;
    return distance(player, current) < distance(player, best) ? current : best;
  }, null);

  if (!target) return;

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

function updateControlledPlayer(delta) {
  const player = homePlayers[0];
  applyInput(player, delta);

  if (ball.owner === player) {
    const offset = 22;
    ball.x = player.x + player.facingX * offset;
    ball.y = player.y + player.facingY * offset;
    ball.vx = 0;
    ball.vy = 0;
  }

  if (ball.owner === null && distance(player, ball) < player.radius + ball.radius + 8) {
    ball.owner = player;
    setStatus(`${player.name} has the ball`);
  }

  shootBall(player);
  passBall(player);
}

function moveAIPlayer(player, targetX, targetY, delta) {
  const dx = targetX - player.x;
  const dy = targetY - player.y;
  const len = Math.hypot(dx, dy) || 1;

  if (len > 1) {
    player.x += (dx / len) * player.speed * delta * 0.9;
    player.y += (dy / len) * player.speed * delta * 0.9;
    player.facingX = dx / len;
    player.facingY = dy / len;
  }

  player.x = clamp(player.x, 18, FIELD.width - 18);
  player.y = clamp(player.y, 18, FIELD.height - 18);
}

function updateAI(delta) {
  homePlayers.forEach((player, index) => {
    if (player.controlled) return;
    let targetX = 120 + index * 65;
    let targetY = 140 + (index % 2 === 0 ? 0 : 200);

    if (ball.owner && ball.owner.team === 'home') {
      targetX = ball.x + (index - 2) * 26;
      targetY = ball.y + (index % 2 === 0 ? -80 : 80);
    } else if (ball.owner && ball.owner.team !== 'home') {
      targetX = Math.max(70, ball.x - 80 + index * 18);
      targetY = 80 + index * 95;
    } else if (distance(player, ball) < 150) {
      targetX = ball.x;
      targetY = ball.y;
    }

    moveAIPlayer(player, clamp(targetX, 70, FIELD.width - 70), clamp(targetY, 70, FIELD.height - 70), delta);
  });

  awayPlayers.forEach((player, index) => {
    let targetX = FIELD.width - 120 - index * 65;
    let targetY = 140 + (index % 2 === 0 ? 0 : 200);

    if (ball.owner && ball.owner.team === 'away') {
      targetX = ball.x - (index - 2) * 26;
      targetY = ball.y + (index % 2 === 0 ? -80 : 80);
    } else if (ball.owner && ball.owner.team !== 'away') {
      targetX = Math.min(FIELD.width - 70, ball.x + 80 - index * 18);
      targetY = 80 + index * 95;
    } else if (distance(player, ball) < 150) {
      targetX = ball.x;
      targetY = ball.y;
    }

    moveAIPlayer(player, clamp(targetX, 70, FIELD.width - 70), clamp(targetY, 70, FIELD.height - 70), delta);
  });
}

function stealBall() {
  const allPlayers = [...homePlayers, ...awayPlayers];
  for (const player of allPlayers) {
    if (ball.owner === player) continue;
    if (distance(player, ball) < player.radius + ball.radius + 8) {
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
  ball.vx *= 0.988;
  ball.vy *= 0.988;

  if (Math.abs(ball.vx) < 2) ball.vx = 0;
  if (Math.abs(ball.vy) < 2) ball.vy = 0;

  ball.trail.push({ x: ball.x, y: ball.y });
  if (ball.trail.length > 8) ball.trail.shift();

  goalCheck();
}

function updateMatch(delta) {
  if (!state.matchRunning) return;

  state.timer = Math.max(0, state.timer - delta);
  if (state.timer === 0) {
    setStatus('Full time');
  }

  updateControlledPlayer(delta);
  updateAI(delta);
  stealBall();
  updateBall(delta);
  syncHud();
}

function drawCrowd() {
  const colors = ['#d8f4ff', '#dff7d9', '#fff8bf', '#ffdbec', '#d8d1ff'];

  for (let row = 0; row < 16; row += 1) {
    for (let col = 0; col < 24; col += 1) {
      const x = 25 + col * 38;
      const y = 16 + row * 18;
      const idx = (row + col) % colors.length;
      ctx.fillStyle = colors[idx];
      ctx.fillRect(x, y, 12, 12);

      const x2 = 25 + col * 38;
      const y2 = FIELD.height - 20 - row * 18;
      ctx.fillStyle = colors[(row * 3 + col) % colors.length];
      ctx.fillRect(x2, y2, 12, 12);
    }
  }
}

function drawPitch() {
  ctx.clearRect(0, 0, FIELD.width, FIELD.height);

  const fieldGrad = ctx.createLinearGradient(0, 0, 0, FIELD.height);
  fieldGrad.addColorStop(0, '#5ecf70');
  fieldGrad.addColorStop(0.5, '#3fa55a');
  fieldGrad.addColorStop(1, '#2c8d4a');
  ctx.fillStyle = fieldGrad;
  ctx.fillRect(0, 0, FIELD.width, FIELD.height);

  drawCrowd();

  ctx.fillStyle = 'rgba(255,255,255,0.04)';
  for (let x = 0; x < FIELD.width; x += 64) {
    ctx.fillRect(x, 0, 32, FIELD.height);
  }

  ctx.strokeStyle = 'rgba(255,255,255,0.85)';
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
  ctx.arc(FIELD.cx, FIELD.cy, 6, 0, Math.PI * 2);
  ctx.fillStyle = 'rgba(255,255,255,0.9)';
  ctx.fill();

  ctx.strokeRect(14, FIELD.cy - FIELD.goalHeight / 2, 28, FIELD.goalHeight);
  ctx.strokeRect(FIELD.width - 42, FIELD.cy - FIELD.goalHeight / 2, 28, FIELD.goalHeight);
  ctx.strokeRect(14, FIELD.cy - FIELD.boxHeight / 2, FIELD.boxWidth, FIELD.boxHeight);
  ctx.strokeRect(FIELD.width - FIELD.boxWidth - 14, FIELD.cy - FIELD.boxHeight / 2, FIELD.boxWidth, FIELD.boxHeight);

  ctx.beginPath();
  ctx.arc(100, FIELD.cy, 30, Math.PI / 2, 1.5 * Math.PI, false);
  ctx.stroke();
  ctx.beginPath();
  ctx.arc(FIELD.width - 100, FIELD.cy, 30, 1.5 * Math.PI, Math.PI / 2, false);
  ctx.stroke();
}

function drawBall() {
  for (let i = 0; i < ball.trail.length; i += 1) {
    const p = ball.trail[i];
    const alpha = (i + 1) / ball.trail.length;
    ctx.beginPath();
    ctx.fillStyle = `rgba(255,255,255,${alpha * 0.3})`;
    ctx.arc(p.x, p.y, ball.radius * (0.4 + alpha * 0.7), 0, Math.PI * 2);
    ctx.fill();
  }

  ctx.beginPath();
  ctx.shadowColor = 'rgba(255,255,255,0.9)';
  ctx.shadowBlur = 18;
  ctx.fillStyle = '#f5f5f5';
  ctx.arc(ball.x, ball.y, ball.radius, 0, Math.PI * 2);
  ctx.fill();
  ctx.shadowBlur = 0;

  ctx.beginPath();
  ctx.lineWidth = 2;
  ctx.strokeStyle = '#d5d5d5';
  ctx.arc(ball.x, ball.y, ball.radius, 0, Math.PI * 2);
  ctx.stroke();
}

function drawPlayer(player) {
  ctx.save();

  ctx.beginPath();
  ctx.fillStyle = 'rgba(0,0,0,0.24)';
  ctx.ellipse(player.x, player.y + player.radius + 8, player.radius * 1.15, player.radius * 0.7, 0, 0, Math.PI * 2);
  ctx.fill();

  const grad = ctx.createLinearGradient(player.x - 10, player.y - 10, player.x + 18, player.y + 18);
  grad.addColorStop(0, player.primary);
  grad.addColorStop(1, player.secondary);

  ctx.beginPath();
  ctx.fillStyle = grad;
  ctx.arc(player.x, player.y, player.radius, 0, Math.PI * 2);
  ctx.fill();

  ctx.beginPath();
  ctx.fillStyle = '#111';
  ctx.arc(player.x, player.y, 5, 0, Math.PI * 2);
  ctx.fill();

  ctx.strokeStyle = 'rgba(255,255,255,0.7)';
  ctx.lineWidth = 2;
  ctx.beginPath();
  ctx.moveTo(player.x, player.y);
  ctx.lineTo(player.x + player.facingX * 18, player.y + player.facingY * 18);
  ctx.stroke();

  ctx.font = 'bold 11px Arial';
  ctx.textAlign = 'center';
  ctx.fillStyle = '#fff';
  ctx.fillText(player.name.slice(0, 1), player.x, player.y + 4);

  ctx.restore();
}

function render() {
  drawPitch();
  [...homePlayers, ...awayPlayers].forEach(drawPlayer);
  drawBall();
}

function gameLoop(timestamp) {
  const delta = Math.min((timestamp - state.lastTime) / 1000 || 0.016, 0.03);
  state.lastTime = timestamp;

  updateMatch(delta);
  render();
  requestAnimationFrame(gameLoop);
}

startMatchBtn.addEventListener('click', startMatch);
renderTeamButtons();
updateTeamLabels();
resetMatch();
requestAnimationFrame(gameLoop);
