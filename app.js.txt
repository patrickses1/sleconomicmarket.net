
const $ = (sel, node=document) => node.querySelector(sel);
const qs = (key, urlSearch) => new URLSearchParams(urlSearch || "").get(key);

let GOOGLE_MAPS_API_KEY = null;
let ADMIN_EMAILS = []; // from env.json (read-only baseline)

window.addEventListener('DOMContentLoaded', () => {
  fetch('env.json').then(r => r.ok ? r.json() : {}).catch(()=>({}))
    .then(env => {
      $('#afr').textContent = (env && env.AFRIMONEY_NUMBER) || '—';
      $('#orm').textContent = (env && env.ORANGEMONEY_NUMBER) || '—';
      GOOGLE_MAPS_API_KEY = env && env.GOOGLE_MAPS_API_KEY ? String(env.GOOGLE_MAPS_API_KEY) : null;
      if (env && Array.isArray(env.ADMIN_EMAILS)) {
        ADMIN_EMAILS = env.ADMIN_EMAILS.map(String);
      } else {
        ADMIN_EMAILS = [];
      }
      
      toggleAdminLink();
    }).catch(()=>{});
});

let _mapsLoading = null;
function ensureGoogleMaps() {
  if (!GOOGLE_MAPS_API_KEY) return Promise.resolve(null);
  if (_mapsLoading) return _mapsLoading;
  _mapsLoading = new Promise((resolve, reject) => {
    const s = document.createElement('script');
    s.src = `https://maps.googleapis.com/maps/api/js?key=${encodeURIComponent(GOOGLE_MAPS_API_KEY)}`;
    s.async = true; s.defer = true;
    s.onload = () => resolve(window.google && window.google.maps ? window.google.maps : null);
    s.onerror = () => reject(new Error('Failed to load Google Maps'));
    document.head.appendChild(s);
  });
  return _mapsLoading;
}

const DB = {
  _load(){
    try { return JSON.parse(localStorage.getItem('sleco_db')||'{}'); }
    catch { return {}; }
  },
  _save(data){ localStorage.setItem('sleco_db', JSON.stringify(data)); },
  get data(){
    const d = this._load();
    d.users ||= [];      // {id,email,passwordHash?,googleId?,role?,limitedAdminStatus?}
    d.sessions ||= {};   // token -> userId
    d.posts ||= [];      // {id,userId,category,title,price_cents,description,boosted_months,id_required_ok,payment_ok,payment_screenshot_name,createdAt}
    d.threads ||= [];    // {id,participants:[userId,userId],updatedAt}
    d.messages ||= [];   // {id,threadId,from,text,ts}
    return d;
  },
  set data(v){ this._save(v); }
};

function uid(){ return Math.random().toString(36).slice(2)+Date.now().toString(36); }
function hash(s){ return btoa(unescape(encodeURIComponent(s||''))); } 

const ADMIN_EMAILS_OVERRIDE_KEY = 'sleco_admin_emails_override';

function loadAdminOverride(){
  try {
    const raw = localStorage.getItem(ADMIN_EMAILS_OVERRIDE_KEY);
    if (!raw) return [];
    const arr = JSON.parse(raw);
    return Array.isArray(arr) ? arr.map(e => String(e).trim().toLowerCase()).filter(Boolean) : [];
  } catch { return []; }
}

function saveAdminOverride(list){
  const clean = Array.from(new Set((list||[]).map(e => String(e).trim().toLowerCase()).filter(Boolean)));
  localStorage.setItem(ADMIN_EMAILS_OVERRIDE_KEY, JSON.stringify(clean));
  return clean;
}

function clearAdminOverride(){ localStorage.removeItem(ADMIN_EMAILS_OVERRIDE_KEY); }

function getEffectiveAdminEmails(){
  const envList = (ADMIN_EMAILS || []).map(e => String(e).trim().toLowerCase()).filter(Boolean);
  const hasOverride = localStorage.getItem(ADMIN_EMAILS_OVERRIDE_KEY) !== null;
  const override = loadAdminOverride();
  return hasOverride ? override : envList;
}


