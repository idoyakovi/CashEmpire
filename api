const http = require('node:http');
const fs = require('node:fs');
const path = require('node:path');
const crypto = require('node:crypto');
const { Server } = require('socket.io');
const Redis = require('ioredis');
const { createAdapter } = require('@socket.io/redis-adapter');

const DATA_DIR = process.env.DATA_DIR || (process.env.VERCEL ? path.join('/tmp', 'cash-empire-data') : path.join(__dirname, 'data'));
const DB_FILE = path.join(DATA_DIR, 'profiles.json');
const SOCKET_PATH = '/api/socket-io/socket.io';
const WORLD = { w: 10800, h: 8100 };
const MAX_PLAYERS = 150;

fs.mkdirSync(DATA_DIR, { recursive: true });

function readProfiles() {
  try {
    const parsed = JSON.parse(fs.readFileSync(DB_FILE, 'utf8'));
    return parsed && typeof parsed === 'object' && !Array.isArray(parsed) ? parsed : {};
  } catch (e) {
    if (e.code !== 'ENOENT') console.error('Could not read profiles database:', e.message);
    return {};
  }
}
let profiles = readProfiles();
let saveTimer = null;
function saveProfiles() {
  if (process.env.REDIS_URL) return; // Vercel's filesystem is ephemeral; Redis is the persistent store.
  if (saveTimer) clearTimeout(saveTimer);
  saveTimer = setTimeout(() => {
    saveTimer = null;
    const temp = DB_FILE + '.tmp';
    try {
      fs.writeFileSync(temp, JSON.stringify(profiles), { mode: 0o600 });
      fs.renameSync(temp, DB_FILE);
    } catch (e) {
      console.error('Could not save local profiles:', e.message);
      try { fs.unlinkSync(temp); } catch {}
    }
  }, 250);
}
const redisUrl = process.env.REDIS_URL;
const redisStore = redisUrl ? new Redis(redisUrl, { maxRetriesPerRequest: 2, lazyConnect: true }) : null;
async function getProfile(key) {
  if (redisStore) {
    try {
      if (redisStore.status === 'wait') await redisStore.connect();
      const raw = await redisStore.get('ce:profile:' + key);
      if (raw) { const value = JSON.parse(raw); profiles[key] = value; return value; }
      if (profiles[key]) { await redisStore.set('ce:profile:' + key, JSON.stringify(profiles[key])); return profiles[key]; }
    } catch (e) { console.error('Redis profile read failed:', e.message); throw e; }
  }
  return profiles[key] || null;
}
async function putProfile(key, profile) {
  profiles[key] = profile;
  if (redisStore) {
    try {
      if (redisStore.status === 'wait') await redisStore.connect();
      await redisStore.set('ce:profile:' + key, JSON.stringify(profile));
    } catch (e) { console.error('Redis profile write failed:', e.message); throw e; }
  } else saveProfiles();
}
async function getAllProfiles() {
  if (!redisStore) return Object.values(profiles);
  if (redisStore.status === 'wait') await redisStore.connect();
  const keys = await redisStore.keys('ce:profile:*');
  if (!keys.length) return Object.values(profiles);
  const values = await redisStore.mget(keys);
  return values.filter(Boolean).map(raw => { try { return JSON.parse(raw); } catch { return null; } }).filter(Boolean);
}
function cleanText(value, max = 80) {
  return String(value ?? '').replace(/[<>\u0000-\u001f\u007f]/g, '').trim().slice(0, max);
}
function profileKey(first, last) {
  return (first + ' ' + last).normalize('NFKC').toLocaleLowerCase('he');
}
function passwordHash(code, salt) {
  return crypto.scryptSync(code, salt, 64).toString('hex');
}
function safeEqual(a, b) {
  if (typeof a !== 'string' || typeof b !== 'string' || a.length !== b.length) return false;
  return crypto.timingSafeEqual(Buffer.from(a), Buffer.from(b));
}
function defaultProfile(first, last) {
  return {
    display: `${first} ${last}`,
    cash: 500, bank: 0, xp: 0, level: 1, storyStep: 0,
    vehicle: 'ברגל', vehicleSpeed: 1, hasHouse: false, houseIncome: Date.now(),
    ownedCars: [], dailyRewardDay: '', creditCard: false, creditLimit: 500,
    creditDebt: 0, paymentMethod: 'cash', hasSeenIntro: false,
    skin: { shirt: '#2675c8', tone: '#f2c7a4', hair: '#3b2a24' },
    cooldowns: {}, activities: {}, achievements: [], collectibles: [], pet: '',
    homeLevel: 0, carColor: '#4e8de2', gardenAt: 0, energy: 100,
    treasure: null, delivery: null, dailyContractsDay: '', dailyContracts: 0,
    txns: [], job: null, checks: [], property: null, mortgage: null,
    items: [], hp: 100, weapons: [], equipped: null, x: 1200, y: 1015
  };
}
function publicPlayer(p) {
  return {
    id: p.id, name: p.name, x: p.x, y: p.y,
    vehicle: p.vehicle || 'ברגל', handsUp: !!p.handsUp,
    moving: !!p.moving, skin: p.skin || { shirt: '#37a3a0', tone: '#f2c7a4', hair: '#3b2a24' }
  };
}
function boundedNumber(n, fallback, min, max) {
  n = Number(n);
  return Number.isFinite(n) ? Math.max(min, Math.min(max, n)) : fallback;
}

