const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const path = require('path');

const app = express();
const server = http.createServer(app);
const io = new Server(server, { cors: { origin: '*' } });

app.use(express.static(path.join(__dirname, 'public')));

// ---- CONSTANTS ----
const MAP_W = 4000;
const MAP_H = 3200;
const PLAYER_SPEED = 4.5;
const BULLET_SPEED = 11;
const PLAYER_RADIUS = 16;
const BULLET_RADIUS = 5;
const MAX_HP = 100;
const RESPAWN_TIME = 4000;
const HEALTHPACK_HEAL = 25;
const HEALTHPACK_RESPAWN = 15000;
const TICK = 1000 / 60;
const MAX_SHIELD = 50;
const SAFE_MARGIN = 28;

// ---- COLLISION HELPER ----
function circleRect(cx, cy, cr, rx, ry, rw, rh) {
  const nearX = Math.max(rx, Math.min(cx, rx + rw));
  const nearY = Math.max(ry, Math.min(cy, ry + rh));
  const dx = cx - nearX, dy = cy - nearY;
  return dx * dx + dy * dy < cr * cr;
}

function isSafe(x, y, radius) {
  if (x - radius < 10 || x + radius > MAP_W - 10) return false;
  if (y - radius < 10 || y + radius > MAP_H - 10) return false;
  for (const w of WALLS) {
    if (circleRect(x, y, radius, w.x, w.y, w.w, w.h)) return false;
  }
  return true;
}

// ---- WALLS ----
const WALLS = [
  // TOP ROW
  { x:100,  y:100,  w:360, h:260 },
  { x:620,  y:100,  w:300, h:220 },
  { x:1080, y:100,  w:320, h:240 },
  { x:1560, y:100,  w:280, h:200 },
  { x:2000, y:100,  w:320, h:240 },
  { x:2480, y:100,  w:300, h:220 },
  { x:2940, y:100,  w:320, h:240 },
  { x:3440, y:100,  w:360, h:260 },
  // SECOND ROW
  { x:100,  y:460,  w:240, h:320 },
  { x:500,  y:440,  w:280, h:220 },
  { x:500,  y:740,  w:200, h:180 },
  { x:920,  y:460,  w:340, h:280 },
  { x:1380, y:480,  w:70,  h:180 },
  { x:1560, y:420,  w:440, h:340 },
  { x:2140, y:480,  w:70,  h:180 },
  { x:2280, y:460,  w:340, h:280 },
  { x:2760, y:440,  w:280, h:220 },
  { x:2760, y:740,  w:200, h:180 },
  { x:3560, y:460,  w:240, h:320 },
  // UPPER-MID
  { x:100,  y:920,  w:220, h:300 },
  { x:100,  y:1300, w:220, h:260 },
  { x:480,  y:1000, w:260, h:200 },
  { x:480,  y:1280, w:220, h:220 },
  { x:800,  y:920,  w:180, h:260 },
  { x:1060, y:960,  w:300, h:220 },
  { x:1060, y:1260, w:260, h:240 },
  // Center plaza
  { x:1440, y:880,  w:520, h:44  },
  { x:1440, y:880,  w:44,  h:600 },
  { x:1916, y:880,  w:44,  h:600 },
  { x:1440, y:1436, w:520, h:44  },
  { x:1560, y:1060, w:140, h:90  },
  { x:1740, y:1160, w:120, h:90  },
  { x:2040, y:960,  w:300, h:220 },
  { x:2040, y:1260, w:260, h:240 },
  { x:2420, y:1000, w:260, h:200 },
  { x:2460, y:1280, w:220, h:220 },
  { x:2620, y:920,  w:180, h:260 },
  { x:3080, y:920,  w:240, h:300 },
  { x:3080, y:1300, w:240, h:260 },
  { x:3580, y:920,  w:220, h:300 },
  { x:3580, y:1300, w:220, h:260 },
  // CENTER HORIZONTAL
  { x:100,  y:1680, w:240, h:280 },
  { x:420,  y:1660, w:220, h:220 },
  { x:420,  y:1960, w:280, h:200 },
  { x:760,  y:1700, w:300, h:240 },
  { x:820,  y:2020, w:240, h:220 },
  { x:1160, y:1660, w:340, h:260 },
  { x:1220, y:2000, w:260, h:220 },
  { x:1640, y:1660, w:100, h:100 },
  { x:1640, y:1880, w:100, h:100 },
  { x:2160, y:1660, w:100, h:100 },
  { x:2160, y:1880, w:100, h:100 },
  { x:2260, y:1660, w:340, h:260 },
  { x:2320, y:2000, w:260, h:220 },
  { x:2740, y:1700, w:300, h:240 },
  { x:2740, y:2020, w:240, h:220 },
  { x:3160, y:1660, w:220, h:220 },
  { x:3160, y:1960, w:280, h:200 },
  { x:3560, y:1680, w:240, h:280 },
  // LOWER SECTION
  { x:100,  y:2380, w:240, h:280 },
  { x:420,  y:2360, w:220, h:220 },
  { x:420,  y:2660, w:280, h:200 },
  { x:760,  y:2400, w:300, h:240 },
  { x:820,  y:2720, w:240, h:220 },
  { x:1180, y:2360, w:340, h:260 },
  { x:1240, y:2700, w:260, h:220 },
  { x:2220, y:2360, w:340, h:260 },
  { x:2280, y:2700, w:260, h:220 },
  { x:2740, y:2400, w:300, h:240 },
  { x:2800, y:2720, w:240, h:220 },
  { x:3160, y:2360, w:220, h:220 },
  { x:3160, y:2660, w:280, h:200 },
  { x:3560, y:2380, w:240, h:280 },
  // BOTTOM ROW
  { x:100,  y:2980, w:340, h:120 },
  { x:620,  y:2960, w:280, h:140 },
  { x:1080, y:2980, w:300, h:120 },
  { x:1540, y:2960, w:300, h:140 },
  { x:2000, y:2980, w:300, h:120 },
  { x:2460, y:2960, w:280, h:140 },
  { x:2900, y:2980, w:300, h:120 },
  { x:3360, y:2960, w:280, h:140 },
  { x:3560, y:2980, w:340, h:120 },
  // STREET COVER
  { x:840,  y:340,  w:70,  h:70  },
  { x:1340, y:360,  w:70,  h:70  },
  { x:2300, y:340,  w:70,  h:70  },
  { x:2920, y:360,  w:70,  h:70  },
  { x:760,  y:1960, w:70,  h:50  },
  { x:1720, y:1960, w:70,  h:50  },
  { x:2580, y:1960, w:70,  h:50  },
  { x:340,  y:1900, w:90,  h:50  },
  { x:3370, y:1900, w:90,  h:50  },
  { x:1340, y:1060, w:90,  h:50  },
  { x:1960, y:1060, w:90,  h:50  },
  { x:900,  y:2560, w:70,  h:70  },
  { x:2400, y:2560, w:70,  h:70  },
  { x:1700, y:2400, w:70,  h:70  },
  { x:2100, y:2400, w:70,  h:70  },
];

