<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>موقع الدردشة</title>
<style>
  :root { --bg:#f3f4f6; --panel:#ffffff; --me:#2563eb; --other:#e5e7eb; --text:#111827; --muted:#6b7280; --border:#e5e7eb; }
  @media (prefers-color-scheme: dark) {
    :root:not([data-theme="light"]) { --bg:#0f172a; --panel:#1e293b; --me:#3b82f6; --other:#334155; --text:#f1f5f9; --muted:#94a3b8; --border:#334155; }
  }
  :root[data-theme="dark"] { --bg:#0f172a; --panel:#1e293b; --me:#3b82f6; --other:#334155; --text:#f1f5f9; --muted:#94a3b8; --border:#334155; }
  * { box-sizing: border-box; }
  html, body { height: 100%; margin: 0; }
  body { background: var(--bg); color: var(--text); font-family: system-ui, "Segoe UI", Tahoma, sans-serif; display: flex; flex-direction: column; }
  header { background: var(--panel); border-bottom: 1px solid var(--border); padding: 12px 16px; display: flex; align-items: center; gap: 10px; padding-top: calc(12px + env(safe-area-inset-top, 0px)); }
  header h1 { font-size: 1.1rem; margin: 0; flex: 1; }
  .rooms { display: flex; gap: 6px; overflow-x: auto; }
  .rooms button { border: 1px solid var(--border); background: transparent; color: var(--text); padding: 6px 12px; border-radius: 999px; cursor: pointer; white-space: nowrap; font-size: .85rem; }
  .rooms button.active { background: var(--me); color: #fff; border-color: var(--me); }
  #messages { flex: 1; overflow-y: auto; padding: 16px; display: flex; flex-direction: column; gap: 10px; }
  .msg { max-width: 75%; padding: 8px 12px; border-radius: 14px; background: var(--other); word-wrap: break-word; }
  .msg.me { align-self: flex-start; background: var(--me); color: #fff; border-bottom-right-radius: 4px; }
  .msg:not(.me) { align-self: flex-end; border-bottom-left-radius: 4px; }
  .meta { font-size: .72rem; opacity: .75; margin-bottom: 2px; }
  .time { font-size: .65rem; opacity: .7; margin-top: 4px; text-align: left; }
  .empty { margin: auto; color: var(--muted); text-align: center; }
  .typing { color: var(--muted); font-size: .8rem; padding: 0 16px 6px; min-height: 1.2em; }
  form { display: flex; gap: 8px; padding: 10px 12px; padding-bottom: calc(10px + env(safe-area-inset-bottom, 0px)); background: var(--panel); border-top: 1px solid var(--border); }
  input[type=text] { flex: 1; padding: 10px 14px; border-radius: 999px; border: 1px solid var(--border); background: var(--bg); color: var(--text); font-size: 1rem; outline: none; }
  input[type=text]:focus { border-color: var(--me); }
  button.send { background: var(--me); color: #fff; border: none; border-radius: 999px; padding: 0 20px; cursor: pointer; font-size: 1rem; }
  button.send:disabled { opacity: .5; cursor: default; }
  .toggle { background: transparent; border: 1px solid var(--border); color: var(--text); border-radius: 8px; padding: 6px 8px; cursor: pointer; }
</style>
</head>
<body>
<header>
  <h1>💬 موقع الدردشة</h1>
  <button class="toggle" id="theme" title="تبديل المظهر">🌓</button>
</header>
<div class="rooms" id="rooms" style="padding:8px 12px;border-bottom:1px solid var(--border);background:var(--panel)"></div>
<div id="messages"></div>
<div class="typing" id="typing"></div>
<form id="form">
  <input type="text" id="input" placeholder="اكتب رسالتك..." autocomplete="off" maxlength="500">
  <button class="send" id="sendBtn" type="submit">إرسال</button>
</form>

<script>
const ROOMS = ["عام", "أخبار", "ترفيه", "تقنية"];
const BOT_NAMES = ["سارة", "أحمد", "ليلى", "يوسف"];
const REPLIES = ["هذا رائع!", "موافق تماماً 👍", "ممكن توضح أكثر؟", "حلو جداً 😄", "أنا أيضاً فكرت في ذلك", "تمام، نتكلم لاحقاً", "سؤال ممتاز!"];
const ME = "أنت";

let state = { room: ROOMS[0], messages: {} };
ROOMS.forEach(r => state.messages[r] = []);

try {
  const saved = localStorage.getItem("chat_state_v1");
  if (saved) {
    const parsed = JSON.parse(saved);
    if (parsed && parsed.messages) state.messages = { ...state.messages, ...parsed.messages };
  }
} catch (e) {}

function save() {
  try { localStorage.setItem("chat_state_v1", JSON.stringify({ messages: state.messages })); } catch (e) {}
}

const roomsEl = document.getElementById("rooms");
const msgsEl = document.getElementById("messages");
const typingEl = document.getElementById("typing");
const form = document.getElementById("form");
const input = document.getElementById("input");
const sendBtn = document.getElementById("sendBtn");

function fmtTime(ts) {
  return new Date(ts).toLocaleTimeString("ar", { hour: "2-digit", minute: "2-digit" });
}

function renderRooms() {
  roomsEl.innerHTML = "";
  ROOMS.forEach(r => {
    const b = document.createElement("button");
    b.textContent = r;
    if (r === state.room) b.classList.add("active");
    b.onclick = () => { state.room = r; renderRooms(); renderMessages(); };
    roomsEl.appendChild(b);
  });
}

function renderMessages() {
  msgsEl.innerHTML = "";
  const list = state.messages[state.room] || [];
  if (!list.length) {
    const e = document.createElement("div");
    e.className = "empty";
    e.textContent = "لا توجد رسائل بعد. ابدأ المحادثة!";
    msgsEl.appendChild(e);
    return;
  }
  list.forEach(m => {
    const d = document.createElement("div");
    d.className = "msg" + (m.from === ME ? " me" : "");
    const meta = document.createElement("div");
    meta.className = "meta";
    meta.textContent = m.from;
    const text = document.createElement("div");
    text.textContent = m.text;
    const t = document.createElement("div");
    t.className = "time";
    t.textContent = fmtTime(m.ts);
    d.append(meta, text, t);
    msgsEl.appendChild(d);
  });
  msgsEl.scrollTop = msgsEl.scrollHeight;
}

function addMessage(room, from, text) {
  state.messages[room].push({ from, text, ts: Date.now() });
  save();
  if (room === state.room) renderMessages();
}

function botReply(room) {
  const name = BOT_NAMES[Math.floor(Math.random() * BOT_NAMES.length)];
  typingEl.textContent = name + " يكتب...";
  setTimeout(() => {
    typingEl.textContent = "";
    const reply = REPLIES[Math.floor(Math.random() * REPLIES.length)];
    addMessage(room, name, reply);
  }, 1200 + Math.random() * 1200);
}

form.addEventListener("submit", e => {
  e.preventDefault();
  const text = input.value.trim();
  if (!text) return;
  const room = state.room;
  addMessage(room, ME, text);
  input.value = "";
  input.focus();
  botReply(room);
});

document.getElementById("theme").onclick = () => {
  const root = document.documentElement;
  const dark = root.getAttribute("data-theme") === "dark" ||
    (!root.getAttribute("data-theme") && matchMedia("(prefers-color-scheme: dark)").matches);
  root.setAttribute("data-theme", dark ? "light" : "dark");
};

renderRooms();
renderMessages();
input.focus();
</script>
</body>
</html>
