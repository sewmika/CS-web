# WhatsApp‑style Chat App — Full Source Code (Node.js + Socket.IO)

Below is a complete, minimal-yet-polished WhatsApp‑like real‑time web chat. It includes a contacts sidebar (online users), chat header, message bubbles with timestamps, typing indicators, message delivery to rooms and DMs, and a clean WhatsApp‑ish theme. No database (in‑memory) to keep it simple; you can add MongoDB/Firebase later.

---

## 📁 Project Structure
```
whatsapp-clone/
│
├── package.json
├── server.js
└── public/
    ├── index.html
    ├── style.css
    └── script.js
```

---

## package.json
```json
{
  "name": "whatsapp-clone",
  "version": "1.0.0",
  "description": "WhatsApp-like real-time chat (Node.js + Socket.IO)",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "socket.io": "^4.7.5",
    "uuid": "^9.0.1"
  }
}
```

---

## server.js (Backend)
```js
const express = require("express");
const { createServer } = require("http");
const { Server } = require("socket.io");
const { v4: uuidv4 } = require("uuid");

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer, {
  cors: { origin: "*" }
});

const PORT = process.env.PORT || 3000;

app.use(express.static("public"));

// In-memory stores
const users = new Map(); // socketId -> { id, name }

function getUserList() {
  return [...users.values()].map(u => ({ id: u.id, name: u.name }));
}

io.on("connection", (socket) => {
  // Join global room by default
  socket.join("global");

  socket.on("register", (name) => {
    const user = { id: socket.id, name: String(name || "User").slice(0, 32) };
    users.set(socket.id, user);

    // Notify self + others
    socket.emit("registered", { self: user, users: getUserList() });
    socket.broadcast.emit("user:joined", user);
  });

  socket.on("typing", (payload) => {
    // payload: { to: "global" | socketId, typing: boolean }
    const from = users.get(socket.id);
    if (!from) return;

    if (payload?.to === "global") {
      socket.to("global").emit("typing", { from: from.id, name: from.name, typing: !!payload.typing, scope: "global" });
    } else if (payload?.to) {
      socket.to(payload.to).emit("typing", { from: from.id, name: from.name, typing: !!payload.typing, scope: "dm" });
    }
  });

  socket.on("message", (msg) => {
    // msg: { to: "global" | socketId, text: string, clientId: string }
    const from = users.get(socket.id);
    if (!from || !msg?.text) return;

    const payload = {
      id: uuidv4(),
      from: { id: from.id, name: from.name },
      to: msg.to,
      text: String(msg.text).slice(0, 4000),
      at: Date.now(),
      clientId: msg.clientId || null
    };

    // Echo back to sender for delivery confirmation
    socket.emit("message:delivered", payload);

    if (msg.to === "global") {
      // Broadcast to everyone else in the room
      socket.to("global").emit("message", payload);
    } else if (msg.to) {
      // Direct message
      io.to(msg.to).emit("message", payload);
    }
  });

  socket.on("disconnect", () => {
    const user = users.get(socket.id);
    if (user) {
      users.delete(socket.id);
      socket.broadcast.emit("user:left", user);
    }
  });
});

httpServer.listen(PORT, () => {
  console.log(`Server running → http://localhost:${PORT}`);
});
```

---

## public/index.html (Frontend)
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>WhatsApp‑style Chat</title>
  <link rel="stylesheet" href="style.css" />
</head>
<body>
  <div id="app">
    <aside class="sidebar">
      <header class="sidebar__header">
        <div class="brand">Chat</div>
        <div class="me" id="meName">…</div>
      </header>
      <div class="sidebar__search">
        <input type="text" id="search" placeholder="Search or start new chat" />
      </div>
      <ul id="users" class="sidebar__list"></ul>
    </aside>

    <main class="chat">
      <header class="chat__header">
        <div class="chat__peer">
          <div id="peerName">Global Chat</div>
          <small id="typing" class="typing" hidden>typing…</small>
        </div>
      </header>

      <ul id="messages" class="chat__messages"></ul>

      <form id="composer" class="composer">
        <input id="input" type="text" placeholder="Type a message" autocomplete="off" />
        <button id="send" type="submit">Send</button>
      </form>
    </main>
  </div>

  <div class="login" id="login">
    <form id="loginForm" class="login__card">
      <h2>Enter your name</h2>
      <input id="name" type="text" maxlength="32" placeholder="e.g. Chamidu" required />
      <button type="submit">Join</button>
    </form>
  </div>

  <script src="/socket.io/socket.io.js"></script>
  <script src="script.js"></script>
</body>
</html>
```

---