function isMainAdmin(user){
  if (!user) return false;
  if (user.role === 'main_admin') return true;
  const list = getEffectiveAdminEmails();
  return list.includes(String(user.email||'').toLowerCase());
}


function toggleAdminLink(){
  const link = document.getElementById('adminLink');
  if (!link) return;
  const me = API._requireUser();
  link.style.display = isMainAdmin(me) ? '' : 'none';
}


const SL_BBOX = { latMin: 6.7, latMax: 10.1, lonMin: -13.5, lonMax: -10.0 };
const GEO_KEY = 'sleco_geo'; // { lat, lon, allowed, reason, checkedAt }

function isInSierraLeone(lat, lon){
  return (
    typeof lat === 'number' && typeof lon === 'number' &&
    lat >= SL_BBOX.latMin && lat <= SL_BBOX.latMax &&
    lon >= SL_BBOX.lonMin && lon <= SL_BBOX.lonMax
  );
}
function getCachedGeo(){
  try { return JSON.parse(localStorage.getItem(GEO_KEY) || 'null'); }
  catch { return null; }
}
function setCachedGeo(v){ localStorage.setItem(GEO_KEY, JSON.stringify(v)); }

async function requestGeolocationAndCache(){
  return new Promise((resolve) => {
    if (!('geolocation' in navigator)){
      const out = { lat:null, lon:null, allowed:false, reason:'no_geolocation', checkedAt: Date.now() };
      setCachedGeo(out); resolve(out); return;
    }
    navigator.geolocation.getCurrentPosition((pos)=>{
      const { latitude, longitude } = pos.coords || {};
      const ok = isInSierraLeone(latitude, longitude);
      const out = { lat: latitude, lon: longitude, allowed: ok, reason: ok?'ok':'outside', checkedAt: Date.now() };
      setCachedGeo(out); resolve(out);
    }, (err)=>{
      const out = { lat:null, lon:null, allowed:false, reason: err && err.code===1 ? 'denied' : 'error', checkedAt: Date.now() };
      setCachedGeo(out); resolve(out);
    }, { enableHighAccuracy:true, timeout:10000, maximumAge:0 });
  });
}


async function ensureGeoAccessFor(seg){
  if (seg[0] === 'admin') return true; // admin pages always allowed
  const me = API._requireUser();
  if (isMainAdmin(me)) return true;    // admins bypass
  if (seg.length === 0 || seg[0] === 'location') return true; // allow home/location

  let g = getCachedGeo();
  if (!g || (Date.now() - (g.checkedAt||0) > 24*60*60*1000)) {
    g = await requestGeolocationAndCache();
  }
  if (g && g.allowed) return true;

  app.innerHTML = `
    <section>
      <h2>Access limited to Sierra Leone</h2>
      <p class="muted">To use this app outside the Admin area, you must be physically located in Sierra Leone. Please verify your location.</p>
      <button id="verifySL">Verify I'm in Sierra Leone</button>
      <p style="margin-top:8px">If you are an Admin, go to <a href="#/admin">Admin</a>.</p>
      ${g && g.reason ? `<p class="muted">Last attempt: ${g.reason.replace('_',' ')}</p>` : ''}
    </section>
  `;
  $('#verifySL').onclick = async ()=>{
    const res = await requestGeolocationAndCache();
    if (res.allowed){ route(); } else { alert('Still not in Sierra Leone (or permission denied).'); }
  };
  return false;
}

