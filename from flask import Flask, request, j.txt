from flask import Flask, request, jsonify, render_template_string
import random
from datetime import date

app = Flask(__name__)

AFFIRMATIONS = [
    "You’re literally the main character 💅✨",
    "Soft, smart, unstoppable 💖",
    "You are doing SO good and it shows 🌷",
    "Your future self is proud of you already 💫",
    "You don’t chase, you attract 💕",
    "Tiny steps still count — you’re progressing 🫶",
    "Your vibe is expensive (in a good way) 💎",
    "Be kind to yourself, bestie 💗",
]

THEMES = {
    "pink": {"bg1": "#ffe1ee", "bg2": "#fff6fb", "card": "rgba(255,255,255,0.78)", "accent": "#ff4d8d"},
    "lavender": {"bg1": "#efe6ff", "bg2": "#fff6ff", "card": "rgba(255,255,255,0.78)", "accent": "#8b5cf6"},
    "mint": {"bg1": "#d9fff4", "bg2": "#f5fffb", "card": "rgba(255,255,255,0.78)", "accent": "#22c55e"},
}

# Simple in-memory storage (resets when app restarts)
STATE = {
    "todos": [],
    "glow_count": 0,
    "last_glow_date": None,
    "theme": "pink",
}

HTML = r"""
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8"/>
  <meta name="viewport" content="width=device-width, initial-scale=1"/>
  <title>GlowHub ✨</title>
  <style>
    :root{
      --bg1: {{bg1}};
      --bg2: {{bg2}};
      --card: {{card}};
      --accent: {{accent}};
      --text: #1f2937;
      --muted: #6b7280;
      --shadow: 0 18px 50px rgba(0,0,0,0.12);
      --radius: 22px;
    }
    *{box-sizing:border-box}
    body{
      margin:0;
      font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Arial;
      background: radial-gradient(1200px 700px at 20% 10%, var(--bg2), transparent),
                  linear-gradient(135deg, var(--bg1), var(--bg2));
      min-height:100vh;
      color:var(--text);
      overflow-x:hidden;
    }
    .wrap{
      max-width:980px;
      margin:0 auto;
      padding:28px 16px 60px;
    }
    .topbar{
      display:flex;
      align-items:center;
      justify-content:space-between;
      gap:12px;
      flex-wrap:wrap;
    }
    .title{
      display:flex; align-items:center; gap:12px;
    }
    .logo{
      width:52px; height:52px;
      border-radius:18px;
      background: linear-gradient(135deg, var(--accent), #ffd1e2);
      display:grid; place-items:center;
      box-shadow: var(--shadow);
      position:relative;
      overflow:hidden;
    }
    .logo::after{
      content:"";
      position:absolute; inset:-40%;
      background: conic-gradient(from 180deg, transparent, rgba(255,255,255,0.7), transparent);
      animation: spin 3.8s linear infinite;
    }
    @keyframes spin{to{transform:rotate(360deg)}}
    .logo span{position:relative; z-index:1; font-size:24px}
    h1{margin:0; font-size:28px; letter-spacing:-0.02em}
    .subtitle{margin:6px 0 0; color:var(--muted); font-size:14px}
    .pillbar{
      display:flex; gap:10px; align-items:center; flex-wrap:wrap;
    }
    .pill{
      padding:10px 12px;
      border-radius:999px;
      background: rgba(255,255,255,0.65);
      backdrop-filter: blur(10px);
      border: 1px solid rgba(255,255,255,0.7);
      box-shadow: 0 10px 25px rgba(0,0,0,0.06);
      display:flex; align-items:center; gap:10px;
    }
    .select, .btn{
      border:0;
      background: transparent;
      font-weight:600;
      color:var(--text);
      cursor:pointer;
      font-size:14px;
      outline:none;
    }
    .btn{
      padding:10px 14px;
      border-radius:999px;
      background: linear-gradient(135deg, var(--accent), #ffb6d0);
      color:white;
      box-shadow: 0 12px 25px rgba(0,0,0,0.12);
      transition: transform .12s ease, filter .12s ease;
      user-select:none;
    }
    .btn:hover{transform: translateY(-1px); filter: brightness(1.02)}
    .btn:active{transform: translateY(1px); filter: brightness(0.98)}
    .grid{
      margin-top:18px;
      display:grid;
      grid-template-columns: repeat(12, 1fr);
      gap:14px;
    }
    .card{
      grid-column: span 12;
      background: var(--card);
      border: 1px solid rgba(255,255,255,0.75);
      border-radius: var(--radius);
      box-shadow: var(--shadow);
      padding:16px;
      backdrop-filter: blur(14px);
      position:relative;
      overflow:hidden;
    }
    .card h2{
      margin:0 0 10px;
      font-size:18px;
      letter-spacing:-0.01em;
    }
    .card p{margin:0; color:var(--muted)}
    .two{grid-column: span 12}
    @media(min-width:860px){
      .two{grid-column: span 6}
    }
    .affirm{
      display:flex; gap:10px; align-items:flex-start; justify-content:space-between; flex-wrap:wrap;
    }
    .quote{
      font-size:18px;
      line-height:1.3;
      padding:12px 12px;
      border-radius: 18px;
      background: rgba(255,255,255,0.55);
      border: 1px dashed rgba(31,41,55,0.14);
      max-width: 650px;
    }
    .mini{
      display:flex; gap:10px; flex-wrap:wrap; align-items:center;
    }
    .input{
      width:100%;
      padding:12px 12px;
      border-radius: 14px;
      border: 1px solid rgba(31,41,55,0.12);
      background: rgba(255,255,255,0.7);
      outline:none;
      font-size:14px;
    }
    .row{display:flex; gap:10px; align-items:center; flex-wrap:wrap}
    .row .input{flex:1; min-width: 220px}
    ul{list-style:none; padding:0; margin:12px 0 0}
    li{
      display:flex; align-items:center; justify-content:space-between; gap:10px;
      padding:10px 12px;
      border-radius: 16px;
      background: rgba(255,255,255,0.55);
      border: 1px solid rgba(31,41,55,0.08);
      margin-bottom:10px;
    }
    .tag{
      font-size:12px;
      color:white;
      padding:4px 9px;
      border-radius:999px;
      background: rgba(31,41,55,0.25);
    }
    .danger{
      background: rgba(239,68,68,0.15);
      border: 1px solid rgba(239,68,68,0.25);
      color: #b91c1c;
      border-radius: 999px;
      padding:8px 12px;
      cursor:pointer;
      font-weight:700;
    }
    .moodwrap{
      display:flex; gap:12px; align-items:center; flex-wrap:wrap;
      margin-top:8px;
    }
    input[type="range"]{
      width:min(520px, 100%);
      accent-color: var(--accent);
    }
    .bigNum{
      font-size:34px;
      font-weight:900;
      letter-spacing:-0.03em;
    }
    .sparkle{
      position:fixed;
      pointer-events:none;
      inset:0;
      overflow:hidden;
      z-index:999;
    }
    .confetti{
      position:absolute;
      width:10px; height:14px;
      border-radius:3px;
      opacity:0.9;
      transform: translateY(-20px);
      animation: fall 1.2s ease-in forwards;
    }
    @keyframes fall{
      to{ transform: translateY(110vh) rotate(360deg); opacity:0; }
    }
    .toast{
      position:fixed;
      bottom:18px;
      left:50%;
      transform:translateX(-50%);
      background: rgba(31,41,55,0.85);
      color:white;
      padding:10px 14px;
      border-radius: 999px;
      font-size:14px;
      opacity:0;
      transition: opacity .2s ease, transform .2s ease;
      z-index:1000;
    }
    .toast.show{
      opacity:1;
      transform:translateX(-50%) translateY(-6px);
    }
  </style>
</head>
<body>
  <div class="wrap">
    <div class="topbar">
      <div class="title">
        <div class="logo"><span>✨</span></div>
        <div>
          <h1>GlowHub</h1>
          <div class="subtitle">a tiny cute space for your mood, glow, and main character energy</div>
        </div>
      </div>

      <div class="pillbar">
        <div class="pill">
          <span style="font-weight:800;">Theme</span>
          <select id="themeSelect" class="select" aria-label="Theme">
            <option value="pink">💗 Pink</option>
            <option value="lavender">💜 Lavender</option>
            <option value="mint">💚 Mint</option>
          </select>
        </div>
        <button class="btn" id="sparkleBtn">Sparkle Bomb ✨</button>
      </div>
    </div>

    <div class="grid">
      <div class="card two">
        <h2>Affirmation of the moment 💌</h2>
        <div class="affirm">
          <div class="quote" id="affirmText">{{affirmation}}</div>
          <div class="mini">
            <button class="btn" id="newAffirmBtn">New one!</button>
            <button class="btn" id="copyAffirmBtn">Copy</button>
          </div>
        </div>
        <p style="margin-top:10px;">Use this when your brain is being rude 😭</p>
      </div>

      <div class="card two">
        <h2>Mood slider 🎀</h2>
        <p>How are we doing bestie? Drag it.</p>
        <div class="moodwrap">
          <input id="mood" type="range" min="1" max="10" value="7"/>
          <div class="pill"><span style="font-weight:900;">Mood:</span> <span id="moodVal">7</span>/10 <span id="moodLabel" class="tag">Cute</span></div>
        </div>
      </div>

      <div class="card two">
        <h2>Mini To-Do List 📝</h2>
        <p>Small tasks only. Like… *drink water*, *stretch*, *reply message*.</p>
        <div class="row" style="margin-top:10px;">
          <input class="input" id="todoInput" placeholder="Add a task..."/>
          <button class="btn" id="addTodoBtn">Add</button>
        </div>
        <ul id="todoList"></ul>
      </div>

      <div class="card two">
        <h2>Daily Glow Counter 🌟</h2>
        <p>Click once per day for your “I showed up” badge.</p>
        <div class="row" style="margin-top:12px; justify-content:space-between;">
          <div>
            <div class="bigNum" id="glowCount">{{glow_count}}</div>
            <div style="color:var(--muted); font-weight:700;">total glow days</div>
          </div>
          <button class="btn" id="glowBtn">I Did Something Today 💖</button>
        </div>
        <p style="margin-top:10px;">Even if it’s just surviving. That counts.</p>
      </div>
    </div>
  </div>

  <div class="sparkle" id="sparkleLayer"></div>
  <div class="toast" id="toast">Saved 💾</div>

<script>
  const toast = document.getElementById("toast");
  const showToast = (msg="Done 💖") => {
    toast.textContent = msg;
    toast.classList.add("show");
    setTimeout(()=>toast.classList.remove("show"), 1200);
  };

  // Theme
  const themeSelect = document.getElementById("themeSelect");
  themeSelect.value = "{{theme}}";
  themeSelect.addEventListener("change", async () => {
    const res = await fetch("/api/theme", {
      method:"POST",
      headers: {"Content-Type":"application/json"},
      body: JSON.stringify({theme: themeSelect.value})
    });
    if(res.ok){ location.reload(); }
  });

  // Affirmations
  const affirmText = document.getElementById("affirmText");
  document.getElementById("newAffirmBtn").addEventListener("click", async () => {
    const res = await fetch("/api/affirmation");
    const data = await res.json();
    affirmText.textContent = data.affirmation;
    showToast("New affirmation ✨");
  });

  document.getElementById("copyAffirmBtn").addEventListener("click", async () => {
    try{
      await navigator.clipboard.writeText(affirmText.textContent);
      showToast("Copied 💅");
    }catch(e){
      showToast("Copy failed 😭");
    }
  });

  // Mood slider
  const mood = document.getElementById("mood");
  const moodVal = document.getElementById("moodVal");
  const moodLabel = document.getElementById("moodLabel");
  const moodName = (v) => {
    if(v <= 2) return ["Oof", "🥲"];
    if(v <= 4) return ["Meh", "😵‍💫"];
    if(v <= 6) return ["Okay", "🙂"];
    if(v <= 8) return ["Cute", "💖"];
    return ["Slay", "🔥"];
  };
  const updateMood = (v) => {
    moodVal.textContent = v;
    const [name, emoji] = moodName(Number(v));
    moodLabel.textContent = name + " " + emoji;
  };
  mood.addEventListener("input", (e) => updateMood(e.target.value));
  updateMood(mood.value);

  // Confetti sparkle button
  const sparkleLayer = document.getElementById("sparkleLayer");
  const sparkleBtn = document.getElementById("sparkleBtn");
  const rand = (a,b)=> Math.random()*(b-a)+a;

  sparkleBtn.addEventListener("click", () => {
    const pieces = 42;
    for(let i=0;i<pieces;i++){
      const el = document.createElement("div");
      el.className = "confetti";
      el.style.left = rand(0, 100) + "vw";
      el.style.top = rand(-10, 10) + "vh";
      el.style.background = `hsl(${Math.floor(rand(290, 360))}, 90%, 70%)`;
      el.style.animationDuration = rand(0.9, 1.6) + "s";
      el.style.transform = `translateY(-20px) rotate(${Math.floor(rand(0,360))}deg)`;
      sparkleLayer.appendChild(el);
      setTimeout(()=> el.remove(), 1700);
    }
    showToast("Sparkles deployed ✨");
  });

  // To-do list
  const todoInput = document.getElementById("todoInput");
  const todoList = document.getElementById("todoList");
  const addTodoBtn = document.getElementById("addTodoBtn");

  const renderTodos = (todos) => {
    todoList.innerHTML = "";
    if(!todos.length){
      const li = document.createElement("li");
      li.innerHTML = `<span style="color:var(--muted);">No tasks yet… we are in our soft era 🌷</span><span class="tag">chill</span>`;
      todoList.appendChild(li);
      return;
    }
    todos.forEach((t, idx) => {
      const li = document.createElement("li");
      li.innerHTML = `
        <span>${t}</span>
        <button class="danger" data-idx="${idx}">Delete</button>
      `;
      todoList.appendChild(li);
    });
  };

  const loadTodos = async () => {
    const res = await fetch("/api/todos");
    const data = await res.json();
    renderTodos(data.todos);
  };

  const addTodo = async () => {
    const text = todoInput.value.trim();
    if(!text){ showToast("Type something bestie 😭"); return; }
    const res = await fetch("/api/todos", {
      method:"POST",
      headers: {"Content-Type":"application/json"},
      body: JSON.stringify({text})
    });
    if(res.ok){
      todoInput.value = "";
      await loadTodos();
      showToast("Added ✅");
    }
  };

  addTodoBtn.addEventListener("click", addTodo);
  todoInput.addEventListener("keydown", (e)=>{ if(e.key==="Enter") addTodo(); });

  todoList.addEventListener("click", async (e) => {
    const btn = e.target.closest("button[data-idx]");
    if(!btn) return;
    const idx = Number(btn.dataset.idx);
    const res = await fetch("/api/todos/" + idx, { method:"DELETE" });
    if(res.ok){
      await loadTodos();
      showToast("Deleted 🗑️");
    }
  });

  // Glow counter
  const glowBtn = document.getElementById("glowBtn");
  const glowCountEl = document.getElementById("glowCount");

  glowBtn.addEventListener("click", async () => {
    const res = await fetch("/api/glow", { method:"POST" });
    const data = await res.json();
    glowCountEl.textContent = data.glow_count;
    showToast(data.message);
  });

  loadTodos();
</script>
</body>
</html>
"""

