cd ~/vibechat && cat << 'EOF' > server.mjs
import express from 'express';
import http from 'http';
import { WebSocketServer, WebSocket } from 'ws';

const app = express();
const PORT = process.env.PORT || 3000;
app.use(express.json({ limit: '50mb' }));

const rooms = {
  genel: { id: 'genel', name: '🌟 Genel Sohbet', description: 'Herkesin buluştuğu ana oda.', announcement: '👑 Kurucu: SONMOHİKAN | Hoş Geldiniz!', messages: [] },
  oyun: { id: 'oyun', name: '🎮 Oyun & Espor', description: 'Oyun ve sohbet.', announcement: '', messages: [] },
  muhabbet: { id: 'muhabbet', name: '☕ Kahve & Muhabbet', description: 'Sıcak sohbetler.', announcement: '', messages: [] },
  muzik: { id: 'muzik', name: '🎵 Müzik & Paylaşım', description: 'Şarkılar ve ses kayıtları.', announcement: '', messages: [] }
};

let adminId = null;
const clients = new Map();

function isSonmohikan(name) {
  if (!name) return false;
  const c = name.trim();
  return c.toUpperCase() === 'SONMOHIKAN' || c.toLocaleUpperCase('tr-TR') === 'SONMOHİKAN';
}

function broadcastToRoom(roomId, data) {
  const str = JSON.stringify(data);
  for (const [ws, c] of clients.entries()) {
    if (ws.readyState === WebSocket.OPEN && c.roomId === roomId) ws.send(str);
  }
}

function getOnlineUsers(roomId) {
  const users = [];
  const seen = new Set();
  for (const [, c] of clients.entries()) {
    if (c.roomId === roomId && !seen.has(c.user.id)) {
      seen.add(c.user.id);
      users.push(c.user);
    }
  }
  return users;
}

app.get('/api/rooms', (req, res) => {
  res.json({ rooms: Object.values(rooms).map(r => ({ id: r.id, name: r.name, description: r.description, announcement: r.announcement, messageCount: r.messages.length })) });
});