// ---------- API (mock) ----------
const API = {
  token: localStorage.getItem('token') || null,
  setToken(t){ this.token = t; localStorage.setItem('token', t||''); renderAuth(); },

  _requireUser(){
    const d = DB.data;
    const uid_ = d.sessions[this.token || ''];
    if (!uid_) return null;
    return d.users.find(u => u.id === uid_) || null;
  },

  async post(path, body){
    const d = DB.data;

    // Auth
    if (path === '/api/auth/signup'){
      const {email, password} = body||{};
      if (!email || !password) return { error: 'Email & password required' };
      if (d.users.some(u => u.email === email)) return { error: 'Email already exists' };
      const isFirstUser = d.users.length === 0;
      const user = {
        id: uid(), email, passwordHash: hash(password),
        role: isFirstUser ? 'main_admin' : 'user',
        limitedAdminStatus:'none'
      };
      d.users.push(user);
      const token = uid(); d.sessions[token] = user.id; DB.data = d;
      return { token };
    }
    if (path === '/api/auth/login'){
      const {email, password} = body||{};
      const user = d.users.find(u => u.email === email && u.passwordHash === hash(password));
      if (!user) return { error: 'Invalid credentials' };
      const token = uid(); d.sessions[token] = user.id; DB.data = d;
      return { token };
    }
    if (path === '/api/auth/google/mock'){
      const {googleId, email} = body||{};
      let user = d.users.find(u => u.googleId === googleId || u.email === email);
      if (!user){
        user = { id: uid(), email: email||`user${Date.now()}@example.com`, googleId, role:'user', limitedAdminStatus:'none' };
        d.users.push(user);
      }
      const token = uid(); d.sessions[token] = user.id; DB.data = d;
      return { token };
    }

    
    if (path === '/api/users/request-limited-admin'){
      const me = this._requireUser();
      if (!me) return { error: 'Unauthorized' };
      me.limitedAdminStatus = 'requested';
      DB.data = d;
      return { status: 'requested' };
    }

   
    if (path === '/api/posts'){
      const me = this._requireUser();
      if (!me) return { error: 'Unauthorized' };
      const now = new Date().toISOString();
      const p = {
        id: uid(),
        userId: me.id,
        category: body.category,
        title: (body.title||'').trim(),
        price_cents: Number(body.price_cents||0),
        description: (body.description||'').trim(),
        boosted_months: Number(body.boosted_months||0),
        id_required_ok: !!body.id_required_ok,
        payment_ok: !!body.payment_ok,
        payment_screenshot_name: body.payment_screenshot_name||'',
        createdAt: now
      };
      d.posts.push(p); DB.data = d;
      return p;
    }

    
    if (path === '/api/admin/list-users'){
      const me = this._requireUser();
      if (!me) return { error: 'Unauthorized' };
      if (!isMainAdmin(me)) return { error: 'Admins only' };
      return { users: DB.data.users };
    }
    if (path === '/api/admin/limited-admin'){
      const me = this._requireUser();
      if (!me) return { error: 'Unauthorized' };
      if (!isMainAdmin(me)) return { error: 'Admins only' };
      const { userId, action } = body||{};
      const d2 = DB.data;
      const u = d2.users.find(u => u.id === userId);
      if (!u) return { error: 'User not found' };
      if (action === 'approve') u.limitedAdminStatus = 'approved';
      else if (action === 'reject') u.limitedAdminStatus = 'rejected';
      else if (action === 'remove') u.limitedAdminStatus = 'none';
      else return { error: 'Unknown action' };
      DB.data = d2;
      return { ok:true, user:u };
    }

    
    if (path === '/api/messages/start'){
      const me = this._requireUser(); if (!me) return { error:'Unauthorized' };
      const { email } = body||{};
      if (!email) return { error:'email required' };
      const other = d.users.find(u => u.email === email);
      if (!other) return { error:'User not found' };
      let th = d.threads.find(t => t.participants.length === 2 &&
        t.participants.includes(me.id) && t.participants.includes(other.id));
      if (!th){
        th = { id: uid(), participants:[me.id, other.id], updatedAt: new Date().toISOString() };
        d.threads.push(th);
        DB.data = d;
      }
      return { threadId: th.id };
    }
    if (path === '/api/messages/send'){
      const me = this._requireUser(); if (!me) return { error:'Unauthorized' };
      const { threadId, text } = body||{};
      if (!threadId || !text) return { error:'threadId & text required' };
      const th = d.threads.find(t => t.id === threadId);
      if (!th || !th.participants.includes(me.id)) return { error:'Not in thread' };
      const msg = { id: uid(), threadId, from: me.id, text: (text||'').trim(), ts: Date.now() };
      d.messages.push(msg);
      th.updatedAt = new Date().toISOString();
      DB.data = d;
      return { ok:true, message: msg };
    }

    return { error: 'Unknown endpoint' };
  },

  async get(path){
    
    if (path.startsWith('/api/posts')){
      const cat = qs('category', path.split('?')[1]||'');
      const d = DB.data;
      const items = (cat ? d.posts.filter(p => p.category === cat) : d.posts)
        .sort((a,b)=> b.createdAt.localeCompare(a.createdAt));
      return items;
    }


    if (path === '/api/messages/threads'){
      const me = this._requireUser(); if (!me) return { error:'Unauthorized' };
      const d = DB.data;
      const mine = d.threads
        .filter(t => t.participants.includes(me.id))
        .sort((a,b)=> b.updatedAt.localeCompare(a.updatedAt))
        .map(t => {
          const otherId = (t.participants || []).find(id => id !== me.id);
          const other = d.users.find(u => u.id === otherId) || {};
          const last = d.messages.filter(m => m.threadId === t.id).sort((a,b)=> b.ts-a.ts)[0];
          return {
            id: t.id,
            withEmail: other.email || '(unknown)',
            lastText: last ? last.text : '',
            updatedAt: t.updatedAt
          };
        });
      return mine;
    }

    
    if (path.startsWith('/api/messages/thread')){
      const me = this._requireUser(); if (!me) return { error:'Unauthorized' };
      const tid = qs('tid', path.split('?')[1]||'');
      const d = DB.data;
      const th = d.threads.find(t => t.id === tid);
      if (!th || !th.participants.includes(me.id)) return { error:'Not in thread' };
      const msgs = d.messages.filter(m => m.threadId === tid).sort((a,b)=> a.ts-b.ts);
      return msgs;
    }

    return { error: 'Unknown endpoint' };
  },

  async postForm(path, form){
    const obj = {};
    for (const [k,v] of form.entries()){
      if (v instanceof File){ obj[`${k}_name`] = v.name || 'upload'; }
      else { obj[k] = v; }
    }
    return this.post(path, obj);
  }
};

