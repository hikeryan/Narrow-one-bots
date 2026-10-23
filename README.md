const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const blueScoreEl = document.getElementById('blueScore');
const redScoreEl = document.getElementById('redScore');
const statusTextEl = document.getElementById('statusText');

const WORLD = { width: 2200, height: 1400 };
let lastTime = 0;
let score = { blue: 0, red: 0 };

const keys = {};
const mouse = { x: canvas.width / 2, y: canvas.height / 2, down: false };

const clamp = (v, min, max) => Math.min(Math.max(v, min), max);

function resizeCanvas() {
  canvas.width = window.innerWidth;
  canvas.height = window.innerHeight;
}
window.addEventListener('resize', resizeCanvas);
resizeCanvas();

window.addEventListener('keydown', (e) => {
  keys[e.key.toLowerCase()] = true;
  if (e.key === ' ') e.preventDefault();
});
window.addEventListener('keyup', (e) => {
  keys[e.key.toLowerCase()] = false;
});
canvas.addEventListener('mousemove', (e) => {
  const rect = canvas.getBoundingClientRect();
  mouse.x = e.clientX - rect.left;
  mouse.y = e.clientY - rect.top;
});
canvas.addEventListener('mousedown', () => { mouse.down = true; });
canvas.addEventListener('mouseup', () => { mouse.down = false; });
canvas.addEventListener('mouseleave', () => { mouse.down = false; });

function createBase(x, y, team) {
  return {
    x, y,
    radius: 70,
    team,
    color: team === 'blue' ? '#4ea1ff' : '#ff5d5d',
  };
}

const blueBase = createBase(180, WORLD.height / 2, 'blue');
const redBase = createBase(WORLD.width - 180, WORLD.height / 2, 'red');

const player = {
  x: WORLD.width / 2,
  y: WORLD.height / 2,
  radius: 18,
  speed: 280,
  team: 'blue',
  angle: 0,
  fireCooldown: 0,
  carryFlag: null,
  health: 100,
  ammo: 999,
};

function createBot(x, y, team, name) {
  return {
    x,
    y,
    radius: 16,
    speed: 170 + Math.random() * 40,
    team,
    angle: 0,
    fireCooldown: 0,
    carryFlag: null,
    health: 100,
    name,
    wanderDir: 0,
    state: 'patrol',
    target: null,
  };
}

const flags = {
  blue: { x: blueBase.x, y: blueBase.y, radius: 12, taken: false, team: 'blue' },
  red: { x: redBase.x, y: redBase.y, radius: 12, taken: false, team: 'red' },
};

const bots = [
  createBot(680, 700, 'red', 'Bot A'),
  createBot(900, 300, 'red', 'Bot B'),
  createBot(1500, 1100, 'blue', 'Bot C'),
  createBot(1750, 600, 'blue', 'Bot D'),
];

const projectiles = [];

function updatePlayer(dt) {
  let dx = 0;
  let dy = 0;
  if (keys.w) dy -= 1;
  if (keys.s) dy += 1;
  if (keys.a) dx -= 1;
  if (keys.d) dx += 1;

  if (dx || dy) {
    const len = Math.hypot(dx, dy) || 1;
    player.x += (dx / len) * player.speed * dt;
    player.y += (dy / len) * player.speed * dt;
  }

  player.x = clamp(player.x, 30, WORLD.width - 30);
  player.y = clamp(player.y, 30, WORLD.height - 30);

  const px = mouse.x - canvas.width / 2;
  const py = mouse.y - canvas.height / 2;
  player.angle = Math.atan2(py, px);

  if (mouse.down && player.fireCooldown <= 0) {
    fireProjectile(player, player.angle);
    player.fireCooldown = 0.18;
  }
  player.fireCooldown = Math.max(0, player.fireCooldown - dt);

  if (player.carryFlag) {
    const base = player.team === 'blue' ? blueBase : redBase;
    const dist = Math.hypot(player.x - base.x, player.y - base.y);
    if (dist < base.radius + 18) {
      score[player.team] += 1;
      blueScoreEl.textContent = score.blue;
      redScoreEl.textContent = score.red;
      player.carryFlag = null;
      const teamFlag = flags[player.team];
      teamFlag.x = base.x;
      teamFlag.y = base.y;
      teamFlag.taken = false;
      statusTextEl.textContent = `${player.team.toUpperCase()} scored!`;
    }
  }
}

function fireProjectile(entity, angle) {
  const speed = 700;
  projectiles.push({
    x: entity.x + Math.cos(angle) * (entity.radius + 10),
    y: entity.y + Math.sin(angle) * (entity.radius + 10),
    vx: Math.cos(angle) * speed,
    vy: Math.sin(angle) * speed,
    radius: 5,
    team: entity.team,
    from: entity,
    life: 1.4,
  });
}