app.use((req, res) => {
  res.send(`<!DOCTYPE html>
<html lang="tr">
<head>
<meta charset="UTF-8"><meta name="viewport" content="width=device-width,initial-scale=1.0,maximum-scale=1.0,user-scalable=no">
<title>VibeChat Pro Live</title>
<script src="https://cdn.tailwindcss.com"></script>
<link href="https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;700;800&display=swap" rel="stylesheet">
<style>body{font-family:'Plus Jakarta Sans',sans-serif;}@keyframes pulse-ring{0%{transform:scale(0.95);opacity:0.8;}50%{transform:scale(1.1);opacity:0.4;}100%{transform:scale(0.95);opacity:0.8;}}.recording-pulse{animation:pulse-ring 1.5s infinite;}</style>
</head>
<body class="bg-slate-950 text-slate-100 flex flex-col h-screen overflow-hidden select-none">
  <header class="bg-slate-900 border-b border-slate-800 px-4 py-3 flex items-center justify-between z-20">
    <div class="flex items-center gap-2.5">
      <button id="toggleMenuBtn" class="p-2 rounded-xl bg-slate-800 text-slate-300">☰</button>
      <div>
        <div class="flex items-center gap-1.5">
          <h1 id="currentRoomTitle" class="text-sm font-bold text-white">🌟 Genel Sohbet</h1>
          <span class="px-1.5 py-0.5 rounded text-[9px] font-bold bg-emerald-500/20 text-emerald-400">● Canlı</span>
        </div>
        <p id="currentRoomDesc" class="text-[10px] text-slate-400">Herkesin buluştuğu ana oda</p>
      </div>
    </div>
    <div class="flex items-center gap-2">
      <button id="onlineUsersBtn" class="flex items-center gap-1 px-2.5 py-1.5 rounded-xl bg-slate-800 text-xs font-bold text-slate-300 border border-slate-700">
        <span class="w-2 h-2 rounded-full bg-emerald-400"></span><span id="onlineCountBadge">1</span>
      </button>
      <div id="userBadge" class="flex items-center gap-1.5 px-3 py-1.5 rounded-xl bg-slate-800 border border-slate-700 text-xs font-bold text-white"></div>
    </div>
  </header>

  <div id="announcementBanner" class="bg-amber-950/40 border-b border-amber-500/30 px-4 py-1.5 flex items-center justify-between text-xs text-amber-200">
    <div class="flex items-center gap-1.5 flex-1 overflow-hidden">
      <span class="text-amber-400 font-bold shrink-0">📢</span>
      <span id="announcementText" class="truncate font-medium">Hoş geldiniz!</span>
    </div>
    <button id="editAnnouncementBtn" class="text-[10px] bg-amber-500/20 text-amber-300 px-2 py-0.5 rounded border border-amber-500/40 hidden">Düzenle</button>
  </div>

  <div class="flex-1 flex overflow-hidden relative">
    <div id="roomsDrawer" class="absolute inset-y-0 left-0 w-72 bg-slate-900 border-r border-slate-800 z-30 transform -translate-x-full transition-transform duration-200 flex flex-col shadow-2xl">
      <div class="p-4 border-b border-slate-800 flex items-center justify-between">
        <h2 class="text-sm font-bold text-white">💬 Odalar</h2>
        <button id="closeDrawerBtn" class="text-slate-400 text-lg">✕</button>
      </div>
      <div class="p-3">
        <button id="createRoomBtn" class="w-full py-2 px-3 rounded-xl bg-indigo-600 text-white font-bold text-xs flex items-center justify-center gap-1">+ Yeni Oda Aç</button>
      </div>
      <div id="roomsList" class="flex-1 overflow-y-auto px-2 space-y-1"></div>
      <div class="p-3 border-t border-slate-800 bg-slate-950 flex items-center justify-between">
        <div class="text-xs font-bold" id="drawerUserName">Kullanıcı</div>
        <button id="changeNameBtn" class="text-xs text-indigo-400 font-medium">İsim Değiştir</button>
      </div>
    </div>

    <div id="messagesContainer" class="flex-1 overflow-y-auto p-4 space-y-3"></div>

    <div id="onlineUsersModal" class="absolute inset-y-0 right-0 w-64 bg-slate-900 border-l border-slate-800 z-30 transform translate-x-full transition-transform duration-200 flex flex-col shadow-2xl">
      <div class="p-4 border-b border-slate-800 flex items-center justify-between">
        <h3 class="text-sm font-bold text-white">🟢 Çevrimiçi Kişiler</h3>
        <button id="closeOnlineModalBtn" class="text-slate-400 text-lg">✕</button>
      </div>
      <div id="onlineUsersList" class="flex-1 overflow-y-auto p-3 space-y-2"></div>
    </div>
  </div>

  <div id="recordingBar" class="hidden bg-rose-950/90 border-t border-rose-500/40 p-3 flex items-center justify-between px-4 z-20">
    <div class="flex items-center gap-2">
      <div class="w-3 h-3 rounded-full bg-rose-500 recording-pulse"></div>
      <span class="text-xs font-bold text-rose-300">Ses Kaydediliyor...</span>
      <span id="recordingTimer" class="font-mono text-xs font-bold text-white bg-slate-900 px-2 py-0.5 rounded">0:00</span>
    </div>
    <div class="flex items-center gap-2">
      <button id="cancelRecordBtn" class="px-3 py-1.5 rounded-xl bg-slate-800 text-xs font-semibold text-slate-300">İptal</button>
      <button id="sendRecordBtn" class="px-3.5 py-1.5 rounded-xl bg-rose-600 text-xs font-bold text-white">✓ Gönder</button>
    </div>
  </div>

  <footer id="chatFooter" class="p-3 bg-slate-900 border-t border-slate-800 flex items-center gap-2 z-20">
    <label class="p-2.5 rounded-xl bg-slate-800 text-slate-300 cursor-pointer text-base">
      📷<input type="file" id="imageInput" accept="image/*" class="hidden" />
    </label>
    <button id="startRecordBtn" class="p-2.5 rounded-xl bg-slate-800 text-slate-300 hover:text-rose-400 text-base">🎙️</button>
    <input id="messageInput" type="text" placeholder="Mesajınızı yazın..." class="flex-1 bg-slate-950 border border-slate-700 rounded-xl px-4 py-2.5 text-sm text-white focus:outline-none focus:border-indigo-500" />
    <button id="sendBtn" class="bg-indigo-600 hover:bg-indigo-500 text-white font-bold px-4 py-2.5 rounded-xl text-sm">➤</button>
  </footer>

<script>
  const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  function playPop(pitch = 800) {
    try {
      if (audioCtx.state === 'suspended') audioCtx.resume();
      const osc = audioCtx.createOscillator(), gain = audioCtx.createGain();
      osc.type = 'sine'; osc.frequency.setValueAtTime(pitch, audioCtx.currentTime);
      gain.gain.setValueAtTime(0.12, audioCtx.currentTime);
      gain.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime + 0.1);
      osc.connect(gain); gain.connect(audioCtx.destination);
      osc.start(); osc.stop(audioCtx.currentTime + 0.1);
    } catch(e){}
  }

  let storedUser = null;
  try { storedUser = JSON.parse(localStorage.getItem('vc_user')); } catch(e){}
  if (!storedUser || !storedUser.name) {
    const defaultName = prompt('Kullanıcı adınız:', 'Misafir') || 'Misafir';
    const avatars = ['⚡', '🚀', '🐱', '🦊', '🔥', '👑', '🎮', '☕'];
    storedUser = { id: 'u-' + Math.random().toString(36).substring(2, 8), name: defaultName.trim(), avatar: avatars[Math.floor(Math.random() * avatars.length)] };
    localStorage.setItem('vc_user', JSON.stringify(storedUser));
  }

  const cleanName = storedUser.name.trim();
  const isAdmin = cleanName.toUpperCase() === 'SONMOHIKAN' || cleanName.toLocaleUpperCase('tr-TR') === 'SONMOHİKAN';
  storedUser.isAdmin = isAdmin;
  if (isAdmin) storedUser.avatar = '👑';

  const userBadge = document.getElementById('userBadge');
  userBadge.innerHTML = \`<span>\${storedUser.avatar}</span> <span>\${storedUser.name}</span>\`;
  if (isAdmin) userBadge.className += ' text-amber-300 border-amber-500/50 bg-amber-950/40';

  document.getElementById('drawerUserName').innerText = (isAdmin ? '👑 ' : '') + storedUser.name;
  document.getElementById('changeNameBtn').onclick = () => {
    const n = prompt('Yeni kullanıcı adınız:', storedUser.name);
    if (n && n.trim()) {
      storedUser.name = n.trim();
      localStorage.setItem('vc_user', JSON.stringify(storedUser));
      location.reload();
    }
  };

  let currentRoomId = 'genel', currentMessages = [];
  const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
  const ws = new WebSocket(protocol + '//' + window.location.host + '/ws');

  ws.onopen = () => { ws.send(JSON.stringify({ type: 'join', roomId: currentRoomId, user: storedUser })); };
  ws.onmessage = (e) => {
    const d = JSON.parse(e.data);
    if (d.type === 'room_history') {
      currentRoomId = d.room.id;
      document.getElementById('currentRoomTitle').innerText = d.room.name;
      document.getElementById('currentRoomDesc').innerText = d.room.description || '';
      updateAnnouncement(d.room.announcement);
      currentMessages = d.room.messages || [];
      renderMessages();
    } else if (d.type === 'new_message') {
      if (d.message.roomId === currentRoomId) {
        currentMessages.push(d.message);
        appendMessage(d.message);
        if (d.message.sender.id !== storedUser.id) playPop(900);
      }
    } else if (d.type === 'reaction_updated') {
      const target = currentMessages.find(m => m.id === d.messageId);
      if (target) { target.reactions = d.reactions; renderMessages(); }
    } else if (d.type === 'message_deleted') {
      currentMessages = currentMessages.filter(m => m.id !== d.messageId);
      renderMessages();
    } else if (d.type === 'announcement_updated') {
      if (d.roomId === currentRoomId) updateAnnouncement(d.announcement);
    } else if (d.type === 'online_users') {
      renderOnlineUsers(d.users);
    } else if (d.type === 'rooms_updated') {
      fetchRooms();
    }
  };

  function updateAnnouncement(text) {
    const banner = document.getElementById('announcementBanner');
    const txt = document.getElementById('announcementText');
    const btn = document.getElementById('editAnnouncementBtn');
    if (!text && !isAdmin) banner.classList.add('hidden');
    else { banner.classList.remove('hidden'); txt.innerText = text || '(Duyuru yok)'; }
    if (isAdmin) btn.classList.remove('hidden');
  }

  document.getElementById('editAnnouncementBtn').onclick = () => {
    const current = document.getElementById('announcementText').innerText;
    const n = prompt('Duyuru Metni:', current);
    if (n !== null) ws.send(JSON.stringify({ type: 'set_announcement', roomId: currentRoomId, announcement: n.trim(), sender: storedUser }));
  };

  function renderMessages() {
    const box = document.getElementById('messagesContainer');
    box.innerHTML = '';
    currentMessages.forEach(appendMessage);
    box.scrollTop = box.scrollHeight;
  }

  function appendMessage(msg) {
    const box = document.getElementById('messagesContainer');
    const isMe = msg.sender.id === storedUser.id;
    const isAdm = msg.sender.isAdmin;

    const wrap = document.createElement('div');
    wrap.className = 'flex flex-col ' + (isMe ? 'items-end' : 'items-start') + ' group';

    let reactionsHtml = '';
    if (msg.reactions && Object.keys(msg.reactions).length > 0) {
      reactionsHtml = '<div class="flex flex-wrap gap-1 mt-1">';
      for (const [emoji, users] of Object.entries(msg.reactions)) {
        if (users.length > 0) {
          const hasMine = users.includes(storedUser.name);
          reactionsHtml += \`<button onclick="toggleReaction('\${msg.id}', '\${emoji}')" class="px-2 py-0.5 rounded-full text-xs flex items-center gap-1 border \${hasMine ? 'bg-indigo-600/30 border-indigo-500 text-indigo-200' : 'bg-slate-900 border-slate-800 text-slate-300'}">\${emoji} <span class="font-bold text-[10px]">\${users.length}</span></button>\`;
        }
      }
      reactionsHtml += '</div>';
    }

    let mediaContent = '';
    if (msg.media) {
      if (msg.media.type === 'audio') {
        mediaContent = \`<div class="my-1 p-2 rounded-xl bg-slate-950/80 border border-slate-800 max-w-[260px]"><audio controls src="\${msg.media.url}" class="h-8 max-w-full"></audio></div>\`;
      } else if (msg.media.type === 'image') {
        mediaContent = \`<div class="my-1 rounded-xl overflow-hidden border border-slate-800 max-w-[280px]"><img src="\${msg.media.url}" class="w-full h-auto object-cover max-h-64 cursor-pointer" onclick="window.open('\${msg.media.url}')" /></div>\`;
      }
    }

    const quickBar = \`<div class="opacity-0 group-hover:opacity-100 transition-opacity flex items-center gap-1 mt-0.5 text-xs bg-slate-900 border border-slate-800 px-2 py-0.5 rounded-full shadow-sm">
      <button onclick="toggleReaction('\${msg.id}', '❤️')">❤️</button>
      <button onclick="toggleReaction('\${msg.id}', '👍')">👍</button>
      <button onclick="toggleReaction('\${msg.id}', '😂')">😂</button>
      <button onclick="toggleReaction('\${msg.id}', '🔥')">🔥</button>
      \${isAdmin ? \`<button onclick="deleteMsg('\${msg.id}')" class="text-rose-400 ml-1 font-bold">🗑️</button>\` : ''}
    </div>\`;

    wrap.innerHTML = \`
      <div class="text-[11px] text-slate-400 mb-0.5 font-semibold flex items-center gap-1">
        <span>\${msg.sender.avatar || '👤'}</span>
        <span class="\${isAdm ? 'text-amber-400 font-bold' : ''}">\${msg.sender.name}</span>
        \${isAdm ? '<span class="text-[9px] bg-amber-500/20 text-amber-300 border border-amber-500/40 px-1 rounded font-bold">KURUCU</span>' : ''}
      </div>
      <div class="px-3.5 py-2.5 rounded-2xl max-w-[85%] text-sm break-words shadow-sm \${isMe ? 'bg-indigo-600 text-white rounded-br-xs' : 'bg-slate-900 border border-slate-800 text-slate-100 rounded-bl-xs'}">
        \${msg.text ? \`<div>\${escapeHtml(msg.text)}</div>\` : ''}
        \${mediaContent}
      </div>
      \${reactionsHtml}
      \${quickBar}
    \`;
    box.appendChild(wrap);
    box.scrollTop = box.scrollHeight;
  }

  function escapeHtml(str) { return str.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;"); }
  window.toggleReaction = (messageId, emoji) => { ws.send(JSON.stringify({ type: 'reaction', roomId: currentRoomId, messageId, emoji, userName: storedUser.name })); };
  window.deleteMsg = (messageId) => { if (confirm('Mesaj silinsin mi?')) ws.send(JSON.stringify({ type: 'delete_message', roomId: currentRoomId, messageId, sender: storedUser })); };

  function sendText() {
    const inp = document.getElementById('messageInput');
    const text = inp.value.trim();
    if (!text) return;
    ws.send(JSON.stringify({ type: 'chat_message', roomId: currentRoomId, text, sender: storedUser }));
    playPop(600);
    inp.value = '';
  }
  document.getElementById('sendBtn').onclick = sendText;
  document.getElementById('messageInput').onkeydown = (e) => { if (e.key === 'Enter') sendText(); };

  document.getElementById('imageInput').onchange = (e) => {
    const file = e.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = (evt) => {
      ws.send(JSON.stringify({ type: 'chat_message', roomId: currentRoomId, media: { type: 'image', url: evt.target.result }, sender: storedUser }));
      playPop(600);
    };
    reader.readAsDataURL(file);
    e.target.value = '';
  };

  let mediaRecorder = null, audioChunks = [], recordTimerInterval = null, recordSeconds = 0;
  document.getElementById('startRecordBtn').onclick = async () => {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      mediaRecorder = new MediaRecorder(stream);
      audioChunks = [];
      mediaRecorder.ondataavailable = (e) => audioChunks.push(e.data);
      mediaRecorder.onstop = () => {
        const audioBlob = new Blob(audioChunks, { type: 'audio/webm' });
        const reader = new FileReader();
        reader.onload = (evt) => {
          ws.send(JSON.stringify({ type: 'chat_message', roomId: currentRoomId, media: { type: 'audio', url: evt.target.result }, sender: storedUser }));
          playPop(600);
        };
        reader.readAsDataURL(audioBlob);
        stream.getTracks().forEach(t => t.stop());
      };
      mediaRecorder.start();
      recordSeconds = 0;
      document.getElementById('recordingTimer').innerText = '0:00';
      document.getElementById('recordingBar').classList.remove('hidden');
      document.getElementById('chatFooter').classList.add('hidden');
      recordTimerInterval = setInterval(() => {
        recordSeconds++;
        const m = Math.floor(recordSeconds / 60), s = recordSeconds % 60;
        document.getElementById('recordingTimer').innerText = m + ':' + (s < 10 ? '0' : '') + s;
      }, 1000);
    } catch(err) { alert('Mikrofon izni gerek: ' + err.message); }
  };

  document.getElementById('cancelRecordBtn').onclick = () => {
    if (mediaRecorder && mediaRecorder.state !== 'inactive') { mediaRecorder.stop(); mediaRecorder.onstop = null; }
    clearInterval(recordTimerInterval);
    document.getElementById('recordingBar').classList.add('hidden');
    document.getElementById('chatFooter').classList.remove('hidden');
  };

  document.getElementById('sendRecordBtn').onclick = () => {
    clearInterval(recordTimerInterval);
    if (mediaRecorder && mediaRecorder.state !== 'inactive') mediaRecorder.stop();
    document.getElementById('recordingBar').classList.add('hidden');
    document.getElementById('chatFooter').classList.remove('hidden');
  };

  const drawer = document.getElementById('roomsDrawer');
  document.getElementById('toggleMenuBtn').onclick = () => drawer.classList.remove('-translate-x-full');
  document.getElementById('closeDrawerBtn').onclick = () => drawer.classList.add('-translate-x-full');

  async function fetchRooms() {
    try {
      const res = await fetch('/api/rooms');
      const data = await res.json();
      const list = document.getElementById('roomsList');
      list.innerHTML = '';
      data.rooms.forEach(r => {
        const btn = document.createElement('button');
        const isActive = r.id === currentRoomId;
        btn.className = \`w-full text-left p-3 rounded-xl flex items-center justify-between transition-colors \${isActive ? 'bg-indigo-600 text-white font-bold' : 'hover:bg-slate-800 text-slate-300'}\`;
        btn.innerHTML = \`<div><div class="text-xs">\${r.name}</div><div class="text-[10px] opacity-70 truncate max-w-[170px]">\${r.description || ''}</div></div><span class="text-[10px] px-1.5 py-0.5 rounded-full bg-slate-950/40">\${r.messageCount}</span>\`;
        btn.onclick = () => {
          currentRoomId = r.id;
          ws.send(JSON.stringify({ type: 'join', roomId: currentRoomId, user: storedUser }));
          drawer.classList.add('-translate-x-full');
        };
        list.appendChild(btn);
      });
    } catch(e) {}
  }
  fetchRooms();

  document.getElementById('createRoomBtn').onclick = () => {
    const name = prompt('Yeni oda adı:');
    if (!name) return;
    const desc = prompt('Açıklama:', 'Yeni sohbet kanalı') || '';
    ws.send(JSON.stringify({ type: 'create_room', name, description: desc, sender: storedUser }));
  };

  const onlineModal = document.getElementById('onlineUsersModal');
  document.getElementById('onlineUsersBtn').onclick = () => onlineModal.classList.remove('translate-x-full');
  document.getElementById('closeOnlineModalBtn').onclick = () => onlineModal.classList.add('translate-x-full');

  function renderOnlineUsers(users) {
    document.getElementById('onlineCountBadge').innerText = users.length;
    const list = document.getElementById('onlineUsersList');
    list.innerHTML = '';
    users.forEach(u => {
      const div = document.createElement('div');
      div.className = 'flex items-center gap-2 p-2 rounded-xl bg-slate-950/60 border border-slate-800';
      div.innerHTML = \`<span class="text-base">\${u.avatar || '👤'}</span><span class="text-xs font-semibold \${u.isAdmin ? 'text-amber-400' : 'text-slate-200'}">\${u.name} \${u.isAdmin ? '👑' : ''}</span>\`;
      list.appendChild(div);
    });
  }
</script>
</body>
</html>`);
});