// ---- SAFE POSITION VALIDATOR ----
function validatePositions(candidates, radius) {
  return candidates.filter(p => isSafe(p.x, p.y, radius + SAFE_MARGIN));
}

// ---- HEALTH PACK POSITIONS ----
const RAW_HEALTHPACK_POSITIONS = [
  // Top strip
  { x:400,  y:360  }, { x:780,  y:340  }, { x:1000, y:350  },
  { x:1320, y:340  }, { x:1760, y:320  }, { x:2220, y:340  },
  { x:2640, y:360  }, { x:3100, y:350  }, { x:3600, y:360  },
  // Upper streets
  { x:370,  y:760  }, { x:790,  y:700  }, { x:1280, y:760  },
  { x:1760, y:700  }, { x:2100, y:700  }, { x:2560, y:760  },
  { x:3100, y:700  }, { x:3620, y:760  },
  // Mid-upper
  { x:370,  y:1160 }, { x:700,  y:1180 }, { x:1200, y:1140 },
  { x:1700, y:1200 }, { x:2180, y:1140 }, { x:2700, y:1160 },
  { x:3420, y:1140 }, { x:3620, y:1180 }, { x:1940, y:1200 },
  // Center zone
  { x:1680, y:1560 }, { x:1940, y:1560 }, { x:2220, y:1560 },
  // Mid-lower
  { x:370,  y:1900 }, { x:700,  y:1840 }, { x:1060, y:1900 },
  { x:1760, y:1860 }, { x:2120, y:1860 }, { x:2820, y:1840 },
  { x:3200, y:1900 }, { x:3620, y:1900 },
  // Lower streets
  { x:370,  y:2560 }, { x:700,  y:2500 }, { x:1060, y:2500 },
  { x:1760, y:2500 }, { x:2100, y:2500 }, { x:2580, y:2500 },
  { x:3100, y:2560 }, { x:3620, y:2560 },
  // Bottom strip
  { x:400,  y:2900 }, { x:860,  y:2880 }, { x:1380, y:2900 },
  { x:1760, y:2880 }, { x:2120, y:2880 }, { x:2680, y:2880 },
  { x:3200, y:2900 }, { x:3600, y:2900 },
];

