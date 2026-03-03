const LEVELS = [
  { name:'Pulse',   sub:'Steady beat',        interval:500, pattern:[1,0,1,0,1,0,1,0], hint:'Tap every other step',           listenLoops:2, playLoops:4 },
  { name:'Skip',    sub:'The missing beat',   interval:450, pattern:[1,0,1,0,0,0,1,0], hint:'Watch for the gap!',             listenLoops:2, playLoops:4 },
  { name:'Double',  sub:'Two in a row',       interval:400, pattern:[1,1,0,1,1,0,1,0], hint:'Quick double taps',              listenLoops:2, playLoops:4 },
  { name:'Offbeat', sub:'Land in the spaces', interval:380, pattern:[0,1,0,1,0,1,0,1], hint:'Tap between the beats',          listenLoops:2, playLoops:4 },
  { name:'Synco',   sub:'Shifted rhythm',     interval:350, pattern:[1,0,0,1,0,1,1,0], hint:'Feel the syncopation',           listenLoops:3, playLoops:4 },
  { name:'Dense',   sub:'Packed pattern',     interval:320, pattern:[1,1,0,1,0,1,1,0], hint:'Stay sharp!',                    listenLoops:3, playLoops:4 },
  { name:'Rush',    sub:'16-step sequence',   interval:280, pattern:[1,0,1,0,0,1,0,1,1,0,0,1,0,1,0,0], hint:'Twice the steps, twice the fun', listenLoops:3, playLoops:4 },
  { name:'Master',  sub:'Full speed ahead',   interval:220, pattern:[1,0,1,1,0,1,0,0,1,1,0,1,0,0,1,0], hint:'The ultimate challenge',         listenLoops:3, playLoops:4 },
];

const SAVE_KEY = 'beat-tap-v1';
function loadSave() { try { return JSON.parse(localStorage.getItem(SAVE_KEY)) || {}; } catch { return {}; } }
function writeSave() { localStorage.setItem(SAVE_KEY, JSON.stringify(save)); }
const save = loadSave();
if (!save.scores)   save.scores   = {};
if (!save.unlocked) save.unlocked = [0];
writeSave();

let audioCtx = null;
function initAudio() {
  if (audioCtx) return;
  try { audioCtx = new (window.AudioContext || window.webkitAudioContext)(); } catch(e) {}
}
function playClick(isTap) {
  if (!audioCtx || audioCtx.state === 'suspended') return;
  try {
    const osc = audioCtx.createOscillator(), gain = audioCtx.createGain();
    osc.connect(gain); gain.connect(audioCtx.destination);
    osc.frequency.value = isTap ? 1000 : 380;
    const t = audioCtx.currentTime;
    gain.gain.setValueAtTime(isTap ? 0.28 : 0.07, t);
    gain.gain.exponentialRampToValueAtTime(0.001, t + 0.08);
    osc.start(t); osc.stop(t + 0.09);
  } catch(e) {}
}
document.addEventListener('visibilitychange', () => {
  if (!document.hidden && audioCtx && audioCtx.state === 'suspended') audioCtx.resume().catch(() => {});
});

const gs = {
  levelIdx:0, phase:'idle', startTime:0,
  lastAbsStep:-1, currentLoop:-1,
  expectedTaps:[], extraTaps:0,
  rafId:null, readyTimer:null, prevActiveEl:null,
};

const $ = id => document.getElementById(id);
const screens = { home:$('s-home'), game:$('s-game'), results:$('s-results') };
function showScreen(name) { Object.values(screens).forEach(s => s.classList.remove('active')); screens[name].classList.add('active'); }

function renderHome() {
  const list = $('level-list');
  list.innerHTML = '';
  LEVELS.forEach((lv, i) => {
    const locked = !save.unlocked.includes(i);
    const score  = save.scores[i];
    const stars  = score == null ? '' : score >= 90 ? '★★★' : score >= 70 ? '★★☆' : score >= 50 ? '★☆☆' : '☆☆☆';
    const card = document.createElement('div');
    card.className = 'level-card' + (locked ? ' locked' : '');
    card.innerHTML = `
      <div class="lc-num">${i+1}</div>
      <div class="lc-info"><div class="lc-name">${lv.name}</div><div class="lc-sub">${lv.sub}</div></div>
      <div class="lc-right">${locked ? '<span style="font-size:1.1rem">🔒</span>' : score != null ? `<div class="lc-score">${score}%</div><div class="lc-stars">${stars}</div>` : '<span class="lc-stars" style="color:var(--border)">— —</span>'}</div>
    `;
    if (!locked) card.addEventListener('click', () => startLevel(i));
    list.appendChild(card);
  });
}