const server = http.createServer();
const io = new Server(server, {
  path: SOCKET_PATH,
  serveClient: true,
  maxHttpBufferSize: 1e5,
  cors: { origin: true, methods: ['GET', 'POST'] },
  pingInterval: 25000,
  pingTimeout: 20000
});
let redisPub = null;
let redisSub = null;
let redisReady = Promise.resolve();
if (redisUrl) {
  redisPub = new Redis(redisUrl, { maxRetriesPerRequest: null, lazyConnect: true });
  redisSub = redisPub.duplicate({ maxRetriesPerRequest: null, lazyConnect: true });
  redisReady = Promise.all([redisPub.connect(), redisSub.connect()]).then(() => {
    io.adapter(createAdapter(redisPub, redisSub));
    console.log('Socket.IO Redis adapter connected');
  }).catch(err => { console.error('Redis adapter connection failed:', err.message); throw err; });
} else {
  console.warn('REDIS_URL is not configured: multiplayer only shares state within a single function instance, and profiles are not durable on Vercel. Configure Redis before public deployment.');
}
const players = new Map();
const limits = new Map();
let cityWeather = '☀️ בהיר';
let lastWeatherChange = 0;
const jokes = [
  'חדשות העיר: היונה ליד הבנק קיבלה קידום לסגנית מנהלת 🐦',
  'משרד התחבורה הודיע: הפקק של שני אנשים והקטנוע נפתח לתנועה 🚦',
  'הסופרמרקט מבקש: מי שלקח את העגלה לטיול בפארק, להחזיר אותה בבקשה 🛒',
  'תחזית להיום: 100% סיכוי שמישהו יגיד "אני רק מסתכל" בסוכנות 🚗',
  'דיווח מהפארק: מישהו ניסה לשלם לברווז בשטר של מונופול 🦆'
];
function rateOk(socket, key, ms) {
  const now = Date.now();
  const all = limits.get(socket.id) || {};
  if (now - (all[key] || 0) < ms) return false;
  all[key] = now;
  limits.set(socket.id, all);
  return true;
}
async function emitWorld() {
  try {
    const sockets = await io.fetchSockets();
    const list = sockets.map(s => s.data.player).filter(Boolean).map(publicPlayer);
    io.emit('world', list);
  } catch (e) { console.error('Could not build world state:', e.message); }
}
async function deliverToName(name, event, payload) {
  const normalized = String(name || '').trim().toLocaleLowerCase('he');
  if (!normalized) return false;
  const sockets = await io.in('name:' + normalized).fetchSockets();
  if (!sockets.length) return false;
  io.to('name:' + normalized).emit(event, payload);
  return true;
}