## public/style.css (Styles)
```css
:root{
  --wa-green:#075E54;
  --wa-light:#25D366;
  --bg:#111b21;
  --panel:#202c33;
  --panel-light:#23323a;
  --text:#e9edef;
  --muted:#8696a0;
  --bubble:#005c4b;
  --bubble-you:#1f2c34;
}
*{box-sizing:border-box}
html,body{height:100%}
body{
  margin:0; font-family:system-ui,Segoe UI,Roboto,Helvetica,Arial; color:var(--text); background:var(--bg);
}
#app{display:flex; height:100vh; max-height:100vh}

/* Sidebar */
.sidebar{width:320px; background:var(--panel); border-right:1px solid #0b141a; display:flex; flex-direction:column}
.sidebar__header{display:flex; align-items:center; justify-content:space-between; padding:12px 14px; background:var(--panel-light);}
.brand{font-weight:700}
.me{font-size:.9rem; color:var(--muted)}
.sidebar__search{padding:8px 12px}
.sidebar__search input{width:100%; padding:10px 12px; border-radius:8px; border:none; outline:none; background:#111b21; color:var(--text)}
.sidebar__list{list-style:none; margin:0; padding:0; overflow:auto}
.sidebar__list li{display:flex; gap:10px; align-items:center; padding:10px 14px; cursor:pointer; border-bottom:1px solid #0b141a}
.sidebar__list li:hover{background:#182229}
.sidebar__list .avatar{width:36px; height:36px; border-radius:50%; background:#2a3942; display:grid; place-items:center; color:#b0bec5; font-weight:700}
.sidebar__list .meta{display:flex; flex-direction:column}
.sidebar__list .name{font-weight:600}
.sidebar__list .hint{font-size:.78rem; color:var(--muted)}

/* Chat */
.chat{flex:1; display:flex; flex-direction:column; background:url('data:image/svg+xml;utf8,<svg xmlns="http://www.w3.org/2000/svg" width="64" height="64" viewBox="0 0 64 64"><rect width="64" height="64" fill="%23111b21"/><circle cx="8" cy="8" r="1" fill="%23202c33"/></svg>')}
.chat__header{display:flex; align-items:center; gap:12px; padding:12px 16px; background:var(--panel-light); border-bottom:1px solid #0b141a}
.chat__peer div{font-weight:700}
.typing{color:var(--muted)}
.chat__messages{list-style:none; margin:0; padding:18px; flex:1; overflow:auto; display:flex; flex-direction:column; gap:6px}
.msg{max-width:70%; padding:8px 10px; border-radius:8px; position:relative; line-height:1.35}
.msg .time{font-size:.7rem; color:#c8d6db; opacity:.8; margin-left:8px}
.msg.you{align-self:flex-end; background:var(--bubble)}
.msg.them{align-self:flex-start; background:var(--bubble-you)}
.composer{display:flex; gap:8px; padding:12px; background:var(--panel)}
#input{flex:1; padding:12px; border-radius:10px; border:none; outline:none; background:#111b21; color:var(--text)}
#send{background:var(--wa-light); color:#003; border:none; padding:0 16px; border-radius:10px; font-weight:700; cursor:pointer}

/* Login overlay */
.login{position:fixed; inset:0; display:grid; place-items:center; background:rgba(0,0,0,.6)}
.login__card{background:var(--panel); padding:24px; border-radius:12px; width:min(92vw,360px); display:flex; flex-direction:column; gap:12px}
.login__card input{padding:10px 12px; border-radius:8px; border:none; outline:none; background:#111b21; color:var(--text)}
.login__card button{padding:10px 12px; border-radius:8px; border:none; background:var(--wa-green); color:#fff; font-weight:700; cursor:pointer}

@media (max-width: 820px){
  .sidebar{display:none}
}
```

---