function renderAuth(){
  const el = $('#authArea');
  if (!API.token){
    el.innerHTML = `
      <input id="email" placeholder="email" />
      <input id="pass" type="password" placeholder="password" />
      <button id="loginBtn">Login</button>
      <button id="signupBtn">Sign up</button>
      <button id="googleBtn">Continue with Google</button>
    `;
    $('#loginBtn').onclick = async () => {
      const email=$('#email').value, password=$('#pass').value;
      const r=await API.post('/api/auth/login',{email,password});
      if(r.token){ API.setToken(r.token); toggleAdminLink(); route(); } else alert(r.error||'Login failed');
    };
    $('#signupBtn').onclick = async () => {
      const email=$('#email').value, password=$('#pass').value;
      const r=await API.post('/api/auth/signup',{email,password});
      if(r.token){ API.setToken(r.token); toggleAdminLink(); route(); } else alert(r.error||'Signup failed');
    };
    $('#googleBtn').onclick = async () => {
      const email=prompt('Google email (mock)');
      const r=await API.post('/api/auth/google/mock',{googleId:'mock-'+Date.now(), email});
      if(r.token){ API.setToken(r.token); toggleAdminLink(); route(); }
    };
    toggleAdminLink();
  } else {
    el.innerHTML = `
      <button id="logoutBtn">Logout</button>
      <button id="reqLA">Request Limited Admin</button>
    `;
    $('#logoutBtn').onclick = () => { API.setToken(null); toggleAdminLink(); location.hash = '#/'; };
    $('#reqLA').onclick = async () => {
      const r=await API.post('/api/users/request-limited-admin',{});
      alert('Limited admin status: '+(r.status||JSON.stringify(r)));
    };
    toggleAdminLink();
  }
}
renderAuth();