io.on('connection', async socket => {
  if (redisUrl) { try { await redisReady; } catch { socket.disconnect(true); return; } }
  socket.data.profileKey = null;
  socket.data.name = null;

  socket.on('hello', async (data = {}) => {
    if (socket.data.profileKey) return;
    if (players.size >= MAX_PLAYERS) {
      socket.emit('profile:error', 'העיר מלאה כרגע. נסה שוב בעוד כמה דקות.');
      return;
    }
    const firstName = cleanText(data.firstName, 18);
    const lastName = cleanText(data.lastName, 18);
    const code = String(data.code ?? '').trim();
    if (!firstName || !lastName || firstName.length < 1 || lastName.length < 1) {
      socket.emit('profile:error', 'נא להזין שם פרטי ושם משפחה.');
      return;
    }
    if (code.length < 4 || code.length > 64) {
      socket.emit('profile:error', 'הקוד האישי חייב להכיל 4 עד 64 תווים.');
      return;
    }
    const key = profileKey(firstName, lastName);
    let existing;
    try { existing = await getProfile(key); } catch { socket.emit('profile:error', 'שרת השמירה לא זמין כרגע. נסה שוב מאוחר יותר.'); return; }
    if (existing) {
      if (!existing.salt || !existing.codeHash || !safeEqual(passwordHash(code, existing.salt), existing.codeHash)) {
        socket.emit('profile:error', 'קוד שגוי לדמות הזאת.');
        return;
      }
    } else {
      const salt = crypto.randomBytes(16).toString('hex');
      existing = { ...defaultProfile(firstName, lastName), salt, codeHash: passwordHash(code, salt) };
      try { await putProfile(key, existing); } catch { socket.emit('profile:error', 'לא ניתן לשמור את הדמות כרגע. נסה שוב מאוחר יותר.'); return; }
    }
    const profile = existing;
    const nameRoom = 'name:' + `${firstName} ${lastName}`.toLocaleLowerCase('he');
    const oldSockets = await io.in(nameRoom).fetchSockets();
    for (const oldSocket of oldSockets) if (oldSocket.id !== socket.id) oldSocket.disconnect(true);
    socket.data.profileKey = key;
    socket.data.name = `${firstName} ${lastName}`;
    await socket.join(nameRoom);
    const x = boundedNumber(profile.x, 1200, 20, WORLD.w - 20);
    const y = boundedNumber(profile.y, 1015, 20, WORLD.h - 20);
    const player = {
      id: socket.id, name: socket.data.name, x, y, vehicle: profile.vehicle || 'ברגל',
      handsUp: false, moving: false, skin: profile.skin || defaultProfile(firstName, lastName).skin,
      lastMove: 0
    };
    players.set(socket.id, player);
    socket.data.player = player;
    const pub = { ...profile, firstName, lastName, x, y, isNew: !profile.hasSeenIntro };
    delete pub.salt; delete pub.codeHash;
    socket.emit('profile:load', pub);
    socket.emit('city:weather', cityWeather);
    await emitWorld();
    io.except(socket.id).emit('player:join', publicPlayer(player));
    console.log(`Player joined: ${socket.data.name} (${players.size} online)`);
  });

  socket.on('player:move', (data = {}) => {
    if (!socket.data.profileKey || !rateOk(socket, 'move', 45)) return;
    const p = players.get(socket.id);
    if (!p) return;
    p.x = boundedNumber(data.x, p.x, 0, WORLD.w);
    p.y = boundedNumber(data.y, p.y, 0, WORLD.h);
    p.vehicle = cleanText(data.vehicle, 24) || 'ברגל';
    p.handsUp = !!data.handsUp;
    p.moving = !!data.moving;
    if (data.skin && typeof data.skin === 'object') {
      const hex = v => /^#[0-9a-f]{6}$/i.test(String(v || '')) ? String(v) : null;
      p.skin = {
        shirt: hex(data.skin.shirt) || p.skin.shirt,
        tone: hex(data.skin.tone) || p.skin.tone,
        hair: hex(data.skin.hair) || p.skin.hair
      };
    }
    socket.data.player = p;
    io.except(socket.id).emit('player:move', publicPlayer(p));
  });

  socket.on('profile:update', async (data = {}) => {
    if (!socket.data.profileKey || !rateOk(socket, 'profile', 1000) || !data || typeof data !== 'object' || Array.isArray(data)) return;
    const profile = await getProfile(socket.data.profileKey);
    if (!profile) return;
    const allowed = [
      'cash','bank','xp','level','storyStep','vehicle','vehicleSpeed','hasHouse','houseIncome',
      'ownedCars','skin','activities','achievements','pet','homeLevel','carColor','gardenAt','energy',
      'treasure','delivery','collectibles','dailyContractsDay','dailyContracts','txns','job','checks',
      'property','mortgage','items','hp','weapons','equipped','cooldowns','creditCard','creditLimit',
      'creditDebt','paymentMethod','x','y'
    ];
    for (const key of allowed) {
      if (Object.prototype.hasOwnProperty.call(data, key)) {
        const value = data[key];
        if (JSON.stringify(value)?.length > 20000) continue;
        profile[key] = value;
      }
    }
    const p = players.get(socket.id);
    if (p) {
      p.x = boundedNumber(profile.x, p.x, 0, WORLD.w);
      p.y = boundedNumber(profile.y, p.y, 0, WORLD.h);
      profile.x = p.x; profile.y = p.y;
    }
    await putProfile(socket.data.profileKey, profile);
  });

  socket.on('story:seen', async () => {
    if (!socket.data.profileKey) return;
    const profile = await getProfile(socket.data.profileKey);
    if (!profile) return;
    profile.hasSeenIntro = true;
    await putProfile(socket.data.profileKey, profile);
  });

  socket.on('daily:claim', async () => {
    if (!socket.data.profileKey || !rateOk(socket, 'daily', 1000)) return;
    const p = await getProfile(socket.data.profileKey);
    if (!p) return;
    const today = new Date().toISOString().slice(0, 10);
    if (p.dailyRewardDay === today) {
      socket.emit('daily:result', { ok: false, message: 'המתנה היומית כבר נאספה היום.' });
      return;
    }
    p.dailyRewardDay = today;
    p.cash = (Number(p.cash) || 0) + 250;
    p.xp = (Number(p.xp) || 0) + 25;
    p.level = Math.max(1, Number(p.level) || 1);
    while (p.xp >= p.level * 100) {
      p.xp -= p.level * 100; p.level++; p.cash += 100;
    }
    await putProfile(socket.data.profileKey, p);
    socket.emit('daily:result', { ok: true, cash: p.cash, xp: p.xp, level: p.level });
  });

  socket.on('chat', raw => {
    if (!socket.data.profileKey || !rateOk(socket, 'chat', 700)) return;
    const text = cleanText(raw, 140);
    if (text) io.emit('chat', { name: socket.data.name, text });
  });
  socket.on('emote', raw => {
    if (!socket.data.profileKey || !rateOk(socket, 'emote', 300)) return;
    const text = cleanText(raw, 120);
    if (text) io.emit('emote', { name: socket.data.name, text });
  });
  socket.on('city:bulletin', raw => {
    if (!socket.data.profileKey || !rateOk(socket, 'bulletin', 30000)) return;
    const text = cleanText(raw, 80);
    if (text) io.emit('city:bulletin', { name: socket.data.name, text });
  });
  socket.on('city:weather', weather => {
    if (!socket.data.profileKey || !rateOk(socket, 'weather', 20000)) return;
    const allowed = ['☀️ בהיר','🌧️ גשם קל','🌫️ ערפל בוקר','🌙 לילה זרוע כוכבים'];
    if (allowed.includes(weather)) {
      cityWeather = weather; lastWeatherChange = Date.now();
      io.emit('city:weather', cityWeather);
    }
  });
  socket.on('leaderboard:request', async () => {
    if (!socket.data.profileKey || !rateOk(socket, 'leaderboard', 2000)) return;
    const allProfiles = await getAllProfiles();
    const list = allProfiles.map(p => ({
      name: p.display || 'תושב העיר', cash: Number(p.cash) || 0,
      bank: Number(p.bank) || 0, level: Number(p.level) || 1,
      reputation: Number(p.xp) || 0
    })).sort((a,b) => (b.cash + b.bank) - (a.cash + a.bank)).slice(0, 10);
    socket.emit('leaderboard:result', list);
  });

  socket.on('phone:message', async (data = {}) => {
    if (!socket.data.profileKey || !rateOk(socket, 'message', 500)) return;
    const to = cleanText(data.to, 40);
    const text = cleanText(data.text, 500);
    let ok = false;
    if (to && text) ok = await deliverToName(to, 'phone:message', { from: socket.data.name, to, text, time: Date.now() });
    socket.emit('phone:message:result', { ok: !!ok, to });
  });
  socket.on('phone:call', async (data = {}) => {
    if (!socket.data.profileKey || !rateOk(socket, 'call', 1000)) return;
    const to = cleanText(data.to, 40);
    const ok = to ? await deliverToName(to, 'phone:call:incoming', { from: socket.data.name }) : false;
    socket.emit('phone:call:result', { ok: !!ok, to });
  });
  socket.on('phone:call:answer', async (data = {}) => {
    if (!socket.data.profileKey || !rateOk(socket, 'answer', 500)) return;
    const to = cleanText(data.to, 40);
    const ok = await deliverToName(to, 'phone:call:answered', { from: socket.data.name, accepted: !!data.accepted });
    if (!ok) socket.emit('phone:call:result', { ok: false, to });
  });

  socket.on('disconnect', async () => {
    limits.delete(socket.id);
    const key = socket.data.profileKey;
    if (key) {
      const p = players.get(socket.id);
      if (p) {
        const profile = await getProfile(key);
        if (profile) { profile.x = p.x; profile.y = p.y; await putProfile(key, profile); }
      }
      players.delete(socket.id);
      io.emit('player:leave', socket.id);
      await emitWorld();
      console.log(`Player left: ${socket.data.name} (${players.size} online)`);
    }
  });
});

setInterval(() => {
  if (players.size) io.emit('city:joke', jokes[Math.floor(Math.random() * jokes.length)]);
}, 60000).unref();

module.exports = server;