function buildBeatGrid(lv) {
  const grid = $('beat-grid');
  grid.innerHTML = ''; gs.prevActiveEl = null;
  grid.setAttribute('data-cols', lv.pattern.length <= 8 ? 4 : 8);
  lv.pattern.forEach((isTap, i) => {
    const cell = document.createElement('div');
    cell.className = 'beat-cell' + (isTap ? ' tap-beat' : '');
    <cell.id> = `bc-${i}`;
    grid.appendChild(cell);
  });
}

function startLevel(idx) {
  initAudio();
  if (audioCtx && audioCtx.state === 'suspended') audioCtx.resume().catch(() => {});
  gs.levelIdx = idx;
  buildBeatGrid(LEVELS[idx]);
  $('lv-name').textContent = `${idx+1}. ${LEVELS[idx].name}`;
  showScreen('game');
  startListen();
}

function startListen() {
  cancelAnimationFrame(gs.rafId); clearTimeout(gs.readyTimer);
  gs.phase = 'listen'; gs.lastAbsStep = -1; gs.currentLoop = -1;
  gs.startTime = performance.now();
  $('phase-tag').textContent = 'LISTEN'; $('phase-tag').className = 'phase-tag';
  $('game-message').textContent = 'Watch the pattern…';
  $('btn-tap').disabled = true;
  $('feedback-popup').textContent = ''; $('loop-counter').textContent = '';
  gs.rafId = requestAnimationFrame(gameLoop);
}

function gameLoop() {
  const lv = LEVELS[gs.levelIdx];
  const elapsed = performance.now() - gs.startTime;
  const patLen = lv.pattern.length, loopDur = patLen * lv.interval;
  const absStep = Math.floor(elapsed / lv.interval);
  const stepInLoop = absStep % patLen;
  const loopsDone = Math.floor(elapsed / loopDur);

  if (absStep !== gs.lastAbsStep) {
    gs.lastAbsStep = absStep;
    if (gs.prevActiveEl) gs.prevActiveEl.classList.remove('active');
    const el = $(`bc-${stepInLoop}`);
    if (el) { el.classList.add('active'); gs.prevActiveEl = el; }
    if (gs.currentLoop !== loopsDone) {
      gs.currentLoop = loopsDone;
      const total = gs.phase === 'listen' ? lv.listenLoops : lv.playLoops;
      $('loop-counter').textContent = `Loop ${Math.min(loopsDone+1, total)} / ${total}`;
    }
    playClick(lv.pattern[stepInLoop] === 1);
    if (gs.phase === 'listen' && loopsDone >= lv.listenLoops) { startGetReady(); return; }
    if (gs.phase === 'play'   && loopsDone >= lv.playLoops)   { endPlay();       return; }
  }
  gs.rafId = requestAnimationFrame(gameLoop);
}

function startGetReady() {
  gs.phase = 'ready';
  if (gs.prevActiveEl) { gs.prevActiveEl.classList.remove('active'); gs.prevActiveEl = null; }
  $('phase-tag').textContent = 'READY'; $('phase-tag').className = 'phase-tag ready';
  $('game-message').textContent = 'Get ready…'; $('loop-counter').textContent = '';
  $('btn-tap').disabled = false;
  gs.readyTimer = setTimeout(startPlay, LEVELS[gs.levelIdx].interval * 2);
}

function startPlay() {
  const lv = LEVELS[gs.levelIdx];
  gs.phase = 'play'; gs.lastAbsStep = -1; gs.currentLoop = -1;
  gs.extraTaps = 0; gs.startTime = performance.now();
  gs.expectedTaps = [];
  const loopDur = lv.pattern.length * lv.interval;
  for (let loop = 0; loop < lv.playLoops; loop++) {
    lv.pattern.forEach((isTap, step) => {
      if (isTap) gs.expectedTaps.push({ time: gs.startTime + loop*loopDur + step*lv.interval, matched:false, score:null, grade:null });
    });
  }
  $('phase-tag').textContent = 'PLAY'; $('phase-tag').className = 'phase-tag play';
  $('game-message').textContent = lv.hint;
  gs.rafId = requestAnimationFrame(gameLoop);
}

function endPlay() {
  cancelAnimationFrame(gs.rafId); gs.phase = 'done';
  if (gs.prevActiveEl) { gs.prevActiveEl.classList.remove('active'); gs.prevActiveEl = null; }
  gs.expectedTaps.forEach(e => { if (!e.matched) { e.score = 0; e.grade = 'miss'; } });
  const result = computeScore();
  const idx = gs.levelIdx;
  if (save.scores[idx] == null || result.accuracy > save.scores[idx]) save.scores[idx] = result.accuracy;
  if (result.accuracy >= 80 && idx+1 < LEVELS.length && !save.unlocked.includes(idx+1)) save.unlocked.push(idx+1);
  writeSave();
  showResults(result);
}