const app = $('#app');
const card = (t,d,b) => {
  const div = document.createElement('div'); div.className='card';
  div.innerHTML = `<span class="badge">${b||''}</span><h3>${t}</h3><p class="muted">${d||''}</p>`;
  return div;
};

async function viewGoods(){
  app.innerHTML = `<h2>Goods Feed</h2><div class="grid" id="grid"></div>`;
  const grid = $('#grid');
  const posts = await API.get('/api/posts?category=goods');
  posts.forEach(p => grid.appendChild(card(
    p.title,
    p.description,
    p.boosted_months>0?`Boosted ${p.boosted_months}m`:'')));
}

function postForm({category, requireID=false, requirePayment=false, allowBoost=false}){
  const wrap = document.createElement('div');
  wrap.innerHTML = `
    <h2>Create ${category.charAt(0).toUpperCase()+category.slice(1)} Post</h2>
    <form id="pform">
      <div class="row">
        <div><label>Title<input name="title" required /></label></div>
        <div><label>Price (¢)<input name="price_cents" type="number" min="0" /></label></div>
      </div>
      <label>Description<textarea name="description"></textarea></label>
      ${allowBoost?`<label>Boost Months (0-12)<input name="boosted_months" type="number" min="0" max="12" value="0"/></label>`:''}
      ${requireID?`<label>Upload ID (Services only)<input name="id_file" type="file" accept="image/*,application/pdf" required></label>`:''}
      ${requirePayment?`
        <fieldset class="pay">
          <legend>Payment Proof</legend>
          <p>Upload a screenshot/receipt of your payment so admins can verify.</p>
          <input name="payment_screenshot" type="file" accept="image/*,application/pdf" required />
        </fieldset>
      `:''}
      <div class="actions"><button type="submit">Publish</button></div>
    </form>
  `;
  const form = wrap.querySelector('#pform');
  form.addEventListener('submit', async (e)=>{
    e.preventDefault();
    const fd = new FormData(form);
    if (requireID) fd.set('id_required_ok','1');
    if (requirePayment) fd.set('payment_ok','0');
    fd.set('category', category);
    if (allowBoost && !fd.get('boosted_months')) fd.set('boosted_months','0');

    const pf = fd.get('payment_screenshot'); if (pf && pf.name) fd.set('payment_screenshot_name', pf.name);
    const idf = fd.get('id_file'); if (idf && idf.name) fd.set('id_file_name', idf.name);

    const res = await API.postForm('/api/posts', fd);
    if (res.error){ alert(res.error); return; }
    alert('Post created!'); location.hash = `#/${category}`;
  });
  return wrap;
}


