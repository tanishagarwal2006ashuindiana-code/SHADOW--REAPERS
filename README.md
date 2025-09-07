# SHADOW--REAPERS
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>SHADOW REAPERS — Esports Tournaments</title>
  <meta name="description" content="SHADOW REAPERS — Custom rooms & tournaments for BGMI, Free Fire, PUBG with non‑transferable rooms and live leaderboards." />
  <link rel="icon" href="data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 256 256'%3E%3Crect width='256' height='256' rx='48' fill='%2305050A'/%3E%3Cpath d='M128 44c-36 0-64 22-64 58 0 26 14 49 38 59l-10 33 36-24 36 24-10-33c24-10 38-33 38-59 0-36-28-58-64-58z' fill='%23d946ef'/%3E%3Cpath d='M96 122c0-19 15-34 32-34s32 15 32 34-15 34-32 34-32-15-32-34z' fill='%2300d1ff' opacity='.7'/%3E%3C/svg%3E" />
  <!-- Tailwind CDN (no build needed) -->
  <script src="https://cdn.tailwindcss.com"></script>
  <script>tailwind.config = { theme: { extend: { colors: { ink: '#05050A' } } } };</script>
  <!-- Lucide Icons -->
  <script type="module" src="https://unpkg.com/lucide@0.469.0/dist/umd/lucide.js"></script>
  <style>
    :root { color-scheme: dark; }
    .neon { text-shadow: 0 0 10px rgba(217,70,239,.8), 0 0 30px rgba(129,140,248,.5); }
    .card { @apply rounded-2xl p-5 border border-white/10 bg-gradient-to-b from-zinc-900/70 to-black/70 shadow-xl; }
    .btn { @apply px-4 py-2 rounded-2xl shadow-lg border border-white/10 bg-white/5 backdrop-blur text-white hover:bg-white/10 transition; }
    .badge { @apply text-xs px-2 py-1 rounded-lg border border-white/10 bg-white/5; }
    .grid-bg { background-image: radial-gradient(transparent 1px, rgba(255,255,255,0.03) 1px); background-size: 20px 20px; }
  </style>