@app.route("/")
def home():
    theme = STATE.get("theme", "pink")
    t = THEMES.get(theme, THEMES["pink"])
    return render_template_string(
        HTML,
        affirmation=random.choice(AFFIRMATIONS),
        glow_count=STATE["glow_count"],
        theme=theme,
        **t
    )

@app.route("/api/affirmation")
def api_affirmation():
    return jsonify({"affirmation": random.choice(AFFIRMATIONS)})

@app.route("/api/todos", methods=["GET", "POST"])
def api_todos():
    if request.method == "GET":
        return jsonify({"todos": STATE["todos"]})
    data = request.get_json(force=True) or {}
    text = (data.get("text") or "").strip()
    if not text:
        return jsonify({"error": "empty"}), 400
    STATE["todos"].insert(0, text)
    STATE["todos"] = STATE["todos"][:20]
    return jsonify({"ok": True, "todos": STATE["todos"]})

@app.route("/api/todos/<int:idx>", methods=["DELETE"])
def api_delete_todo(idx):
    try:
        STATE["todos"].pop(idx)
        return jsonify({"ok": True, "todos": STATE["todos"]})
    except Exception:
        return jsonify({"error": "bad index"}), 404

@app.route("/api/glow", methods=["POST"])
def api_glow():
    today = date.today().isoformat()
    if STATE["last_glow_date"] == today:
        return jsonify({"glow_count": STATE["glow_count"], "message": "Already claimed today 😌💖"})
    STATE["glow_count"] += 1
    STATE["last_glow_date"] = today
    return jsonify({"glow_count": STATE["glow_count"], "message": "Glow day +1 🌟 proud of you bestie"})

@app.route("/api/theme", methods=["POST"])
def api_theme():
    data = request.get_json(force=True) or {}
    theme = data.get("theme")
    if theme not in THEMES:
        return jsonify({"error": "unknown theme"}), 400
    STATE["theme"] = theme
    return jsonify({"ok": True, "theme": theme})

if __name__ == "__main__":
    # Visit: http://127.0.0.1:5000
    app.run(debug=True)