function updateBots(dt) {
  for (const bot of bots) {
    if (bot.health <= 0) continue;

    const foeFlag = bot.team === 'blue' ? flags.red : flags.blue;
    const ownFlag = bot.team === 'blue' ? flags.blue : flags.red;
    const base = bot.team === 'blue' ? blueBase : redBase;

    if (bot.carryFlag) {
      const target = base;
      const dx = target.x - bot.x;
      const dy = target.y - bot.y;
      const len = Math.hypot(dx, dy) || 1;
      bot.x += (dx / len) * bot.speed * dt;
      bot.y += (dy / len) * bot.speed * dt;
      if (Math.hypot(bot.x - target.x, bot.y - target.y) < target.radius + 18) {
        score[bot.team] += 1;
        blueScoreEl.textContent = score.blue;
        redScoreEl.textContent = score.red;
        bot.carryFlag = null;
        ownFlag.x = base.x;
        ownFlag.y = base.y;
        ownFlag.taken = false;
      }
      continue;
    }

    if (foeFlag.taken) {
      const dx = foeFlag.x - bot.x;
      const dy = foeFlag.y - bot.y;
      const len = Math.hypot(dx, dy) || 1;
      bot.x += (dx / len) * bot.speed * dt;
      bot.y += (dy / len) * bot.speed * dt;
      if (Math.hypot(bot.x - foeFlag.x, bot.y - foeFlag.y) < 22) {
        bot.carryFlag = foeFlag.team;
        foeFlag.taken = true;
      }
    } else {
      const dx = foeFlag.x - bot.x;
      const dy = foeFlag.y - bot.y;
      const len = Math.hypot(dx, dy) || 1;
      bot.x += (dx / len) * bot.speed * 0.25 * dt;
      bot.y += (dy / len) * bot.speed * 0.25 * dt;
    }

    const distToPlayer = Math.hypot(bot.x - player.x, bot.y - player.y);
    if (distToPlayer < 300) {
      const dx = player.x - bot.x;
      const dy = player.y - bot.y;
      const ang = Math.atan2(dy, dx);
      bot.angle = ang;
      if (bot.fireCooldown <= 0 && distToPlayer < 450) {
        fireProjectile(bot, ang);
        bot.fireCooldown = 0.8 + Math.random() * 0.4;
      }
    }

    bot.fireCooldown = Math.max(0, bot.fireCooldown - dt);
  }
}

function handleFlagPickupAndDrops() {
  const allEntities = [player, ...bots.filter(b => b.health > 0)];
  for (const flag of Object.values(flags)) {
    if (flag.taken) continue;

    for (const actor of allEntities) {
      if (actor.team !== flag.team) {
        const dist = Math.hypot(actor.x - flag.x, actor.y - flag.y);
        if (dist < actor.radius + flag.radius + 6) {
          flag.taken = true;
          actor.carryFlag = flag.team;
          statusTextEl.textContent = `${actor.team.toUpperCase()} picked up the ${flag.team} flag`;
        }
      }
    }
  }
}

function updateProjectiles(dt) {
  for (let i = projectiles.length - 1; i >= 0; i--) {
    const p = projectiles[i];
    p.x += p.vx * dt;
    p.y += p.vy * dt;
    p.life -= dt;

    const hit = [player, ...bots].find((entity) => {
      if (entity.team === p.team || entity.health <= 0) return false;
      return Math.hypot(entity.x - p.x, entity.y - p.y) < entity.radius + p.radius;
    });

    if (hit) {
      hit.health -= 20;
      if (hit.health <= 0) {
        hit.x = hit.team === 'blue' ? blueBase.x : redBase.x;
        hit.y = hit.team === 'blue' ? blueBase.y : redBase.y;
        hit.health = 100;
        if (hit.carryFlag) {
          const flag = flags[hit.carryFlag];
          flag.taken = false;
          flag.x = hit.team === 'blue' ? blueBase.x : redBase.x;
          flag.y = hit.team === 'blue' ? blueBase.y : redBase.y;
          hit.carryFlag = null;
        }
      }
      projectiles.splice(i, 1);
      continue;
    }

    if (p.life <= 0 || p.x < 0 || p.y < 0 || p.x > WORLD.width || p.y > WORLD.height) {
      projectiles.splice(i, 1);
    }
  }
}

function updateCamera() {
  const camX = clamp(player.x - canvas.width / 2, 0, WORLD.width - canvas.width);
  const camY = clamp(player.y - canvas.height / 2, 0, WORLD.height - canvas.height);
  ctx.setTransform(1, 0, 0, 1, 0, 0);
  ctx.translate(-camX, -camY);
}