async function viewAdmin(){
  const me = API._requireUser();
  if (!me){ app.innerHTML = `<p>Please log in to access Admin.</p>`; return; }
  if (!isMainAdmin(me)){ app.innerHTML = `<p>Admins only.</p>`; return; }

  app.innerHTML = `
    <section>
      <div style="display:flex;justify-content:space-between;align-items:center;gap:12px;flex-wrap:wrap">
        <h2 style="margin:0">Admin · Limited Admin Requests</h2>
        <a href="#/admin/settings" style="border:1px solid #e5e7eb;padding:6px 10px;border-radius:8px;background:#fff;text-decoration:none">Settings</a>
      </div>
      <div id="pending" style="margin-top:10px"></div>
      <h3>All Users</h3>
      <div id="users"></div>
    </section>
  `;

  const render = async () => {
    const res = await API.post('/api/admin/list-users', {});
    if (res.error){ app.innerHTML = `<p>${res.error}</p>`; return; }
    const users = res.users || [];

    const pending = users.filter(u => u.limitedAdminStatus === 'requested');
    $('#pending').innerHTML = pending.length
      ? `
        <h3>Pending Requests</h3>
        <table class="table">
          <thead><tr><th>Email</th><th>Status</th><th>Actions</th></tr></thead>
          <tbody>
            ${pending.map(u => `
              <tr data-id="${u.id}">
                <td>${u.email}</td>
                <td>${u.limitedAdminStatus}</td>
                <td>
                  <button class="approve" data-id="${u.id}">Approve</button>
                  <button class="reject" data-id="${u.id}">Reject</button>
                </td>
              </tr>
            `).join('')}
          </tbody>
        </table>
      `
      : `<p>No pending requests.</p>`;

    $('#users').innerHTML = `
      <table class="table">
        <thead><tr><th>Email</th><th>Role</th><th>Limited Admin</th><th>Actions</th></tr></thead>
        <tbody>
          ${users.map(u => `
            <tr data-id="${u.id}">
              <td>${u.email}${isMainAdmin(u) ? ' <small>(main admin)</small>' : ''}</td>
              <td>${u.role||'user'}</td>
              <td>${u.limitedAdminStatus||'none'}</td>
              <td>
                <button class="approve" data-id="${u.id}">Approve</button>
                <button class="reject" data-id="${u.id}">Reject</button>
                <button class="remove" data-id="${u.id}">Remove</button>
              </td>
            </tr>
          `).join('')}
        </tbody>
      </table>
    `;

    const doAction = async (userId, action) => {
      const r = await API.post('/api/admin/limited-admin', { userId, action });
      if (r.error) { alert(r.error); return; }
      await render();
    };

    app.querySelectorAll('button.approve').forEach(btn=>{ btn.onclick = () => doAction(btn.dataset.id, 'approve'); });
    app.querySelectorAll('button.reject').forEach(btn=>{ btn.onclick = () => doAction(btn.dataset.id, 'reject'); });
    app.querySelectorAll('button.remove').forEach(btn=>{ btn.onclick = () => doAction(btn.dataset.id, 'remove'); });
  };

  await render();
}

async function viewAdminSettings(){
  const me = API._requireUser();
  if (!me){ app.innerHTML = `<p>Please log in to access Admin Settings.</p>`; return; }
  if (!isMainAdmin(me)){ app.innerHTML = `<p>Admins only.</p>`; return; }

  const envList = (ADMIN_EMAILS || []).map(e => String(e).trim().toLowerCase()).filter(Boolean);
  const hadOverride = localStorage.getItem(ADMIN_EMAILS_OVERRIDE_KEY) !== null;
  const currentOverride = loadAdminOverride();
  const effective = getEffectiveAdminEmails();

  app.innerHTML = `
    <section>
      <h2>Admin · Settings</h2>
      <p class="muted">Manage the local allow-list of admin emails. This list overrides <code>env.json → ADMIN_EMAILS</code> and is stored only in this browser.</p>

      <div class="card" style="margin:10px 0;">
        <h3 style="margin:0 0 8px 0;">Effective Admin Emails</h3>
        <pre style="white-space:pre-wrap;margin:0">${effective.length ? effective.join('\n') : '(none)'}</pre>
      </div>

      <div class="card" style="margin:10px 0;">
        <h3 style="margin:0 0 8px 0;">env.json Admin Emails (read-only)</h3>
        <pre style="white-space:pre-wrap;margin:0">${envList.length ? envList.join('\n') : '(none in env.json)'}</pre>
      </div>

      <div class="card" style="margin:10px 0;">
        <h3 style="margin:0 0 8px 0;">Local Override (one email per line)</h3>
        <p class="muted" style="margin-top:0;">If you save an override (even empty), it fully replaces env.json's list on this device.</p>
        <textarea id="overrideTxt" style="width:100%; min-height:160px; font-family:ui-monospace, SFMono-Regular, Menlo, monospace;">${(hadOverride ? currentOverride : envList).join('\n')}</textarea>
        <div class="actions" style="margin-top:10px">
          <button id="saveOverride">Save Override</button>
          <button id="resetOverride">Reset (use env.json)</button>
        </div>
        <p id="status" class="muted" style="margin-top:8px;"></p>
      </div>

      <p><a href="#/admin">← Back to Admin</a></p>
    </section>
  `;

  const status = $('#status');
  $('#saveOverride').onclick = () => {
    const lines = $('#overrideTxt').value.split('\n')
      .map(s => s.trim().toLowerCase())
      .filter(s => s.length > 0);
    const bad = lines.find(e => !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(e));
    if (bad){ status.textContent = `Invalid email: ${bad}`; return; }
    const saved = saveAdminOverride(lines);
    status.textContent = `Saved ${saved.length} admin email(s) to local override.`;
    toggleAdminLink();
  };
  $('#resetOverride').onclick = () => {
    clearAdminOverride();
    $('#overrideTxt').value = envList.join('\n');
    status.textContent = 'Override cleared. Using env.json list.';
    toggleAdminLink();
  };
}