const server = http.createServer(app);
const wss = new WebSocketServer({ server, path: '/ws' });

wss.on('connection', (ws) => {
  ws.on('message', (raw) => {
    try {
      const data = JSON.parse(raw);
      if (data.type === 'join') {
        const u = data.user || { id: 'u-' + Date.now(), name: 'Misafir', avatar: '👤' };
        if (isSonmohikan(u.name)) {
          if (!adminId) adminId = u.id;
          if (u.id === adminId) { u.isAdmin = true; u.avatar = '👑'; }
        }
        clients.set(ws, { ws, roomId: data.roomId || 'genel', user: u });
        const room = rooms[data.roomId || 'genel'] || rooms.genel;
        ws.send(JSON.stringify({ type: 'room_history', room }));
        broadcastToRoom(room.id, { type: 'online_users', users: getOnlineUsers(room.id) });
      } else if (data.type === 'chat_message') {
        const targetRoom = rooms[data.roomId];
        if (!targetRoom) return;
        const msg = {
          id: 'msg-' + Date.now() + '-' + Math.random().toString(36).substring(2, 6),
          roomId: data.roomId,
          text: data.text,
          media: data.media,
          sender: data.sender,
          reactions: {},
          timestamp: Date.now()
        };
        targetRoom.messages.push(msg);
        broadcastToRoom(data.roomId, { type: 'new_message', message: msg });
      } else if (data.type === 'reaction') {
        const targetRoom = rooms[data.roomId];
        if (!targetRoom) return;
        const msg = targetRoom.messages.find(m => m.id === data.messageId);
        if (msg) {
          if (!msg.reactions[data.emoji]) msg.reactions[data.emoji] = [];
          const idx = msg.reactions[data.emoji].indexOf(data.userName);
          if (idx >= 0) {
            msg.reactions[data.emoji].splice(idx, 1);
            if (msg.reactions[data.emoji].length === 0) delete msg.reactions[data.emoji];
          } else {
            msg.reactions[data.emoji].push(data.userName);
          }
          broadcastToRoom(data.roomId, { type: 'reaction_updated', messageId: msg.id, reactions: msg.reactions });
        }
      } else if (data.type === 'delete_message') {
        if (isSonmohikan(data.sender?.name)) {
          const targetRoom = rooms[data.roomId];
          if (targetRoom) {
            targetRoom.messages = targetRoom.messages.filter(m => m.id !== data.messageId);
            broadcastToRoom(data.roomId, { type: 'message_deleted', messageId: data.messageId });
          }
        }
      } else if (data.type === 'set_announcement') {
        if (isSonmohikan(data.sender?.name)) {
          const targetRoom = rooms[data.roomId];
          if (targetRoom) {
            targetRoom.announcement = data.announcement;
            broadcastToRoom(data.roomId, { type: 'announcement_updated', roomId: data.roomId, announcement: data.announcement });
          }
        }
      } else if (data.type === 'create_room') {
        const id = 'room-' + Date.now();
        rooms[id] = { id, name: data.name, description: data.description || '', announcement: '', messages: [] };
        for (const [clientWs] of clients.entries()) {
          if (clientWs.readyState === WebSocket.OPEN) clientWs.send(JSON.stringify({ type: 'rooms_updated' }));
        }
      }
    } catch(err) { console.error('WS Error:', err); }
  });

  ws.on('close', () => {
    const c = clients.get(ws);
    if (c) {
      const rId = c.roomId;
      clients.delete(ws);
      broadcastToRoom(rId, { type: 'online_users', users: getOnlineUsers(rId) });
    }
  });
});

server.listen(PORT, '0.0.0.0', () => {
  console.log(`✅ VibeChat Pro Sunucusu Port ${PORT} üzerinde hazır!`);
});
EOF
node server.mjs