const HEALTHPACK_POSITIONS = validatePositions(RAW_HEALTHPACK_POSITIONS, 14);
let healthPacks = HEALTHPACK_POSITIONS.map((p, i) => ({ id: i, x: p.x, y: p.y, active: true }));

// ---- SHIELD PACK POSITIONS ----
const RAW_SHIELD_POSITIONS = [
  // Moved positions that were clipping into top buildings (560->660, 760->900)
  { x:370,  y:660  }, { x:3620, y:660  },
  { x:700,  y:1560 }, { x:3200, y:1560 },
  { x:1940, y:900  }, { x:1940, y:2380 },
  { x:370,  y:2180 }, { x:3620, y:2180 },
  { x:1200, y:2180 }, { x:2680, y:2180 },
  { x:1200, y:1560 }, { x:2680, y:1560 },
  // Extra shields so players always find one on the large map
  { x:1940, y:1560 }, { x:700,  y:2560 },
  { x:3200, y:2560 }, { x:1940, y:3000 },
];

const SHIELD_POSITIONS = validatePositions(RAW_SHIELD_POSITIONS, 14);
let shieldPacks = SHIELD_POSITIONS.map((p, i) => ({ id: i, x: p.x, y: p.y, active: true }));

// ---- BOUNCE GUN POSITIONS ----
const RAW_BOUNCE_GUN_POSITIONS = [
  { x:1940, y:660  }, // moved from 560 — was clipping top buildings
  { x:700,  y:1180 }, { x:3200, y:1180 },
  { x:1200, y:1940 }, { x:2680, y:1940 },
  { x:1940, y:2560 },
  { x:370,  y:1560 }, { x:3620, y:1560 },
  { x:700,  y:2880 }, { x:3200, y:2880 },
  // Extra guns in mid-map streets
  { x:1200, y:900  }, { x:2680, y:900  },
  { x:1940, y:2180 }, { x:700,  y:2180 },
  { x:3200, y:2180 },
];

const BOUNCE_GUN_POSITIONS = validatePositions(RAW_BOUNCE_GUN_POSITIONS, 14);
let bounceGuns = BOUNCE_GUN_POSITIONS.map((p, i) => ({ id: i, x: p.x, y: p.y, active: true, type: 'bounce' }));

// ---- PLAYER SPAWN POSITIONS ----
const RAW_SPAWNS = [
  { x:370,  y:760  }, { x:3620, y:760  },
  { x:370,  y:2380 }, { x:3620, y:2380 },
  { x:1940, y:360  }, { x:1940, y:2880 },
  { x:900,  y:1560 }, { x:2980, y:1560 },
  { x:1940, y:1560 },
  { x:700,  y:2180 }, { x:3200, y:2180 },
  { x:700,  y:940  }, { x:3200, y:940  },
];

const SPAWNS = validatePositions(RAW_SPAWNS, PLAYER_RADIUS).length > 0
  ? validatePositions(RAW_SPAWNS, PLAYER_RADIUS)
  : [{ x:200, y:200 }];

function randomSpawn() {
  for (let attempt = 0; attempt < 40; attempt++) {
    const s = SPAWNS[Math.floor(Math.random() * SPAWNS.length)];
    const x = s.x + (Math.random() - 0.5) * 60;
    const y = s.y + (Math.random() - 0.5) * 60;
    if (isSafe(x, y, PLAYER_RADIUS + 10)) return { x, y };
  }
  return { ...SPAWNS[Math.floor(Math.random() * SPAWNS.length)] };
}

console.log(`Health packs: ${healthPacks.length}/${RAW_HEALTHPACK_POSITIONS.length} validated`);
console.log(`Shield packs: ${shieldPacks.length}/${RAW_SHIELD_POSITIONS.length} validated`);
console.log(`Bounce guns:  ${bounceGuns.length}/${RAW_BOUNCE_GUN_POSITIONS.length} validated`);
console.log(`Spawn points: ${SPAWNS.length}/${RAW_SPAWNS.length} validated`);

const COLORS = ['#e85d3a','#4a9eff','#f0c040','#5cd65c','#cc66ff','#ff6699'];
const VALID_COLORS = [...COLORS];