async function viewInbox(){
  const me = API._requireUser();
  if (!me){ app.innerHTML = `<p>Please log in to view your inbox.</p>`; return; }

  app.innerHTML = `
    <section>
      <h2>Inbox</h2>
      <form id="newChat" class="row" style="margin:10px 0;">
        <input name="email" placeholder="Start chat with email" required />
        <button type="submit">Start</button>
      </form>
      <div id="threads"></div>
    </section>
  `;

  $('#newChat').onsubmit = async (e) => {
    e.preventDefault();
    const email = new FormData(e.target).get('email');
    const r = await API.post('/api/messages/start', { email });
    if (r.error){ alert(r.error); return; }
    location.hash = `#/chat/${r.threadId}`;
  };

  const threads = await API.get('/api/messages/threads');
  if (threads.error){ app.innerHTML = `<p>${threads.error}</p>`; return; }
  $('#threads').innerHTML = threads.length ? `<div class="grid" id="tgrid"></div>` : `<p>No conversations yet.</p>`;

  const tgrid = $('#tgrid');
  (threads||[]).forEach(t => {
    const div = document.createElement('div'); div.className='card';
    div.innerHTML = `<h3>${t.withEmail}</h3><p class="muted">${t.lastText || '—'}</p><a href="#/chat/${t.id}">Open</a>`;
    tgrid.appendChild(div);
  });
}

async function viewChat(threadId){
  const me = API._requireUser();
  if (!me){ app.innerHTML = `<p>Please log in to view chat.</p>`; return; }

  app.innerHTML = `
    <section>
      <h2>Chat</h2>
      <div id="msgs" style="display:flex;flex-direction:column;gap:8px;margin:10px 0;"></div>
      <form id="sendForm" class="row">
        <input name="text" placeholder="Type a message…" required />
        <button type="submit">Send</button>
      </form>
    </section>
  `;

  async function render(){
    const msgs = await API.get(`/api/messages/thread?tid=${encodeURIComponent(threadId)}`);
    if (msgs.error){ app.innerHTML = `<p>${msgs.error}</p>`; return; }
    const box = $('#msgs');
    box.innerHTML = '';
    msgs.forEach(m => {
      const mine = m.from === me.id;
      const item = document.createElement('div');
      item.style.alignSelf = mine ? 'flex-end' : 'flex-start';
      item.className = 'card';
      item.style.maxWidth = '75%';
      item.innerHTML = `<p>${m.text}</p><small class="muted">${new Date(m.ts).toLocaleString()}</small>`;
      box.appendChild(item);
    });
    box.scrollTop = box.scrollHeight;
  }

  $('#sendForm').onsubmit = async (e) => {
    e.preventDefault();
    const text = new FormData(e.target).get('text');
    const r = await API.post('/api/messages/send', { threadId, text });
    if (r.error){ alert(r.error); return; }
    e.target.reset();
    await render();
  };

  await render();
}

