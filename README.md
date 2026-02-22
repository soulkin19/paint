<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Soulkin Paint - Canvas 2000 & Smooth Zoom</title>
    <style>
        :root { --primary: #6366f1; --danger: #f43f5e; --bg: #f8fafc; --text: #1e293b; --card-bg: #ffffff; }
        body { font-family: 'Inter', sans-serif; background: var(--bg); color: var(--text); margin: 0; display: flex; flex-direction: column; align-items: center; min-height: 100vh; overflow: hidden; }
        .hidden { display: none !important; }

        #lobby-page { width: 100%; display: flex; flex-direction: column; align-items: center; overflow-y: auto; height: 100vh; -webkit-overflow-scrolling: touch; }
        .card { background: var(--card-bg); padding: 24px; border-radius: 20px; box-shadow: 0 10px 15px -3px rgba(0,0,0,0.05); margin: 10px; width: 90%; max-width: 450px; box-sizing: border-box; flex-shrink: 0; }
        input { margin: 8px 0; padding: 14px; width: 100%; border: 1px solid #e2e8f0; border-radius: 12px; box-sizing: border-box; font-size: 16px; }
        button { padding: 14px; cursor: pointer; background: var(--primary); color: white; border: none; border-radius: 12px; font-weight: 700; transition: 0.2s; }
        
        .header { background: rgba(255,255,255,0.9); backdrop-filter: blur(10px); width: 100%; padding: 12px 20px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #e2e8f0; position: sticky; top:0; z-index: 100; box-sizing: border-box; }
        
        #room-list { width: 90%; max-width: 450px; padding-bottom: 50px; }
        .room-card { background: white; margin-bottom: 12px; padding: 18px; border-radius: 16px; display: flex; justify-content: space-between; align-items: center; border: 1px solid #e2e8f0; }

        #canvas-wrap { width: 100%; height: 100%; overflow: hidden; background: #cbd5e1; flex-grow: 1; position: relative; touch-action: none; cursor: crosshair; }
        #canvas-container { position: absolute; top: 0; left: 0; transform-origin: 0 0; will-change: transform; display: flex; justify-content: center; align-items: center; }
        #canvas { background: white; box-shadow: 0 4px 25px rgba(0,0,0,0.2); display: block; }
        
        #lock-overlay { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.1); display: none; z-index: 400; cursor: not-allowed; align-items: center; justify-content: center; font-weight: bold; color: white; text-shadow: 0 2px 4px rgba(0,0,0,0.5); font-size: 20px; pointer-events: all; }

        .toolbar-wrapper { width: 100%; background: white; border-top: 1px solid #e2e8f0; padding: 12px 0; z-index: 200; }
        .toolbar-scroll { display: flex; overflow-x: auto; padding: 0 15px; gap: 10px; align-items: center; }
        .tool-btn { flex: 0 0 auto; min-width: 48px; height: 48px; display: flex; align-items: center; justify-content: center; font-size: 20px; background: #f1f5f9; border-radius: 12px; border: 2px solid transparent; }
        .tool-btn.active { border-color: var(--primary); background: #e0e7ff; color: var(--primary); }
        
        .layer-box { display: flex; align-items: center; gap: 5px; background: #f1f5f9; padding: 5px 10px; border-radius: 14px; flex: 0 0 auto; }
        .layer-item { display: flex; align-items: center; background: white; border-radius: 8px; border: 1px solid #ddd; overflow: hidden; }
        .layer-btn { padding: 8px 12px; font-size: 13px; border: none; background: transparent; cursor: pointer; }
        .layer-item.active { background: var(--primary); border-color: var(--primary); }
        .layer-item.active .layer-btn { color: white; }
        .layer-del { padding: 8px 6px; font-size: 10px; color: #94a3b8; background: rgba(0,0,0,0.05); border: none; cursor: pointer; }
        
        .btn-outline { background: #fff; border: 1px solid #e2e8f0; color: #64748b; padding: 8px 16px; border-radius: 12px; }
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
            <button id="btn-logout" class="btn-outline">退出</button>
        </div>
        <div class="card">
            <h3 style="margin-top:0">部屋を作る</h3>
            <input type="text" id="room-name" placeholder="部屋の名前">
            <div style="display:flex; gap:10px;">
                <input type="number" id="room-w" value="400" min="100" max="2000">
                <input type="number" id="room-h" value="600" min="100" max="2000">
            </div>
            <input type="password" id="room-join-pass" placeholder="入室パスワード">
            <input type="password" id="room-del-pass" placeholder="削除用パスワード">
            <button id="btn-create" style="width:100%;">作成</button>
        </div>
        <div id="room-list"></div>
    </div>

    <div id="game-page" class="hidden" style="display:flex; flex-direction:column; height:100vh; width:100%;">
        <div id="reaction-container"></div>
        <div class="header">
            <div>
                <b id="room-label"></b> 
                <small id="online-count-badge" style="margin-left:8px; background:#e0e7ff; color:var(--primary); padding:2px 8px; border-radius:10px; cursor:pointer;">
                    👤 <span id="online-count">0</span>
                </small> 
                <span id="zoom-label" style="cursor:pointer; font-size:12px; color:#64748b; background:#f1f5f9; padding:2px 8px; border-radius:10px; margin-left:5px;" title="クリックでリセット">1.0x</span>
            </div>
            <div style="display:flex; gap:8px;"><button id="btn-save" class="btn-outline">💾</button><button id="btn-leave" class="btn-outline">退室</button></div>
        </div>

        <div id="canvas-wrap">
            <div id="lock-overlay">🔒 現在ロックされています</div>
            <div id="canvas-container"><canvas id="canvas"></canvas></div>
        </div>

        <div class="toolbar-wrapper">
            <div id="stamp-menu" class="hidden" style="position: absolute; bottom: 75px; left: 15px; background: white; border: 1px solid #e2e8f0; border-radius: 12px; display: flex; gap: 8px; padding: 10px; box-shadow: 0 10px 20px rgba(0,0,0,0.1); z-index: 1000;">
                <span class="stamp-option" style="font-size:28px; cursor:pointer;">👍</span><span class="stamp-option" style="font-size:28px; cursor:pointer;">😊</span><span class="stamp-option" style="font-size:28px; cursor:pointer;">❤️</span><span class="stamp-option" style="font-size:28px; cursor:pointer;">😲</span><span class="stamp-option" style="font-size:28px; cursor:pointer;">🎨</span><span class="stamp-option" style="font-size:28px; cursor:pointer;">🙏</span>
            </div>
            <div class="toolbar-scroll">
                <button id="btn-lock" class="tool-btn hidden">🔓</button>
                <button id="btn-open-stamps" class="tool-btn">💬</button>
                <div class="layer-box"><div id="layer-list" style="display:flex; gap:4px;"></div><button id="btn-add-layer" style="border:none; background:#ddd; border-radius:8px; padding:8px; cursor:pointer;">＋</button></div>
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

        let myName = localStorage.getItem('soulkin_user') || "", activeRoomId = null, mode = 'pen', activeLayer = "1", roomLayers = ["1"], undoStack = [], isLocked = false, onlineUsers = [];

        // --- ズーム・移動システム ---
        let scale = 1.0, posX = 0, posY = 0, lastPinchDist = 0;
        const container = document.getElementById('canvas-container');
        const wrap = document.getElementById('canvas-wrap');

        function updateTransform() {
            container.style.transform = `translate(${posX}px, ${posY}px) scale(${scale})`;
            document.getElementById('zoom-label').innerText = scale.toFixed(1) + "x";
        }

        // ズームリセット
        document.getElementById('zoom-label').onclick = () => {
            scale = 1.0; posX = (wrap.clientWidth - canvas.width) / 2; posY = (wrap.clientHeight - canvas.height) / 2;
            updateTransform();
        };

        // --- ログイン・部屋管理 ---
        window.onload = () => { if(myName) loginSuccess(myName); };
        document.getElementById('btn-action').onclick = async () => {
            const n = document.getElementById('username').value.trim(), p = document.getElementById('password').value.trim();
            if(!n || !p) return;
            const nameCheck = await getDocs(query(collection(db,"users"), where("name","==",n)));
            if(nameCheck.empty) { await addDoc(collection(db,"users"), {name:n, pass:p}); localStorage.setItem('soulkin_user', n); loginSuccess(n); }
            else {
                let matched = false; nameCheck.forEach(d => { if(d.data().pass === p) matched = true; });
                if(matched) { localStorage.setItem('soulkin_user', n); loginSuccess(n); }
                else alert("パスワードが違います");
            }
        };
        function loginSuccess(n) { myName = n; document.getElementById('user-label').innerText = n; document.getElementById('auth-page').classList.add('hidden'); document.getElementById('lobby-page').classList.remove('hidden'); loadRooms(); }
        document.getElementById('btn-logout').onclick = () => { localStorage.removeItem('soulkin_user'); location.reload(); };

        function loadRooms() {
            onSnapshot(query(collection(db,"rooms"), orderBy("createdAt","desc")), snap => {
                const list = document.getElementById('room-list'); list.innerHTML = "<h3>部屋一覧</h3>";
                snap.forEach(d => {
                    const r = d.data();
                    const div = document.createElement('div'); div.className = "room-card";
                    div.innerHTML = `<div><b>${r.joinPass ? '🔒 ' : ''}${r.name}</b><br><small>${r.w}x${r.h}</small></div>
                        <div style="display:flex; gap:8px;"><button class="btn-outline" onclick="window.tryJoin('${d.id}','${r.name}',${r.w},${r.h},'${r.host}','${r.joinPass || ""}')">入室</button>
                        <button onclick="window.deleteRoom('${d.id}','${r.delPass}')" style="color:red; border:none; background:none; font-size:20px;">🗑️</button></div>`;
                    list.appendChild(div);
                });
            });
        }
        window.tryJoin = (id, n, w, h, host, jp) => { 
            if (jp && jp.trim() !== "") {
                const input = prompt("パスワード：");
                if (input !== jp) return alert("違います");
            }
            window.joinRoom(id, n, w, h, host); 
        };
        window.deleteRoom = async (id, correct) => { if(prompt("削除パスワード") === correct) { await deleteDoc(doc(db,"rooms",id)); remove(ref(rtdb,`draws/${id}`)); } };

        document.getElementById('btn-create').onclick = async () => {
            const n = document.getElementById('room-name').value;
            // 上限を2000に変更
            let w = Math.max(100, Math.min(2000, parseInt(document.getElementById('room-w').value) || 400));
            let h = Math.max(100, Math.min(2000, parseInt(document.getElementById('room-h').value) || 600));
            const jp = document.getElementById('room-join-pass').value, dp = document.getElementById('room-del-pass').value;
            if(!n || !dp) return;
            const d = await addDoc(collection(db,"rooms"), {name:n, w, h, joinPass:jp, delPass:dp, host:myName, createdAt:Date.now()});
            window.joinRoom(d.id, n, w, h, myName);
        };

        // --- 描画・キャンバス制御 ---
        const canvas = document.getElementById('canvas'), ctx = canvas.getContext('2d');
        let drawing = false, lx, ly;

        window.joinRoom = (id, name, w, h, host) => {
            activeRoomId = id; canvas.width = w; canvas.height = h;
            document.getElementById('lobby-page').classList.add('hidden'); document.getElementById('game-page').classList.remove('hidden');
            document.getElementById('room-label').innerText = name;
            if (host === myName) { document.getElementById('btn-clear').style.display = "flex"; document.getElementById('btn-lock').classList.remove('hidden'); }
            
            // 初期位置を中央に
            posX = (wrap.clientWidth - w) / 2; posY = (wrap.clientHeight - h) / 2;
            updateTransform();

            set(ref(rtdb, `rooms/${id}/users/${myName}`), true);
            onDisconnect(ref(rtdb, `rooms/${id}/users/${myName}`)).remove();
            onValue(ref(rtdb, `rooms/${id}/users`), snap => { document.getElementById('online-count').innerText = Object.keys(snap.val() || {}).length; });
            onValue(ref(rtdb, `rooms/${id}/isLocked`), snap => {
                isLocked = snap.val() || false;
                document.getElementById('lock-overlay').style.display = isLocked ? "flex" : "none";
                document.getElementById('btn-lock').innerText = isLocked ? "🔒" : "🔓";
            });
            onValue(ref(rtdb, `draws/${id}`), snap => {
                ctx.clearRect(0,0,canvas.width,canvas.height);
                const data = snap.val() || {}, all = [], layers = [];
                Object.keys(data).forEach(lid => { layers.push(lid); Object.keys(data[lid]).forEach(kid => { if(data[lid][kid].x1!==undefined) all.push({...data[lid][kid], lid}); }); });
                all.sort((a,b)=>a.lid.localeCompare(b.lid,undefined,{numeric:true})).forEach(d => {
                    ctx.beginPath(); ctx.strokeStyle = d.c; ctx.lineWidth = d.s; ctx.lineCap = "round";
                    ctx.moveTo(d.x1, d.y1); ctx.lineTo(d.x2, d.y2); ctx.stroke();
                });
                roomLayers = layers.sort((a,b)=>a.localeCompare(b,undefined,{numeric:true})); if(roomLayers.length===0) roomLayers=["1"];
                renderLayerUI();
            });
        };

        // --- タッチ・マウス操作（修正版） ---
        const getCanvasPos = (e) => {
            const touch = e.touches ? e.touches[0] : e;
            const rect = wrap.getBoundingClientRect();
            const x = (touch.clientX - rect.left - posX) / scale;
            const y = (touch.clientY - rect.top - posY) / scale;
            return [x, y];
        };

        wrap.addEventListener('mousedown', (e) => {
            if(isLocked) return;
            drawing = true; [lx, ly] = getCanvasPos(e); undoStack.push([]);
        });

        window.addEventListener('mousemove', (e) => {
            if(!drawing) return;
            const [x, y] = getCanvasPos(e);
            const c = mode==='eraser'?'#ffffff':document.getElementById('color-picker').value;
            const s = document.getElementById('size-range').value;
            const r = push(ref(rtdb, `draws/${activeRoomId}/${activeLayer}`), {x1:lx, y1:ly, x2:x, y2:y, c, s});
            undoStack[undoStack.length-1].push({l:activeLayer, k:r.key}); [lx, ly] = [x, y];
        });

        window.addEventListener('mouseup', () => drawing = false);

        // 2本指操作（移動とズーム）
        wrap.addEventListener('touchstart', (e) => {
            if (e.touches.length === 2) {
                drawing = false;
                lastPinchDist = Math.hypot(e.touches[0].pageX - e.touches[1].pageX, e.touches[0].pageY - e.touches[1].pageY);
            } else if (!isLocked) {
                drawing = true; [lx, ly] = getCanvasPos(e); undoStack.push([]);
            }
        }, {passive:false});

        wrap.addEventListener('touchmove', (e) => {
            if (e.touches.length === 2) {
                e.preventDefault();
                const dist = Math.hypot(e.touches[0].pageX - e.touches[1].pageX, e.touches[0].pageY - e.touches[1].pageY);
                const deltaScale = dist / lastPinchDist;
                const newScale = Math.max(0.1, Math.min(5, scale * deltaScale));
                
                // 指の中心点を基準にズーム
                const centerX = (e.touches[0].clientX + e.touches[1].clientX) / 2;
                const centerY = (e.touches[0].clientY + e.touches[1].clientY) / 2;
                const rect = wrap.getBoundingClientRect();
                posX -= (centerX - rect.left - posX) * (newScale / scale - 1);
                posY -= (centerY - rect.top - posY) * (newScale / scale - 1);
                
                scale = newScale;
                lastPinchDist = dist;
                updateTransform();
            } else if (drawing) {
                e.preventDefault();
                const [x, y] = getCanvasPos(e);
                const c = mode==='eraser'?'#ffffff':document.getElementById('color-picker').value;
                push(ref(rtdb, `draws/${activeRoomId}/${activeLayer}`), {x1:lx, y1:ly, x2:x, y2:y, c, s:document.getElementById('size-range').value});
                [lx, ly] = [x, y];
            }
        }, {passive:false});

        wrap.addEventListener('touchend', () => { drawing = false; });

        // マウスホイールでのズーム
        wrap.addEventListener('wheel', (e) => {
            e.preventDefault();
            const delta = e.deltaY > 0 ? 0.9 : 1.1;
            const newScale = Math.max(0.1, Math.min(5, scale * delta));
            const rect = wrap.getBoundingClientRect();
            posX -= (e.clientX - rect.left - posX) * (newScale / scale - 1);
            posY -= (e.clientY - rect.top - posY) * (newScale / scale - 1);
            scale = newScale;
            updateTransform();
        }, {passive:false});

        // --- その他UI機能 ---
        function renderLayerUI() {
            const list = document.getElementById('layer-list'); list.innerHTML = "";
            roomLayers.forEach(l => {
                if(l==="_init") return;
                const div = document.createElement('div'); div.className = `layer-item ${activeLayer===l?'active':''}`;
                div.innerHTML = `<button class="layer-btn">L${l}</button><button class="layer-del">×</button>`;
                div.querySelector('.layer-btn').onclick = () => { activeLayer = l; renderLayerUI(); };
                div.querySelector('.layer-del').onclick = () => { if(roomLayers.length>1 && confirm("消去？")) remove(ref(rtdb, `draws/${activeRoomId}/${l}`)); };
                list.appendChild(div);
            });
        }
        document.getElementById('btn-add-layer').onclick = () => {
            const next = Math.max(...roomLayers.filter(l=>l!=="_init").map(Number)) + 1;
            set(ref(rtdb, `draws/${activeRoomId}/${next}/_init`), true); activeLayer = String(next);
        };
        document.getElementById('btn-undo').onclick = () => { const last = undoStack.pop(); if(last) last.forEach(i => remove(ref(rtdb, `draws/${activeRoomId}/${i.l}/${i.k}`))); };
        document.getElementById('btn-clear').onclick = () => { if(confirm("全消去？")) remove(ref(rtdb, `draws/${activeRoomId}`)); };
        document.getElementById('btn-pen').onclick = () => { mode='pen'; updateBtn('btn-pen'); };
        document.getElementById('btn-eraser').onclick = () => { mode='eraser'; updateBtn('btn-eraser'); };
        function updateBtn(id) { document.querySelectorAll('.tool-btn').forEach(b => b.classList.remove('active')); document.getElementById(id).classList.add('active'); }
        document.getElementById('btn-lock').onclick = () => set(ref(rtdb, `rooms/${activeRoomId}/isLocked`), !isLocked);
        document.getElementById('btn-open-stamps').onclick = (e) => { e.stopPropagation(); document.getElementById('stamp-menu').classList.toggle('hidden'); };
        document.getElementById('btn-save').onclick = () => { const a = document.createElement('a'); a.href = canvas.toDataURL(); a.download = 'art.png'; a.click(); };
        document.getElementById('btn-leave').onclick = () => location.reload();
    </script>
</body>
