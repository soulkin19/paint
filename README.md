<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Soulkin Paint - Chat Stickers</title>
    <style>
        :root { --primary: #6366f1; --danger: #f43f5e; --bg: #f8fafc; --text: #1e293b; --card-bg: #ffffff; }
        body { font-family: 'Inter', sans-serif; background: var(--bg); color: var(--text); margin: 0; display: flex; flex-direction: column; align-items: center; min-height: 100vh; overflow: hidden; }
        .hidden { display: none !important; }

        .card { background: var(--card-bg); padding: 24px; border-radius: 20px; box-shadow: 0 10px 15px -3px rgba(0,0,0,0.05); margin: 10px; width: 90%; max-width: 450px; box-sizing: border-box; }
        input { margin: 8px 0; padding: 14px; width: 100%; border: 1px solid #e2e8f0; border-radius: 12px; box-sizing: border-box; font-size: 16px; }
        button { padding: 14px; cursor: pointer; background: var(--primary); color: white; border: none; border-radius: 12px; font-weight: 700; transition: 0.2s; }
        button:active { transform: scale(0.96); }
        .btn-outline { background: #fff; border: 1px solid #e2e8f0; color: #64748b; width: auto; padding: 8px 16px; font-size: 14px; }

        .header { background: rgba(255,255,255,0.9); backdrop-filter: blur(10px); width: 100%; padding: 12px 20px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #e2e8f0; position: sticky; top:0; z-index: 100; box-sizing: border-box; }
        
        #room-list { width: 95%; max-width: 500px; padding: 10px 0 120px 0; }
        .room-card { background: white; margin-bottom: 12px; padding: 18px; border-radius: 16px; display: flex; justify-content: space-between; align-items: center; border: 1px solid #e2e8f0; }

        #canvas-wrap { width: 100%; overflow: auto; display: flex; justify-content: center; align-items: center; background: #cbd5e1; flex-grow: 1; position: relative; }
        #canvas-container { display: flex; justify-content: center; align-items: center; }
        #canvas { background: white; box-shadow: 0 4px 25px rgba(0,0,0,0.1); touch-action: none; display: block; flex-shrink: 0; }
        
        .toolbar-wrapper { width: 100%; background: white; border-top: 1px solid #e2e8f0; padding: 12px 0; z-index: 200; }
        .toolbar-scroll { display: flex; overflow-x: auto; padding: 0 15px; gap: 10px; align-items: center; }
        .tool-btn { flex: 0 0 auto; min-width: 48px; height: 48px; display: flex; align-items: center; justify-content: center; font-size: 20px; background: #f1f5f9; border-radius: 12px; border: 2px solid transparent; }
        .tool-btn.active { border-color: var(--primary); background: #e0e7ff; color: var(--primary); }
        
        /* 会話用スタンプのアニメーション */
        #reaction-container { position: absolute; bottom: 100px; left: 0; width: 100%; height: 0; pointer-events: none; z-index: 500; overflow: visible; }
        .reaction-bubble { position: absolute; bottom: 0; left: 50%; transform: translateX(-50%); background: white; padding: 10px 20px; border-radius: 20px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); font-size: 24px; animation: floatUp 2s ease-out forwards; display: flex; flex-direction: column; align-items: center; }
        .reaction-user { font-size: 10px; color: #64748b; margin-top: 4px; }
        @keyframes floatUp { 0% { opacity: 0; transform: translate(-50%, 0); } 20% { opacity: 1; } 80% { opacity: 1; } 100% { opacity: 0; transform: translate(-50%, -100px); } }

        #stamp-menu { position: absolute; bottom: 75px; left: 15px; background: white; border: 1px solid #e2e8f0; border-radius: 12px; display: flex; gap: 8px; padding: 10px; box-shadow: 0 10px 20px rgba(0,0,0,0.1); z-index: 1000; }
        .stamp-option { font-size: 28px; cursor: pointer; padding: 5px; }

        .layer-box { display: flex; align-items: center; gap: 5px; background: #f1f5f9; padding: 5px 10px; border-radius: 14px; flex: 0 0 auto; }
        .layer-btn { padding: 8px 12px; font-size: 13px; border-radius: 8px; border: 1px solid #ddd; background: white; }
        .layer-btn.active { background: var(--primary); color: white; border-color: var(--primary); }
    </style>
</head>
<body>

    <div id="auth-page">
        <div class="card" style="margin-top: 60px;">
            <h2 style="text-align:center; color:var(--primary);">Soulkin Paint</h2>
            <input type="text" id="username" placeholder="名前">
            <input type="password" id="password" placeholder="パスワード">
            <button id="btn-action" style="width:100%;">ログイン / 登録</button>
        </div>
    </div>

    <div id="lobby-page" class="hidden">
        <div class="header">
            <span>👤 <b id="user-label"></b></span>
            <button id="btn-logout" class="btn-outline">ログアウト</button>
        </div>
        <div class="card">
            <h3 style="margin-top:0">新しく部屋を作る</h3>
            <input type="text" id="room-name" placeholder="部屋の名前">
            <div style="display:flex; gap:10px;">
                <input type="number" id="room-w" value="400" min="100" max="800">
                <input type="number" id="room-h" value="600" min="100" max="800">
            </div>
            <input type="password" id="room-join-pass" placeholder="入室パスワード">
            <input type="password" id="room-del-pass" placeholder="削除用パスワード">
            <button id="btn-create" style="width:100%;">作成して入室</button>
        </div>
        <div id="room-list"></div>
    </div>

    <div id="game-page" class="hidden" style="display:flex; flex-direction:column; height:100vh; width:100%;">
        <div id="reaction-container"></div>
        <div class="header">
            <div><b id="room-label"></b> <small id="online-count-badge" style="margin-left:8px; background:#e0e7ff; color:var(--primary); padding:2px 8px; border-radius:10px; cursor:pointer;">👤 <span id="online-count">1</span></small> <span id="zoom-label" style="cursor:pointer; font-size:12px; color:#64748b; background:#f1f5f9; padding:2px 8px; border-radius:10px; margin-left:5px;">1.0x</span></div>
            <div style="display:flex; gap:8px;"><button id="btn-save" class="btn-outline">💾</button><button id="btn-leave" class="btn-outline">退室</button></div>
        </div>

        <div id="canvas-wrap">
            <div id="canvas-container"><canvas id="canvas"></canvas></div>
        </div>

        <div class="toolbar-wrapper">
            <div id="stamp-menu" class="hidden">
                <span class="stamp-option">👍</span><span class="stamp-option">😊</span><span class="stamp-option">❤️</span><span class="stamp-option">😲</span><span class="stamp-option">🎨</span><span class="stamp-option">🙏</span>
            </div>
            <div class="toolbar-scroll">
                <button id="btn-open-stamps" class="tool-btn">💬</button>
                <div class="layer-box">
                    <div id="layer-list" style="display:flex; gap:4px;"></div>
                    <button id="btn-add-layer" style="border:none; background:#ddd; border-radius:8px; padding:8px;">＋</button>
                </div>
                <input type="color" id="color-picker" value="#6366f1" style="width:48px; height:48px; border:none; padding:0; flex-shrink:0;">
                <button id="btn-pen" class="tool-btn active">🖊️</button>
                <button id="btn-eraser" class="tool-btn">🧽</button>
                <button id="btn-undo" class="tool-btn">↩️</button>
                <button id="btn-clear" class="tool-btn" style="background:var(--danger); color:white; display:none;">💣</button>
                <input type="range" id="size-range" min="1" max="50" value="5" style="min-width:80px;">
            </div>
        </div>
    </div>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
        import { getFirestore, collection, addDoc, getDocs, query, where, onSnapshot, orderBy, doc, deleteDoc } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";
        import { getDatabase, ref, push, onValue, set, onDisconnect, remove, limitToLast, query as dbQuery } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-database.js";

        const firebaseConfig = { apiKey: "AIzaSyCwhHspaG94goiCIjVj3h-Un5pBK3JTjMU", authDomain: "soulkin-aa3b7.firebaseapp.com", databaseURL: "https://soulkin-aa3b7-default-rtdb.firebaseio.com", projectId: "soulkin-aa3b7", storageBucket: "soulkin-aa3b7.firebasestorage.app", messagingSenderId: "358331064206", appId: "1:358331064206:web:d7760ea0919259418a4edf" };
        const app = initializeApp(firebaseConfig);
        const db = getFirestore(app);
        const rtdb = getDatabase(app);

        let myName = localStorage.getItem('soulkin_user') || "", activeRoomId = null, mode = 'pen', activeLayer = "1", roomLayers = ["1"], undoStack = [], scale = 1.0, initialDist = 0;

        // ログイン・ロビー処理
        window.onload = () => { if(myName) loginSuccess(myName); };
        document.getElementById('btn-action').onclick = async () => {
            const n = document.getElementById('username').value.trim(), p = document.getElementById('password').value.trim();
            if(!n || !p) return alert("入力不足");
            const s = await getDocs(query(collection(db,"users"),where("name","==",n),where("pass","==",p)));
            if(s.empty) { await addDoc(collection(db,"users"), {name:n, pass:p}); }
            localStorage.setItem('soulkin_user', n); loginSuccess(n);
        };
        function loginSuccess(n) { myName = n; document.getElementById('user-label').innerText = n; document.getElementById('auth-page').classList.add('hidden'); document.getElementById('lobby-page').classList.remove('hidden'); loadRooms(); }
        document.getElementById('btn-logout').onclick = () => { localStorage.removeItem('soulkin_user'); location.reload(); };

        function loadRooms() {
            onSnapshot(query(collection(db,"rooms"), orderBy("createdAt","desc")), snap => {
                const list = document.getElementById('room-list'); list.innerHTML = "<h3>部屋一覧</h3>";
                snap.forEach(d => {
                    const r = d.data();
                    const isLocked = r.joinPass && r.joinPass !== "";
                    const div = document.createElement('div'); div.className = "room-card";
                    div.innerHTML = `<div class="room-info"><b>${isLocked ? '🔒 ' : ''}${r.name}</b><br><small>${r.w}x${r.h}</small></div>
                        <div style="display:flex; gap:8px;"><button class="btn-outline" onclick="window.tryJoin('${d.id}','${r.name}',${r.w},${r.h},'${r.host}','${r.joinPass || ""}')">入室</button>
                        <button onclick="window.deleteRoom('${d.id}','${r.delPass}')" style="background:none; border:none; color:red; font-size:18px;">🗑️</button></div>`;
                    list.appendChild(div);
                });
            });
        }
        window.tryJoin = (id, n, w, h, host, jp) => { if(jp !== "" && prompt("パスワード") !== jp) return alert("違うよ"); window.joinRoom(id, n, w, h, host); };
        window.deleteRoom = async (id, correct) => { if(prompt("削除パスワード") === correct) { await deleteDoc(doc(db,"rooms",id)); remove(ref(rtdb,`draws/${id}`)); } };
        document.getElementById('btn-create').onclick = async () => {
            const n = document.getElementById('room-name').value;
            let w = Math.max(100, Math.min(800, parseInt(document.getElementById('room-w').value) || 400));
            let h = Math.max(100, Math.min(800, parseInt(document.getElementById('room-h').value) || 600));
            const jp = document.getElementById('room-join-pass').value, dp = document.getElementById('room-del-pass').value;
            if(!n || !dp) return alert("部屋名と削除パス必須");
            const d = await addDoc(collection(db,"rooms"), {name:n, w, h, joinPass:jp, delPass:dp, host:myName, createdAt:Date.now()});
            window.joinRoom(d.id, n, w, h, myName);
        };

        // ペイント・ズーム処理
        const canvas = document.getElementById('canvas'), ctx = canvas.getContext('2d'), container = document.getElementById('canvas-container');
        let drawing = false, lx, ly;

        window.joinRoom = (id, name, w, h, host) => {
            activeRoomId = id; canvas.width = w; canvas.height = h;
            document.getElementById('lobby-page').classList.add('hidden'); document.getElementById('game-page').classList.remove('hidden');
            document.getElementById('room-label').innerText = name;
            document.getElementById('btn-clear').style.display = (host === myName) ? "flex" : "none";
            
            // プレゼンス管理
            const pRef = ref(rtdb, `rooms/${id}/users/${myName}`); set(pRef, true); onDisconnect(pRef).remove();
            
            // 会話スタンプの購読（最新のもののみ）
            onValue(dbQuery(ref(rtdb, `rooms/${id}/stamps`), limitToLast(1)), snap => {
                const data = snap.val(); if(!data) return;
                const key = Object.keys(data)[0]; const s = data[key];
                if(Date.now() - s.time < 2000) showReaction(s.emoji, s.user);
            });

            // 描画同期
            onValue(ref(rtdb, `draws/${id}`), snap => {
                ctx.clearRect(0,0,canvas.width,canvas.height);
                const data = snap.val() || {}, all = [];
                Object.keys(data).forEach(lid => {
                    Object.keys(data[lid]).forEach(kid => { if(data[lid][kid].x1 !== undefined) all.push({...data[lid][kid], lid}); });
                });
                all.sort((a,b) => a.lid.localeCompare(b.lid, undefined, {numeric:true})).forEach(d => {
                    ctx.beginPath(); ctx.strokeStyle = d.c; ctx.lineWidth = d.s; ctx.lineCap = "round";
                    ctx.moveTo(d.x1, d.y1); ctx.lineTo(d.x2, d.y2); ctx.stroke();
                });
            });
        };

        // 会話スタンプ機能
        document.getElementById('btn-open-stamps').onclick = (e) => { e.stopPropagation(); document.getElementById('stamp-menu').classList.toggle('hidden'); };
        document.querySelectorAll('.stamp-option').forEach(el => {
            el.onclick = () => {
                push(ref(rtdb, `rooms/${activeRoomId}/stamps`), { emoji: el.innerText, user: myName, time: Date.now() });
                document.getElementById('stamp-menu').classList.add('hidden');
            };
        });

        function showReaction(emoji, user) {
            const div = document.createElement('div'); div.className = 'reaction-bubble';
            div.innerHTML = `${emoji}<span class="reaction-user">${user}</span>`;
            document.getElementById('reaction-container').appendChild(div);
            setTimeout(() => div.remove(), 2000);
        }

        // ズーム/描画ロジック（前回維持）
        function updateZoom(ns) { scale = Math.max(0.1, Math.min(5, ns)); container.style.transform = `scale(${scale})`; document.getElementById('zoom-label').innerText = scale.toFixed(1) + "x"; }
        document.getElementById('zoom-label').onclick = () => { container.style.transformOrigin = "center"; updateZoom(1.0); };
        const setZoomOrigin = (x, y) => { const r = container.getBoundingClientRect(); container.style.transformOrigin = `${(x - r.left) / scale}px ${(y - r.top) / scale}px`; };
        window.addEventListener('wheel', (e) => { if (activeRoomId && e.ctrlKey) { e.preventDefault(); setZoomOrigin(e.clientX, e.clientY); updateZoom(scale + (e.deltaY > 0 ? -0.1 : 0.1)); } }, { passive: false });

        canvas.addEventListener('touchstart', (e) => {
            if (e.touches.length === 2) {
                setZoomOrigin((e.touches[0].pageX + e.touches[1].pageX) / 2, (e.touches[0].pageY + e.touches[1].pageY) / 2);
                initialDist = Math.hypot(e.touches[0].pageX - e.touches[1].pageX, e.touches[0].pageY - e.touches[1].pageY);
            } else if (e.touches.length === 1) { start(e); }
        }, { passive: true });

        canvas.addEventListener('touchmove', (e) => {
            if (e.touches.length === 2) {
                const dist = Math.hypot(e.touches[0].pageX - e.touches[1].pageX, e.touches[0].pageY - e.touches[1].pageY);
                if (initialDist > 0) updateZoom(scale * (dist / initialDist));
                initialDist = dist;
            } else if (e.touches.length === 1 && drawing) { move(e); if(e.cancelable) e.preventDefault(); }
        }, { passive: false });

        const getPos = (e) => {
            const r = canvas.getBoundingClientRect(); const cx = e.touches ? e.touches[0].clientX : e.clientX, cy = e.touches ? e.touches[0].clientY : e.clientY;
            return [(cx - r.left) * (canvas.width / r.width), (cy - r.top) * (canvas.height / r.height)];
        };
        const start = (e) => { if(e.touches && e.touches.length > 1) return; drawing = true; [lx, ly] = getPos(e); undoStack.push([]); };
        const move = (e) => {
            if(!drawing) return; const [x,y] = getPos(e);
            const c = mode==='eraser' ? '#ffffff' : document.getElementById('color-picker').value;
            const r = push(ref(rtdb, `draws/${activeRoomId}/${activeLayer}`), {x1:lx, y1:ly, x2:x, y2:y, c, s:document.getElementById('size-range').value});
            undoStack[undoStack.length-1].push({l:activeLayer, k:r.key}); [lx,ly] = [x,y];
        };
        canvas.addEventListener('mousedown', start); window.addEventListener('mousemove', move); window.addEventListener('mouseup', () => drawing=false);
        canvas.addEventListener('touchend', () => { drawing=false; initialDist=0; });

        document.getElementById('btn-pen').onclick = () => { mode='pen'; updateBtn('btn-pen'); };
        document.getElementById('btn-eraser').onclick = () => { mode='eraser'; updateBtn('btn-eraser'); };
        function updateBtn(id) { document.querySelectorAll('.tool-btn').forEach(b => b.classList.remove('active')); document.getElementById(id).classList.add('active'); }
        document.getElementById('btn-undo').onclick = () => { const last = undoStack.pop(); if(last) last.forEach(i => remove(ref(rtdb, `draws/${activeRoomId}/${i.l}/${i.k}`))); };
        document.getElementById('btn-clear').onclick = () => { if(confirm("全消去？")) set(ref(rtdb, `draws/${activeRoomId}`), null); };
        document.getElementById('btn-save').onclick = () => { const a = document.createElement('a'); a.href = canvas.toDataURL(); a.download = 'paint.png'; a.click(); };
        document.getElementById('btn-leave').onclick = () => { location.reload(); };
        window.onclick = () => document.getElementById('stamp-menu').classList.add('hidden');
    </script>
</body>
</html>