function drawMap() {
  ctx.fillStyle = '#122234';
  ctx.fillRect(0, 0, WORLD.width, WORLD.height);

  for (let x = 0; x <= WORLD.width; x += 100) {
    ctx.strokeStyle = 'rgba(255,255,255,0.06)';
    ctx.beginPath();
    ctx.moveTo(x, 0);
    ctx.lineTo(x, WORLD.height);
    ctx.stroke();
  }
  for (let y = 0; y <= WORLD.height; y += 100) {
    ctx.strokeStyle = 'rgba(255,255,255,0.06)';
    ctx.beginPath();
    ctx.moveTo(0, y);
    ctx.lineTo(WORLD.width, y);
    ctx.stroke();
  }

  ctx.fillStyle = 'rgba(255,255,255,0.04)';
  ctx.fillRect(0, 0, WORLD.width, WORLD.height);

  // bases
  drawBase(blueBase);
  drawBase(redBase);

  // flags
  drawFlag(flags.blue, 'blue');
  drawFlag(flags.red, 'red');
}

function drawBase(base) {
  ctx.beginPath();
  ctx.fillStyle = base.color;
  ctx.arc(base.x, base.y, base.radius, 0, Math.PI * 2);
  ctx.fill();
  ctx.strokeStyle = 'rgba(255,255,255,0.7)';
  ctx.lineWidth = 4;
  ctx.stroke();
}

function drawFlag(flag, team) {
  ctx.fillStyle = team === 'blue' ? '#4ea1ff' : '#ff5d5d';
  if (flag.taken) return;
  ctx.beginPath();
  ctx.arc(flag.x, flag.y, flag.radius, 0, Math.PI * 2);
  ctx.fill();
}

function drawEntity(entity, color) {
  ctx.beginPath();
  ctx.fillStyle = color;
  ctx.arc(entity.x, entity.y, entity.radius, 0, Math.PI * 2);
  ctx.fill();

  ctx.beginPath();
  ctx.moveTo(entity.x, entity.y);
  ctx.lineTo(
    entity.x + Math.cos(entity.angle) * (entity.radius + 12),
    entity.y + Math.sin(entity.angle) * (entity.radius + 12)
  );
  ctx.strokeStyle = '#fff';
  ctx.lineWidth = 2;
  ctx.stroke();

  ctx.font = '12px sans-serif';
  ctx.fillStyle = '#fff';
  ctx.fillText(entity.name || 'Player', entity.x - 18, entity.y - 22);

  if (entity.carryFlag) {
    ctx.fillStyle = entity.carryFlag === 'blue' ? '#4ea1ff' : '#ff5d5d';
    ctx.fillRect(entity.x + 12, entity.y - 4, 18, 8);
  }
}

function drawProjectiles() {
  for (const p of projectiles) {
    ctx.beginPath();
    ctx.fillStyle = p.team === 'blue' ? '#60a5fa' : '#f87171';
    ctx.arc(p.x, p.y, p.radius, 0, Math.PI * 2);
    ctx.fill();
  }
}

function drawHudBars() {
  const drawBar = (x, y, w, h, value, color) => {
    ctx.fillStyle = 'rgba(255,255,255,0.2)';
    ctx.fillRect(x, y, w, h);
    ctx.fillStyle = color;
    ctx.fillRect(x, y, w * (value / 100), h);
  };

  drawBar(30, 30, 180, 18, player.health, '#60a5fa');
  drawBar(canvas.width - 210, 30, 180, 18, 100, '#f87171');
}

function render() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  updateCamera();
  drawMap();
  drawProjectiles();

  drawEntity(player, '#60a5fa');
  for (const bot of bots) {
    if (bot.health > 0) drawEntity(bot, bot.team === 'blue' ? '#60a5fa' : '#f87171');
  }

  // reset transform for HUD overlays
  ctx.setTransform(1, 0, 0, 1, 0, 0);
  drawHudBars();
}

function loop(ts) {
  const dt = Math.min((ts - lastTime) / 1000 || 0.016, 0.033);
  lastTime = ts;

  updatePlayer(dt);
  updateBots(dt);
  handleFlagPickupAndDrops();
  updateProjectiles(dt);
  render();

  requestAnimationFrame(loop);
}

requestAnimationFrame(loop);

statusTextEl.textContent = 'Bot match active';
blueScoreEl.textContent = '0';
redScoreEl.textContent = '0';

const resetFlags = () => {
  flags.blue.x = blueBase.x;
  flags.blue.y = blueBase.y;
  flags.red.x = redBase.x;
  flags.red.y = redBase.y;
  flags.blue.taken = false;
  flags.red.taken = false;
};

setInterval(() => {
  if (score.blue >= 5 || score.red >= 5) {
    score.blue = 0;
    score.red = 0;
    blueScoreEl.textContent = '0';
    redScoreEl.textContent = '0';
    resetFlags();
    statusTextEl.textContent = 'New round';
  }
}, 1000);

window.addEventListener('contextmenu', (e) => e.preventDefault());