let players = {};
let bullets = [];
let bulletId = 0;
let chatHistory = [];

io.on('connection', (socket) => {
  console.log('Connected:', socket.id);
  const spawn = randomSpawn();
  const colorIdx = Object.keys(players).length % COLORS.length;

  players[socket.id] = {
    id: socket.id,
    x: spawn.x, y: spawn.y,
    angle: 0,
    hp: MAX_HP, shield: 0,
    weapon: 'default',
    score: 0, deaths: 0,
    alive: true,
    name: 'Player',
    color: COLORS[colorIdx],
    character: 'guy',
  };

  socket.emit('init', {
    id: socket.id, players,
    walls: WALLS, healthPacks, shieldPacks, bounceGuns,
    chatHistory,
    mapSize: { w: MAP_W, h: MAP_H },
    maxHp: MAX_HP,
  });

  socket.broadcast.emit('playerJoined', players[socket.id]);

  socket.on('input', (data) => {
    const p = players[socket.id];
    if (!p || !p.alive) return;

    let vx = 0, vy = 0;
    if (data.up)    vy -= PLAYER_SPEED;
    if (data.down)  vy += PLAYER_SPEED;
    if (data.left)  vx -= PLAYER_SPEED;
    if (data.right) vx += PLAYER_SPEED;
    if (vx && vy) { vx *= 0.707; vy *= 0.707; }

    let nx = p.x + vx, ny = p.y + vy;
    for (const w of WALLS) {
      if (circleRect(nx, p.y, PLAYER_RADIUS, w.x, w.y, w.w, w.h)) nx = p.x;
      if (circleRect(p.x, ny, PLAYER_RADIUS, w.x, w.y, w.w, w.h)) ny = p.y;
    }
    p.x = Math.max(PLAYER_RADIUS, Math.min(MAP_W - PLAYER_RADIUS, nx));
    p.y = Math.max(PLAYER_RADIUS, Math.min(MAP_H - PLAYER_RADIUS, ny));
    p.angle = data.angle || 0;

    for (const hp of healthPacks) {
      if (!hp.active) continue;
      const dx = p.x - hp.x, dy = p.y - hp.y;
      if (dx*dx + dy*dy < (PLAYER_RADIUS + 16)**2 && p.hp < MAX_HP) {
        p.hp = Math.min(MAX_HP, p.hp + HEALTHPACK_HEAL);
        hp.active = false;
        io.emit('healthPackPickup', { packId: hp.id, playerId: socket.id, hp: p.hp });
        setTimeout(() => { hp.active = true; io.emit('healthPackRespawn', hp.id); }, HEALTHPACK_RESPAWN);
      }
    }

    for (const sp of shieldPacks) {
      if (!sp.active) continue;
      const dx = p.x - sp.x, dy = p.y - sp.y;
      if (dx*dx + dy*dy < (PLAYER_RADIUS + 16)**2 && p.shield < MAX_SHIELD) {
        p.shield = MAX_SHIELD;
        sp.active = false;
        io.emit('shieldPackPickup', { packId: sp.id, playerId: socket.id, shield: p.shield });
        setTimeout(() => { sp.active = true; io.emit('shieldPackRespawn', sp.id); }, 20000);
      }
    }

    if (data.pickupE) {
      for (const bg of bounceGuns) {
        if (!bg.active) continue;
        const dx = p.x - bg.x, dy = p.y - bg.y;
        if (dx*dx + dy*dy < (PLAYER_RADIUS + 28)**2) {
          const prevWeapon = p.weapon;
          p.weapon = bg.type || 'bounce';
          bg.active = false;
          io.emit('bounceGunPickup', { packId: bg.id, playerId: socket.id });
          io.to(socket.id).emit('weaponChanged', { weapon: p.weapon });
          setTimeout(() => { bg.active = true; io.emit('bounceGunRespawn', bg.id); }, 60000);
          if (prevWeapon !== 'default') {
            const droppedId = Date.now();
            bounceGuns.push({ id: droppedId, x: p.x, y: p.y, active: true, type: prevWeapon });
            io.emit('bounceGunSpawned', { id: droppedId, x: p.x, y: p.y, type: prevWeapon });
          }
          break;
        }
      }
    }
  });

  socket.on('shoot', (data) => {
    const p = players[socket.id];
    if (!p || !p.alive) return;
    const angle = data.angle;
    const isBounce = p.weapon === 'bounce';
    bullets.push({
      id: bulletId++,
      ownerId: socket.id,
      x: p.x + Math.cos(angle) * 22,
      y: p.y + Math.sin(angle) * 22,
      vx: Math.cos(angle) * BULLET_SPEED,
      vy: Math.sin(angle) * BULLET_SPEED,
      color: p.color,
      life: 260,
      bouncy: isBounce,
      bounces: 0,
    });
  });

  socket.on('setName', (name) => {
    if (players[socket.id]) {
      players[socket.id].name = String(name).slice(0, 16);
      io.emit('playerUpdated', players[socket.id]);
    }
  });

  socket.on('setColor', (color) => {
    if (players[socket.id] && VALID_COLORS.includes(color)) {
      players[socket.id].color = color;
      io.emit('playerUpdated', players[socket.id]);
    }
  });

  socket.on('setCharacter', (character) => {
    if (players[socket.id] && ['guy', 'girl'].includes(character)) {
      players[socket.id].character = character;
      io.emit('playerUpdated', players[socket.id]);
    }
  });

  socket.on('privateMessage', ({ targetName, text }) => {
    const sender = players[socket.id];
    if (!sender) return;
    const target = Object.values(players).find(p => p.name.toLowerCase() === targetName.toLowerCase());
    if (!target) {
      socket.emit('chat', { name: 'SYSTEM', color: '#888', text: `Player "${targetName}" not found.` });
      return;
    }
    io.to(target.id).emit('privateMessage', { from: sender.name, to: target.name, text: String(text).slice(0, 80), color: sender.color });
  });

  socket.on('chat', (msg) => {
    const p = players[socket.id];
    if (!p) return;
    const message = { name: p.name, color: p.color, text: String(msg).slice(0, 80), time: Date.now() };
    chatHistory.push(message);
    if (chatHistory.length > 50) chatHistory.shift();
    io.emit('chat', message);
  });

  socket.on('rtc-signal', ({ targetId, signal }) => {
    io.to(targetId).emit('rtc-signal', { fromId: socket.id, signal });
  });

  socket.on('disconnect', () => {
    delete players[socket.id];
    io.emit('playerLeft', socket.id);
  });
});