function handleTap() {
  if (gs.phase !== 'play') return;
  const now = performance.now(), lv = LEVELS[gs.levelIdx];
  const okWindow = lv.interval * 0.38;
  let nearest = null, nearestDist = Infinity;
  for (const exp of gs.expectedTaps) {
    if (exp.matched) continue;
    const dist = Math.abs(now - exp.time);
    if (dist < okWindow && dist < nearestDist) { nearest = exp; nearestDist = dist; }
  }
  const btn = $('btn-tap');
  btn.classList.add('pressed'); setTimeout(() => btn.classList.remove('pressed'), 90);
  if (nearest) {
    nearest.matched = true;
    if      (nearestDist < lv.interval * 0.12) { nearest.score=100; nearest.grade='perfect'; showFeedback('Perfect!','perfect'); }
    else if (nearestDist < lv.interval * 0.22) { nearest.score=70;  nearest.grade='good';    showFeedback('Good!','good'); }
    else                                        { nearest.score=35;  nearest.grade='ok';      showFeedback('Ok','ok'); }
    const el = gs.prevActiveEl;
    if (el) { el.classList.add(`hit-${nearest.grade}`); setTimeout(() => el.classList.remove(`hit-${nearest.grade}`), 200); }
  } else {
    gs.extraTaps++; showFeedback('Extra!','extra');
  }
}

function showFeedback(text, grade) {
  const el = $('feedback-popup');
  el.textContent = text; el.className = `feedback-popup ${grade}`;
  clearTimeout(el._t);
  el._t = setTimeout(() => { el.textContent = ''; el.className = 'feedback-popup'; }, 650);
}

function computeScore() {
  const maxPts = gs.expectedTaps.length * 100;
  let earned = gs.expectedTaps.reduce((s,e) => s + (e.score||0), 0) - gs.extraTaps * 20;
  const accuracy = Math.max(0, Math.round(earned / maxPts * 100));
  const counts = { perfect:0, good:0, ok:0, miss:0 };
  gs.expectedTaps.forEach(e => { if (e.grade) counts[e.grade]++; });
  return { accuracy, counts, extra: gs.extraTaps };
}

function showResults(result) {
  const lv = LEVELS[gs.levelIdx], idx = gs.levelIdx;
  $('r-level').textContent = `Level ${idx+1} · ${lv.name}`;
  const scoreEl = $('r-score-big');
  scoreEl.textContent = `${result.accuracy}%`; scoreEl.className = 'r-score-big pop-in';
  $('r-stars').textContent = result.accuracy >= 90 ? '★★★' : result.accuracy >= 70 ? '★★☆' : result.accuracy >= 50 ? '★☆☆' : '☆☆☆';
  $('r-details').innerHTML = `
    <div class="r-detail-item"><span class="r-detail-value" style="color:var(--good)">${result.counts.perfect}</span><span class="r-detail-label">Perfect</span></div>
    <div class="r-detail-item"><span class="r-detail-value" style="color:#b0f060">${result.counts.good}</span><span class="r-detail-label">Good</span></div>
    <div class="r-detail-item"><span class="r-detail-value" style="color:var(--ok)">${result.counts.ok}</span><span class="r-detail-label">Ok</span></div>
    <div class="r-detail-item"><span class="r-detail-value" style="color:var(--miss)">${result.counts.miss}</span><span class="r-detail-label">Miss</span></div>
    <div class="r-detail-item"><span class="r-detail-value" style="color:var(--miss)">${result.extra}</span><span class="r-detail-label">Extra</span></div>
  `;
  const nextUnlocked = idx+1 < LEVELS.length && save.unlocked.includes(idx+1);
  $('r-msg').textContent = result.accuracy >= 80
    ? (nextUnlocked ? `🔓 Level ${idx+2} unlocked!` : idx+1 >= LEVELS.length ? '🏆 All levels complete!' : '🔓 Next level unlocked!')
    : `Need 80% to unlock the next level — you're ${80 - result.accuracy}% away.`;
  $('btn-next').style.display = nextUnlocked ? '' : 'none';
  showScreen('results');
}

$('btn-back').addEventListener('click', () => {
  cancelAnimationFrame(gs.rafId); clearTimeout(gs.readyTimer);
  gs.phase = 'idle'; renderHome(); showScreen('home');
});
$('btn-tap').addEventListener('click', handleTap);
$('btn-tap').addEventListener('touchstart', e => { e.preventDefault(); handleTap(); }, { passive: false });
document.addEventListener('keydown', e => { if (e.code === 'Space' && !e.repeat) { e.preventDefault(); handleTap(); } });
$('btn-retry').addEventListener('click', () => startLevel(gs.levelIdx));
$('btn-next').addEventListener('click', () => { const n = gs.levelIdx+1; if (n < LEVELS.length && save.unlocked.includes(n)) startLevel(n); });

renderHome();