async function viewLocation(){
  app.innerHTML = `
    <section>
      <h2>My Location</h2>
      <p class="muted">Click the button to request your current position.</p>
      <button id="locBtn">Get My Location</button>
      <div id="locInfo" style="margin-top:10px;"></div>
      <div id="map" style="height:360px;margin-top:10px;border:1px solid #e5e7eb;border-radius:10px;display:none;"></div>
    </section>
  `;

  $('#locBtn').onclick = async () => {
    if (!('geolocation' in navigator)){
      $('#locInfo').innerHTML = `<p>Geolocation is not supported by this browser.</p>`;
      return;
    }
    $('#locBtn').disabled = true; $('#locBtn').textContent = 'Requesting…';
    navigator.geolocation.getCurrentPosition(async (pos) => {
      const { latitude, longitude, accuracy } = pos.coords || {};
      const ok = isInSierraLeone(latitude, longitude);
      setCachedGeo({ lat: latitude, lon: longitude, allowed: ok, reason: ok?'ok':'outside', checkedAt: Date.now() });

      $('#locInfo').innerHTML = `
        <p><strong>Latitude:</strong> ${latitude?.toFixed(6)}<br/>
        <strong>Longitude:</strong> ${longitude?.toFixed(6)}<br/>
        <strong>Accuracy:</strong> ~${Math.round(accuracy||0)} meters<br/>
        <strong>Access:</strong> ${ok ? 'Allowed (Sierra Leone)' : 'Blocked (outside Sierra Leone)'}</p>
      `;

      try {
        const maps = await ensureGoogleMaps();
        if (maps){
          const mapEl = $('#map'); mapEl.style.display = 'block';
          const center = { lat: latitude, lng: longitude };
          const map = new maps.Map(mapEl, { center, zoom: 15 });
          new maps.Marker({ position: center, map, title: 'You are here' });
        }
      } catch {}

      $('#locBtn').disabled = false; $('#locBtn').textContent = 'Update Location';
    }, (err) => {
      setCachedGeo({ lat:null, lon:null, allowed:false, reason: err && err.code===1 ? 'denied' : 'error', checkedAt: Date.now() });
      $('#locInfo').innerHTML = `<p>Could not get location: ${err.message || err}</p>`;
      $('#locBtn').disabled = false; $('#locBtn').textContent = 'Get My Location';
    }, { enableHighAccuracy:true, timeout:10000, maximumAge:0 });
  };
}

async function route(){
  const h = (location.hash || '#/').replace(/^#/, '');
  const seg = h.split('/').filter(Boolean);

  const ok = await ensureGeoAccessFor(seg);
  if (!ok) return;

  if (seg.length === 0){
    app.innerHTML = `
      <section>
        <h2>Welcome</h2>
        <p>Select a category above or view the goods feed.</p>
      </section>
    `;
    return;
  }

  if (seg[0] === 'goods'){ await viewGoods(); return; }

  if (seg[0] === 'post'){
    const cat = seg[1]||'goods';
    const spec = {
      goods:    {category:'goods',    requireID:false, requirePayment:false, allowBoost:true},
      services: {category:'services', requireID:true,  requirePayment:true,  allowBoost:true},
      rentals:  {category:'rentals',  requireID:false, requirePayment:false, allowBoost:true},
      jobs:     {category:'jobs',     requireID:false, requirePayment:true,  allowBoost:false},
      ads:      {category:'ads',      requireID:false, requirePayment:false, allowBoost:true},
      advertising:{category:'ads',    requireID:false, requirePayment:false, allowBoost:true},
    }[cat] || {category:cat, requireID:false, requirePayment:false, allowBoost:false};
    app.innerHTML = ''; app.appendChild(postForm(spec)); return;
  }

  if (seg[0] === 'admin' && seg[1] === 'settings'){ await viewAdminSettings(); return; }
  if (seg[0] === 'admin'){ await viewAdmin(); return; }

  if (seg[0] === 'inbox'){ await viewInbox(); return; }
  if (seg[0] === 'chat' && seg[1]){ await viewChat(seg[1]); return; }

  if (seg[0] === 'location'){ await viewLocation(); return; }

  app.innerHTML = `<p>Not found. <a href="#/">Go Home</a></p>`;
}

window.addEventListener('hashchange', route);
window.addEventListener('DOMContentLoaded', route);