</head>
<body class="min-h-screen bg-ink text-white grid-bg relative overflow-x-hidden">

  <!-- Background Glow -->
  <div aria-hidden class="pointer-events-none absolute inset-0 -z-10">
    <div class="absolute -top-32 -left-32 w-[42rem] h-[42rem] bg-fuchsia-600/20 blur-3xl rounded-full"></div>
    <div class="absolute -bottom-32 -right-32 w-[42rem] h-[42rem] bg-indigo-600/20 blur-3xl rounded-full"></div>
  </div>

  <!-- Navbar -->
  <header class="sticky top-0 z-50 backdrop-blur bg-black/30 border-b border-white/10">
    <div class="max-w-7xl mx-auto px-4 py-3 flex items-center justify-between">
      <div class="flex items-center gap-3">
        <i data-lucide="skull" class="w-7 h-7 text-fuchsia-400"></i>
        <div class="font-black tracking-wide text-xl">SHADOW <span class="text-fuchsia-400 neon">REAPERS</span></div>
      </div>
      <nav class="flex items-center gap-2 text-sm">
        <button class="btn" data-view="home">Home</button>
        <button class="btn" data-view="tournaments">Tournaments</button>
        <button class="btn" data-view="leaderboard">Leaderboard</button>
        <button class="btn" data-view="admin">Admin</button>
      </nav>
      <div id="authBox" class="flex items-center gap-2"></div>
    </div>
  </header>

  <!-- Main -->
  <main id="app" class="max-w-7xl mx-auto px-4 pt-10 pb-24">
    <!-- views injected by JS -->
  </main>

  <footer class="border-t border-white/10">
    <div class="max-w-7xl mx-auto px-4 py-8 text-sm opacity-80 flex items-center gap-2">
      <i data-lucide="shield" class="w-4 h-4"></i>
      Non-transferable rooms • Variable entry fees • Built for BGMI / Free Fire / PUBG
    </div>
  </footer>

  <script>
    // ---------- Simple State (persisted to localStorage) ----------
    const DB = {
      load(key, fallback) { try { return JSON.parse(localStorage.getItem(key)) ?? fallback; } catch { return fallback; } },
      save(key, val) { localStorage.setItem(key, JSON.stringify(val)); }
    };
    const state = {
      user: DB.load('sr_user', null),
      tournaments: DB.load('sr_tournaments', null),
    };

    // Seed tournaments if empty
    if (!state.tournaments) {
      const now = Date.now();
      state.tournaments = [
        mkTournament('t1','BGMI Midnight Scrims','BGMI',29,100, now + 60*60*1000),
        mkTournament('t2','Free Fire Toxic Arena','Free Fire',19,48, now + 3*60*60*1000),
        mkTournament('t3','PUBG Proving Grounds','PUBG',49,100, now + 5*60*60*1000),
      ];
      DB.save('sr_tournaments', state.tournaments);
    }

    function mkTournament(id, name, title, fee, maxPlayers, startAt) {
      return {
        id, name, title, fee, maxPlayers, startAt,
        status: 'upcoming',
        entries: [],            // { uid, name, gamingId, paidAt }
        rooms: {},              // gamingId -> { roomId, roomPass }
        results: []             // { gamingId, kills, placement, points }
      };
    }

    // ---------- Utilities ----------
    const $ = sel => document.querySelector(sel);
    const $$ = sel => Array.from(document.querySelectorAll(sel));
    function save() { DB.save('sr_tournaments', state.tournaments); DB.save('sr_user', state.user); }
    function hmsLeft(ts) { const d = Math.max(0, ts - Date.now()); const h = Math.floor(d/3.6e6); const m = Math.floor((d%3.6e6)/6e4); return `${h}h ${m}m`; }
    function pointsFor(kills, placement) { return Math.max(0,100 - placement) + kills*2; }
    function uid() { return 'u_'+Math.random().toString(36).slice(2,9); }
    function roomFor(t, gamingId) {
      if (t.rooms[gamingId]) return t.rooms[gamingId];
      const room = { roomId: `${t.id.toUpperCase()}-${Math.random().toString(36).slice(2,6).toUpperCase()}`, roomPass: Math.random().toString(36).slice(2,8).toUpperCase() };
      t.rooms[gamingId] = room; return room;
    }
    function finishTournament(t) { t.status='finished'; t.rooms = {}; }

    // ---------- Auth UI ----------
    function renderAuth() {
      const box = $('#authBox');
      if (!state.user) {
        box.innerHTML = `
          <input id="nameIn" placeholder="Name" class="px-3 py-2 rounded-xl bg-black/40 border border-white/10 text-sm" />
          <input id="gidIn" placeholder="Gaming ID" class="px-3 py-2 rounded-xl bg-black/40 border border-white/10 text-sm" />
          <button class="btn flex items-center gap-1" id="loginBtn"><i data-lucide="log-in" class="w-4 h-4"></i> Login</button>`;
        $('#loginBtn').onclick = () => {
          const name = $('#nameIn').value || 'Player';
          const gid = $('#gidIn').value || 'GUEST';
          state.user = { uid: uid(), name, inGameId: gid };
          save(); renderAuth(); render(currentView);
        };
      } else {
        box.innerHTML = `
          <div class="text-sm opacity-80">${state.user.name} • <span class="opacity-60">${state.user.inGameId}</span></div>
          <button class="btn flex items-center gap-1" id="logoutBtn"><i data-lucide="log-out" class="w-4 h-4"></i> Logout</button>`;
        $('#logoutBtn').onclick = () => { state.user = null; save(); renderAuth(); render(currentView); };
      }
      lucide.createIcons();
    }

    // ---------- Views ----------
    let currentView = 'home';
    function switchView(v){ currentView = v; render(v); $$("nav .btn").forEach(btn=>btn.classList.remove('ring-2')); document.querySelector(`[data-view="${v}"]`).classList.add('ring-2'); }

    $$("[data-view]").forEach(btn => btn.addEventListener('click', () => switchView(btn.dataset.view)));

    function render(view){
      const app = $('#app');
      if (view==='home') {
        app.innerHTML = `
          <section class="grid md:grid-cols-2 gap-10 items-center">
            <div>
              <h1 class="text-4xl md:text-6xl font-black leading-tight">Enter the <span class="text-fuchsia-400 neon">SHADOW REAPERS</span> Arena</h1>
              <p class="mt-5 text-zinc-300/80 text-lg">Custom rooms. Real competition. Variable entry fees. Instant leaderboards for <b>BGMI</b>, <b>Free Fire</b>, and <b>PUBG</b>.</p>
              <div class="mt-6 flex gap-3">
                <button class="btn" onclick="switchView('tournaments')"><i data-lucide="trophy" class="w-5 h-5"></i> Explore Tournaments</button>
                <a class="btn border-fuchsia-400/40 bg-fuchsia-500/10" href="#" onclick="alert('Join our Discord coming soon!')"><i data-lucide="users" class="w-5 h-5"></i> Join Discord</a>
              </div>
              <div class="mt-6 flex gap-4 text-zinc-400 text-sm">
                <span class="flex items-center gap-2"><i data-lucide="gamepad-2" class="w-4 h-4"></i> Custom Rooms</span>
                <span class="flex items-center gap-2"><i data-lucide="lock" class="w-4 h-4"></i> Non‑Transferable</span>
                <span class="flex items-center gap-2"><i data-lucide="zap" class="w-4 h-4"></i> Real‑time Boards</span>
              </div>
            </div>
            <div class="relative">
              <div class="aspect-video rounded-3xl border border-fuchsia-500/40 bg-gradient-to-br from-black to-zinc-900 shadow-[0_0_120px_rgba(217,70,239,0.15)] overflow-hidden grid place-items-center">
                <div class="text-center">
                  <i data-lucide="skull" class="w-20 h-20 mx-auto text-fuchsia-400"></i>
                  <div class="mt-3 text-xl tracking-wide">ONE ID • ONE ROOM</div>
                  <div class="text-zinc-400 text-sm">Room auto‑deletes after match</div>
                </div>
              </div>
            </div>
          </section>`;
        lucide.createIcons();
        return;
      }

      if (view==='tournaments') {
        const cards = state.tournaments.map(t=>{
          const entries = t.entries.length;
          const spots = Math.max(0, t.maxPlayers - entries);
          const joined = state.user && t.entries.some(e=>e.uid===state.user.uid);
          return `
            <div class="card relative overflow-hidden">
              <div class="absolute inset-0 pointer-events-none opacity-10 bg-[radial-gradient(circle_at_30%_20%,#d946ef,transparent_40%),radial-gradient(circle_at_70%_80%,#818cf8,transparent_35%)]"></div>
              <div class="flex items-center justify-between">
                <div class="flex items-center gap-2">
                  <span class="badge">${t.title}</span>
                  <div class="font-bold text-lg">${t.name}</div>
                </div>
                <div class="text-sm text-zinc-400">Starts in ${hmsLeft(t.startAt)}</div>
              </div>
              <div class="mt-3 flex items-center justify-between text-zinc-300">
                <div class="flex items-center gap-3">
                  <span class="flex items-center gap-1"><i data-lucide="users" class="w-4 h-4"></i> ${entries}/${t.maxPlayers}</span>
                  <span class="flex items-center gap-1"><i data-lucide="indian-rupee" class="w-4 h-4"></i> ${t.fee}</span>
                </div>
                <div>
                  ${t.status==='upcoming' ? `<button class="btn joinBtn" data-id="${t.id}" ${!state.user||spots===0?'disabled':''}><i data-lucide="log-in" class="w-4 h-4"></i> ${joined?'Joined':'Join'}</button>` : ''}
                  ${t.status==='live' ? `<span class="text-emerald-400 font-semibold">LIVE</span>` : ''}
                  ${t.status==='finished' ? `<span class="text-zinc-400">Finished</span>` : ''}
                </div>
              </div>
              ${ state.user && joined ? roomPanelHTML(t) : '' }
              ${ t.status==='finished' && t.results.length ? inRoomRankingHTML(t) : '' }
            </div>`;
        }).join('');
        app.innerHTML = `<section><h2 class="text-2xl font-bold mb-4">Active & Upcoming Tournaments</h2><div class="grid md:grid-cols-2 xl:grid-cols-3 gap-6">${cards}</div></section>`;
        $$('.joinBtn').forEach(b=> b.addEventListener('click', ()=> joinTournament(b.dataset.id)));
        lucide.createIcons();
        return;
      }

      if (view==='leaderboard') {
        const totals = {}; // gamingId -> {points,kills,events}
        state.tournaments.forEach(t=> t.results.forEach(r=>{
          const o = totals[r.gamingId] || (totals[r.gamingId] = { points:0,kills:0,events:0 });
          o.points += r.points; o.kills += r.kills; o.events += 1;
        }));
        const rows = Object.entries(totals).map(([gamingId, v])=>({ gamingId, ...v })).sort((a,b)=>b.points-a.points).slice(0,100);
        const body = rows.map((r,i)=>`<tr class="odd:bg-white/5"><td class="p-2">${i+1}</td><td class="p-2 font-mono">${r.gamingId}</td><td class="p-2 font-semibold">${r.points}</td><td class="p-2">${r.kills}</td><td class="p-2">${r.events}</td></tr>`).join('');
        $('#app').innerHTML = `
          <section>
            <h2 class="text-2xl font-bold mb-4 flex items-center gap-2"><i data-lucide="trophy" class="w-5 h-5"></i> Global Leaderboard</h2>
            ${rows.length?`
            <div class="card overflow-x-auto">
              <table class="min-w-full text-sm">
                <thead>
                  <tr class="text-zinc-400">
                    <th class="text-left p-2">#</th>
                    <th class="text-left p-2">Gaming ID</th>
                    <th class="text-left p-2">Points</th>
                    <th class="text-left p-2">Kills</th>
                    <th class="text-left p-2">Events</th>
                  </tr>
                </thead>
                <tbody>${body}</tbody>
              </table>
            </div>` : `<div class="card text-zinc-400 text-sm">No results yet. Play an event to appear here.</div>`}
          </section>`;
        lucide.createIcons();
        return;
      }

      if (view==='admin') {
        const opts = state.tournaments.map(t=>`<option value="${t.id}">${t.name} — ${t.status}</option>`).join('');
        app.innerHTML = `
          <section class="grid md:grid-cols-2 gap-6">
            <div class="card">
              <div class="text-sm text-zinc-400 mb-3">Create Tournament</div>
              <div class="grid grid-cols-2 gap-3 text-sm">
                <label class="col-span-2">Title
                  <select id="a_title" class="mt-1 w-full bg-black/40 border border-white/10 rounded-xl p-2">
                    <option>BGMI</option><option>Free Fire</option><option>PUBG</option>
                  </select>
                </label>
                <label class="col-span-2">Name
                  <input id="a_name" placeholder="e.g., BGMI Night Rush" class="mt-1 w-full bg-black/40 border border-white/10 rounded-xl p-2" />
                </label>
                <label>Entry Fee (₹)
                  <input id="a_fee" type="number" value="10" class="mt-1 w-full bg-black/40 border border-white/10 rounded-xl p-2" />
                </label>
                <label>Max Players
                  <input id="a_max" type="number" value="100" class="mt-1 w-full bg-black/40 border border-white/10 rounded-xl p-2" />
                </label>
                <label class="col-span-2">Start At
                  <input id="a_start" type="datetime-local" class="mt-1 w-full bg-black/40 border border-white/10 rounded-xl p-2" />
                </label>
              </div>
              <div class="mt-4 flex gap-3">
                <button id="a_publish" class="btn flex items-center gap-2"><i data-lucide="zap" class="w-4 h-4"></i> Publish</button>
              </div>
              <div class="mt-2 text-xs text-zinc-500">Entry fees are variable and controlled here per event.</div>
            </div>

            <div class="card">
              <div class="text-sm text-zinc-400 mb-3">Control Room</div>
              <div class="grid gap-2">
                <select id="sel_t" class="bg-black/40 border border-white/10 rounded-xl p-2">
                  <option value="">Select Tournament</option>
                  ${opts}
                </select>
                <div class="flex gap-2">
                  <button id="startBtn" class="btn">Start (simulate)</button>
                  <button id="finishBtn" class="btn border-emerald-400/40 bg-emerald-500/10">Finish & Score</button>
                </div>
                <div class="text-xs text-zinc-500">Finishing will generate sample results and <b>auto‑delete rooms</b>.</div>
              </div>
            </div>
          </section>`;
        $('#a_start').value = new Date(Date.now()+60*60*1000).toISOString().slice(0,16);
        $('#a_publish').onclick = () => {
          const id = 't_'+Math.random().toString(36).slice(2,8);
          const t = mkTournament(id, $('#a_name').value || `${$('#a_title').value} Custom Room`, $('#a_title').value, Math.max(0,Number($('#a_fee').value)), Math.max(2, Number($('#a_max').value)), new Date($('#a_start').value).getTime());
          state.tournaments.push(t); save(); render('admin');
        };
        $('#startBtn').onclick = () => {
          const id = $('#sel_t').value; if (!id) return;
          const t = state.tournaments.find(x=>x.id===id); if (!t) return;
          t.status='live'; t.startAt=Date.now(); save(); render('admin');
        };
        $('#finishBtn').onclick = () => {
          const id = $('#sel_t').value; if (!id) return;
          const t = state.tournaments.find(x=>x.id===id); if (!t) return;
          // generate results for each entry
          t.results = t.entries.map(e=>{ const kills = Math.floor(Math.random()*12); const placement = Math.max(1, Math.floor(Math.random()* t.entries.length)); return { gamingId: e.gamingId, kills, placement, points: pointsFor(kills, placement) }; });
          finishTournament(t); save(); render('admin');
        };
        lucide.createIcons();
        return;
      }
    }

    function roomPanelHTML(t) {
      const entry = t.entries.find(e=> e.uid === state.user.uid);
      const r = t.rooms[entry?.gamingId] || {};
      const hasRoom = !!r.roomId;
      return `
        <div class="mt-4 card bg-black/40 border-fuchsia-500/20">
          <div class="flex items-center justify-between">
            <div class="flex items-center gap-2"><i data-lucide="lock" class="w-4 h-4 text-fuchsia-400"></i><div class="font-semibold">Your Room (locked to <span class="text-fuchsia-300">${entry.gamingId}</span>)</div></div>
            <span class="badge border-fuchsia-400/40 bg-fuchsia-500/10">Non‑Transferable</span>
          </div>
          <div class="mt-3 grid grid-cols-2 gap-4 text-sm">
            <div class="p-3 rounded-xl bg-white/5 border border-white/10">
              <div class="opacity-60">Room ID</div>
              <div class="font-mono text-lg">${hasRoom ? r.roomId : 'TBA'}</div>
            </div>
            <div class="p-3 rounded-xl bg-white/5 border border-white/10">
              <div class="opacity-60">Password</div>
              <div class="font-mono text-lg">${hasRoom ? r.roomPass : 'TBA'}</div>
            </div>
          </div>
          <div class="mt-3 text-xs text-zinc-400">Rooms auto‑delete when the match finishes.</div>
        </div>`;
    }

    function inRoomRankingHTML(t){
      const sorted = [...t.results].sort((a,b)=>b.points-a.points);
      const rows = sorted.map((r,i)=>`<tr class="odd:bg-white/5"><td class="p-2">${i+1}</td><td class="p-2 font-mono">${r.gamingId}</td><td class="p-2">${r.kills}</td><td class="p-2">${r.placement}</td><td class="p-2 font-semibold">${r.points}</td></tr>`).join('');
      return `
        <div class="mt-4">
          <div class="font-semibold mb-2">Match Ranking (individual)</div>
          <div class="overflow-x-auto">
            <table class="min-w-full text-sm">
              <thead>
                <tr class="text-zinc-400">
                  <th class="text-left p-2">#</th>
                  <th class="text-left p-2">Gaming ID</th>
                  <th class="text-left p-2">Kills</th>
                  <th class="text-left p-2">Placement</th>
                  <th class="text-left p-2">Points</th>
                </tr>
              </thead>
              <tbody>${rows}</tbody>
            </table>
          </div>
        </div>`;
    }

    // ---------- Actions ----------
    function joinTournament(id){
      const t = state.tournaments.find(x=>x.id===id);
      if (!t || !state.user) return;
      if (t.entries.some(e=>e.uid===state.user.uid)) return alert('Already joined.');
      if (t.entries.length >= t.maxPlayers) return alert('No spots left.');
      // Simulate payment success
      if (!confirm(`Pay ₹${t.fee} entry fee? (simulation)`)) return;
      t.entries.push({ uid: state.user.uid, name: state.user.name, gamingId: state.user.inGameId, paidAt: Date.now() });
      // Assign unique room locked to gamingId (non‑transferable)
      const room = roomFor(t, state.user.inGameId);
      save(); render('tournaments');
      alert(`Room Assigned:
ID: ${room.roomId}
Pass: ${room.roomPass}`);
    }

    // ---------- Init ----------
    renderAuth(); switchView('home');
  </script>
</body>
</html>