## public/script.js (Client Logic)
```js
const socket = io();

// State
let SELF = null;
let CURRENT_TARGET = { id: "global", name: "Global Chat", scope: "global" }; // scope: global | dm
let TYPING_TIMEOUT = null;

// Elements
const login = document.getElementById("login");
const loginForm = document.getElementById("loginForm");
const nameInput = document.getElementById("name");
const meName = document.getElementById("meName");
const usersEl = document.getElementById("users");
const searchEl = document.getElementById("search");
const peerName = document.getElementById("peerName");
const typingEl = document.getElementById("typing");
const messagesEl = document.getElementById("messages");
const composer = document.getElementById("composer");
const input = document.getElementById("input");

// Helpers
function fmtTime(ts){
  const d = new Date(ts);
  return d.toLocaleTimeString([], { hour: '2-digit', minute:'2-digit' });
}
function addMsg(payload, isSelf){
  const li = document.createElement('li');
  li.className = `msg ${isSelf ? 'you' : 'them'}`;
  li.innerHTML = `${escapeHtml(payload.text)} <span class="time">${fmtTime(payload.at)}</span>`;
  messagesEl.appendChild(li);
  messagesEl.scrollTop = messagesEl.scrollHeight;
}
function escapeHtml(str){
  return str.replace(/[&<>"]+/g, s => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;'}[s]));
}
function renderUsers(list){
  const q = (searchEl.value||'').toLowerCase();
  usersEl.innerHTML = '';
  // Global pseudo-contact first
  const liGlobal = document.createElement('li');
  liGlobal.innerHTML = `<div class="avatar">G</div><div class="meta"><div class="name">Global Chat</div><div class="hint">Everyone</div></div>`;
  liGlobal.onclick = () => selectTarget({ id:'global', name:'Global Chat', scope:'global' });
  usersEl.appendChild(liGlobal);

  list.filter(u => u.id !== SELF?.id)
      .filter(u => u.name.toLowerCase().includes(q))
      .forEach(u => {
        const li = document.createElement('li');
        li.innerHTML = `<div class="avatar">${(u.name[0]||'#').toUpperCase()}</div>
                        <div class="meta"><div class="name">${escapeHtml(u.name)}</div><div class="hint">Online</div></div>`;
        li.onclick = () => selectTarget({ id:u.id, name:u.name, scope:'dm' });
        usersEl.appendChild(li);
      });
}
function selectTarget(t){
  CURRENT_TARGET = t;
  peerName.textContent = t.name;
  messagesEl.innerHTML = '';
}

// Login
loginForm.addEventListener('submit', (e) => {
  e.preventDefault();
  const name = nameInput.value.trim() || 'User';
  socket.emit('register', name);
});

// Typing indicator
input.addEventListener('input', () => {
  socket.emit('typing', { to: CURRENT_TARGET.id, typing: true });
  clearTimeout(TYPING_TIMEOUT);
  TYPING_TIMEOUT = setTimeout(() => socket.emit('typing', { to: CURRENT_TARGET.id, typing: false }), 900);
});

// Send message
composer.addEventListener('submit', (e) => {
  e.preventDefault();
  const text = input.value.trim();
  if(!text) return;
  const clientId = crypto.randomUUID ? crypto.randomUUID() : String(Math.random());
  socket.emit('message', { to: CURRENT_TARGET.id, text, clientId });
  input.value = '';
});

// Socket events
socket.on('registered', ({ self, users }) => {
  SELF = self;
  meName.textContent = self.name;
  renderUsers(users);
  login.hidden = true; // hide overlay
});

socket.on('user:joined', (user) => {
  renderUsers([...(window._users || []), user]);
});

socket.on('user:left', (user) => {
  const list = (window._users || []).filter(u => u.id !== user.id);
  window._users = list;
  renderUsers(list);
});

socket.on('typing', ({ from, name, typing, scope }) => {
  if (CURRENT_TARGET.scope === 'global' && scope === 'global') {
    typingEl.hidden = !typing;
  } else if (CURRENT_TARGET.scope === 'dm' && from === CURRENT_TARGET.id) {
    typingEl.hidden = !typing;
  }
  if (typing) clearTimeout(TYPING_TIMEOUT), TYPING_TIMEOUT = setTimeout(() => typingEl.hidden = true, 1200);
});

socket.on('message', (payload) => {
  const isForMe = (payload.to === 'global' && CURRENT_TARGET.id === 'global') || (payload.from.id === CURRENT_TARGET.id) || (payload.to === SELF.id && CURRENT_TARGET.id === payload.from.id);
  if (isForMe) addMsg(payload, payload.from.id === SELF.id);
});

socket.on('message:delivered', (payload) => {
  // Always show own message instantly
  const isSelf = payload.from.id === SELF.id;
  if (isSelf) addMsg(payload, true);
});

// Maintain a cached list of users from server push
socket.on('registered', ({ users }) => { window._users = users; renderUsers(users); });
socket.on('user:joined', (u) => { window._users = [ ...(window._users||[]), u ]; renderUsers(window._users); });
socket.on('user:left', (u) => { window._users = (window._users||[]).filter(x=>x.id!==u.id); renderUsers(window._users); });
```

---

## 🚀 Run Locally
```bash
npm install
npm start
# Open: http://localhost:3000
```
Open the URL in **two tabs** or **two devices** to test real‑time chat, typing indicator, and DMs.

---

## 🌐 Free Deploy (Render)
1. Push this folder to **GitHub** (public repo).
2. Go to **render.com** → New → **Web Service** → Connect repo.
3. Set Build Command: `npm install`  •  Start Command: `npm start`.
4. Deploy → you’ll get a public URL.

**Vercel/Netlify** can host the `public/` as static, but you still need a Node server for Socket.IO. Use **Render** or **Railway** for the backend.

---

## 🧩 Ideas to Extend
- Persist chats with MongoDB or Firebase.
- Auth with phone/email.
- Image/file messages (use uploads + object storage).
- Message receipts (sent/delivered/read).
- Group rooms with creation/invite.

> Need me to add **MongoDB persistence** or **image sending** next? Tell me what you want, I’ll extend this code.