// ---- GAME LOOP ----
setInterval(() => {
  bullets = bullets.filter(b => {
    b.x += b.vx; b.y += b.vy; b.life--;
    if (b.x < 0 || b.x > MAP_W || b.y < 0 || b.y > MAP_H || b.life <= 0) return false;

    for (const w of WALLS) {
      if (circleRect(b.x, b.y, BULLET_RADIUS, w.x, w.y, w.w, w.h)) {
        if (b.bouncy && b.bounces < 3) {
          const inHoriz = b.x >= w.x && b.x <= w.x + w.w;
          if (inHoriz) b.vy *= -1; else b.vx *= -1;
          b.bounces++;
          break;
        }
        return false;
      }
    }

    for (const [id, p] of Object.entries(players)) {
      if (id === b.ownerId || !p.alive) continue;
      const dx = b.x - p.x, dy = b.y - p.y;
      if (dx*dx + dy*dy < (PLAYER_RADIUS + BULLET_RADIUS)**2) {
        if (p.shield > 0) {
          p.shield = Math.max(0, p.shield - 25);
        } else {
          p.hp -= 25;
        }
        if (p.hp <= 0) {
          p.alive = false; p.hp = 0; p.deaths++;
          if (players[b.ownerId]) players[b.ownerId].score++;
          io.emit('playerDied', { id: p.id, killerId: b.ownerId, killerName: players[b.ownerId]?.name });
          setTimeout(() => {
            if (!players[id]) return;
            const s = randomSpawn();
            Object.assign(players[id], { x: s.x, y: s.y, hp: MAX_HP, shield: 0, alive: true });
            io.emit('playerRespawned', players[id]);
          }, RESPAWN_TIME);
        } else {
          io.emit('playerHit', { id: p.id, hp: p.hp, shield: p.shield });
        }
        return false;
      }
    }
    return true;
  });

  io.emit('gameState', {
    players: Object.values(players).map(p => ({
      id: p.id, x: p.x, y: p.y, angle: p.angle,
      hp: p.hp, alive: p.alive, score: p.score,
      deaths: p.deaths, name: p.name, color: p.color,
      shield: p.shield, weapon: p.weapon, character: p.character,
    })),
    bullets: bullets.map(b => ({ id: b.id, x: b.x, y: b.y, color: b.color, bouncy: b.bouncy })),
  });
}, TICK);

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => console.log(`Urban Strike running on port ${PORT}`));
