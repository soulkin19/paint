<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Soulkin Paint Ultra - Persistent Login</title>
    <style>
        :root { --primary: #6366f1; --danger: #f43f5e; --bg: #f8fafc; --text: #1e293b; --card-bg: #ffffff; }
        body { font-family: 'Inter', sans-serif; background: var(--bg); color: var(--text); margin: 0; display: flex; flex-direction: column; align-items: center; overflow-x: hidden; }
        .hidden { display: none !important; }

        .card { background: var(--card-bg); padding: 24px; border-radius: 20px; box-shadow: 0 10px 15px -3px rgba(0,0,0,0.05); margin: 10px; width: 90%; max-width: 450px; box-sizing: border-box; }
        input, select { margin: 8px 0; padding: 14px; width: 100%; border: 1px solid #e2e8f0; border-radius: 12px; box-sizing: border-box; font-size: 16px; }
        button { padding: 14px; cursor: pointer; background: var(--primary); color: white; border: none; border-radius: 12px; font-weight: 700; transition: 0.2s; }
        button:active { transform: scale(0.96); }
        .btn-outline { background: #fff; border: 1px solid #e2e8f0; color: #64748b; width: auto; padding: 8px 16px; font-size: 14px; }

        .header { background: rgba(255,255,255,0.85); backdrop-filter: blur(12px); width: 100%; padding: 12px 20px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #e2e8f0; position: sticky; top:0; z-index:100; box-sizing: border-box; }
        .room-card { background: white; margin-bottom: 12px; padding: 18px; border-radius: 16px; display: flex; justify-content: space-between; align-items: center; border: 1px solid #e2e8f0; cursor: pointer; }

        #canvas-wrap { width: 100%; overflow: auto; display: flex; justify-content: center; padding: 20px 0; }
        #canvas { background: white; display: block; border-radius: 12px; box-shadow: 0 4px 25px rgba(0,0,0,0.06); touch-action: none; border: 1px solid #e2e8f0; }
        
        .toolbar-wrapper { position: fixed; bottom: 0; width: 100%; background: white; border-top: 1px solid #e2e8f0; padding: 12px 0; z-index: 200; }
        .toolbar-scroll { display: flex; overflow-x: auto; padding: 0 15px; gap: 10px; -webkit-overflow-scrolling: touch; align-items: center; }
        .toolbar-scroll::-webkit-scrollbar { display: none; }
        .tool-btn { flex: 0 0 auto; min-width: 50px; height: 50px; display: flex; align-items: center; justify-content: center; font-size: 18px; background: #f1f5f9; border-radius: 14px; border: 2px solid transparent; }
        .tool-btn.active { border-color: var(--primary); background: #e0e7ff; color: var(--primary); }
        .tool-sep { width: 1px; background: #e2e8f0; height: 30px; flex: 0 0 auto; }
        
        .layer-manager { display: flex; align-items: center; gap: 6px; background: #f1f5f9; padding: 5px 12px; border-radius: 16px; flex: 0 0 auto; }
        #layer-list { display: flex; gap: 4px; }
        .layer-btn { padding: 8px 12px; font-size: 13px; border-radius: 10px; border: 1px solid #e2e8f0; background: #fff; color: #64748b; font-weight: bold; cursor: pointer; }
        .layer-btn.active { background: var(--primary); color: #fff; border-color: var(--primary); }
        .add-layer-btn { background: #e2e8f0; color: #1e293b; border-radius: 10px; padding: 8px 12px; border: none; font-weight: bold; }

        #user-list-modal { position: fixed; top: 70px; right: 20px; background: white; border-radius: 16px; box-shadow: 0 10px 30px rgba(0,0,0,0.15); z-index: 1000; width: 180px; padding: 12px; }
    </style>
</head>
<body>

    <div id="auth-page">
        <div class="card" style="margin-top: 60px;">
            <h2 style="text-align:center; color:var(--primary);">Soulkin Paint</h2>
            <input type="text" id="username" placeholder="名前">
            <input type="password" id="password" placeholder="パスワード">
            <button id="btn-action" style="width:100%;">ログイン</button>
            <p style="text-align:center; font-size:14px; margin-top:20px;">
                <span id="toggle-auth" style="color:var(--primary); cursor:pointer; font-weight:bold;">新規登録はこちら</span>
            </p>
        </div>
    </div>

    <div id="lobby-page" class="hidden">
        <div class="header">
            <span>👤 <b id="user-label"></b></span>
            <button id="btn-logout" class="btn-outline">ログアウト</button>
        </div>
        <div class="card">
            <h3 style="margin-top:0">部屋を作る</h3>
            <input type="text" id="room-name" placeholder="部屋の名前">
            <div style="display:flex; gap:10px;">
                <input type="number" id="room-width" placeholder="幅(px)" value="360">
                <input type="number" id="room-height" placeholder="高(px)" value="520">
            </div>
            <input type="password" id="room-pass" placeholder="入室パス (任意)">
            <input type="password" id="room-del-pass" placeholder="管理用パス (必須)">
            <button id="btn-create" style="width:100%">作成</button>
        </div>
        <div style="width: 95%; max-width: 500px; padding-bottom: 120px;">
            <h3 style="padding-left:10px;">キャンバス一覧</h3>
            <div id="room-list"></div>
        </div>
    </div>

    <div id="game-page" class="hidden">
        <div class="header">
            <div><b id="room-label"></b> <small id="online-count-badge" style="background:#e0e7ff; color:var(--primary); padding:4px 10px; border-radius:20px; font-weight:bold; cursor:pointer;">👤 <span id="online-count">1</span></small></div>
            <button id="btn-leave" class="btn-outline">退室</button>
        </div>
        <div id="user-list-modal" class="hidden"><div id="user-names-list"></div></div>
        <div id="canvas-wrap"><canvas id="canvas"></canvas></div>

        <div class="toolbar-wrapper">
            <div class="toolbar-scroll">
                <div class="layer-manager">
                    <div id="layer-list"></div>
                    <button class="add-layer-btn" id="btn-add-layer">+</button>
                    <button id="btn-del-layer" style="background:none; color:var(--danger); padding:5px; font-size:12px;">✖</button>
                </div>
                <div class="tool-sep"></div>
                <input type="color" id="color-picker" value="#6366f1" style="flex:0 0 50px; height:50px; border:none; border-radius:14px; padding:0;">
                <button id="btn-pen" class="tool-btn active">🖊️</button>
                <button id="btn-rainbow" class="tool-btn">🌈</button>
                <button id="btn-eraser" class="tool-btn">🧽</button>
                <button id="btn-undo" class="tool-btn">↩️</button>
                <button id="btn-clear" class="tool-btn" style="background:var(--danger); color:white; display:none;">💣</button>
                <div style="flex:0 0 auto; display:flex; align-items:center; background:#f1f5f9; padding:0 15px; border-radius:14px;">
                    <input type="range" id="size-range" min="1" max="50" value="5" style="width:80px; margin:0;">
                </div>
            </div>
        </div>
    </div>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
        import { getFirestore, collection, addDoc, getDocs, query, where, onSnapshot, orderBy } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";
        import { getDatabase, ref, push, onValue, set, onDisconnect, remove, off } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-database.js";

        const firebaseConfig = {
            apiKey: "AIzaSyCwhHspaG94goiCIjVj3h-Un5pBK3JTjMU",
            authDomain: "soulkin-aa3b7.firebaseapp.com",
            databaseURL: "https://soulkin-aa3b7-default-rtdb.firebaseio.com",
            projectId: "soulkin-aa3b7",
            storageBucket: "soulkin-aa3b7.firebasestorage.app",
            messagingSenderId: "358331064206",
            appId: "1:358331064206:web:d7760ea0919259418a4edf"
        };

        const app = initializeApp(firebaseConfig);
        const db = getFirestore(app);
        const rtdb = getDatabase(app);

        let myName = "", isSignup = false, activeRoomId = null;
        let mode = 'pen', activeLayer = "1", hue = 0, currentStroke = [], undoStack = [];
        let roomLayers = ["1"];

        // --- 認証 & 自動ログイン ---
        window.onload = () => {
            const savedName = localStorage.getItem('soulkin_user');
            if (savedName) {
                loginSuccess(savedName);
            }
        };

        document.getElementById('toggle-auth').onclick = () => { isSignup = !isSignup; document.getElementById('btn-action').innerText = isSignup ? "登録" : "ログイン"; };
        
        document.getElementById('btn-action').onclick = async () => {
            const name = document.getElementById('username').value.trim();
            const pass = document.getElementById('password').value.trim();
            if(!name || !pass) return;
            const usersRef = collection(db, "users");
            if(isSignup) {
                await addDoc(usersRef, { name, pass });
                alert("登録しました"); isSignup = false;
            } else {
                const snap = await getDocs(query(usersRef, where("name", "==", name), where("pass", "==", pass)));
                if(snap.empty) return alert("名前かパスワードが違います");
                localStorage.setItem('soulkin_user', name);
                loginSuccess(name);
            }
        };

        function loginSuccess(name) {
            myName = name;
            document.getElementById('auth-page').classList.add('hidden');
            document.getElementById('lobby-page').classList.remove('hidden');
            document.getElementById('user-label').innerText = name;
            loadRooms();
        }

        document.getElementById('btn-logout').onclick = () => {
            localStorage.removeItem('soulkin_user');
            location.reload();
        };

        // --- ロビー ---
        function loadRooms() {
            onSnapshot(query(collection(db, "rooms"), orderBy("createdAt", "desc")), (snap) => {
                const list = document.getElementById('room-list'); list.innerHTML = "";
                snap.forEach(d => {
                    const r = d.data();
                    const card = document.createElement('div'); card.className = "room-card";
                    card.innerHTML = `<div><b>${r.name}</b><small>ホスト: ${r.host} (${r.w}x${r.h})</small></div><button class="btn-outline">入室</button>`;
                    card.onclick = () => window.joinRoom(d.id, r.name, r.pass, r.host, r.w, r.h);
                    list.appendChild(card);
                });
            });
        }

        document.getElementById('btn-create').onclick = async () => {
            const name = document.getElementById('room-name').value;
            const w = document.getElementById('room-width').value || 360;
            const h = document.getElementById('room-height').value || 520;
            const doc = await addDoc(collection(db, "rooms"), { name, w, h, pass: document.getElementById('room-pass').value, delPass: document.getElementById('room-del-pass').value, host: myName, createdAt: Date.now() });
            window.joinRoom(doc.id, name, "", myName, w, h);
        };

        // --- ゲームメイン ---
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        let drawing = false, lx = 0, ly = 0;

        window.joinRoom = (id, name, pass, host, w, h) => {
            if(pass && prompt("パスワード") !== pass) return;
            activeRoomId = id; canvas.width = w; canvas.height = h;
            document.getElementById('lobby-page').classList.add('hidden');
            document.getElementById('game-page').classList.remove('hidden');
            document.getElementById('room-label').innerText = name;
            document.getElementById('btn-clear').style.display = (host === myName) ? "flex" : "none";

            const pRef = ref(rtdb, `rooms/${id}/users/${myName}`);
            set(pRef, true); onDisconnect(pRef).remove();
            
            onValue(ref(rtdb, `draws/${id}`), (snap) => {
                ctx.clearRect(0, 0, canvas.width, canvas.height);
                const allData = [];
                const layersInDb = [];
                snap.forEach(layerSnap => {
                    layersInDb.push(layerSnap.key);
                    layerSnap.forEach(strokeSnap => { allData.push({ ...strokeSnap.val(), layerId: layerSnap.key }); });
                });
                allData.sort((a, b) => a.layerId.localeCompare(b.layerId, undefined, {numeric: true})).forEach(d => {
                    ctx.beginPath(); ctx.strokeStyle = d.color; ctx.lineWidth = d.size;
                    ctx.lineCap = "round"; ctx.moveTo(d.x1, d.y1); ctx.lineTo(d.x2, d.y2); ctx.stroke();
                });
                if(layersInDb.length > 0) {
                    roomLayers = layersInDb.sort((a,b) => a.localeCompare(b, undefined, {numeric:true}));
                    renderLayerUI();
                }
            });

            onValue(ref(rtdb, `rooms/${id}/users`), s => document.getElementById('online-count').innerText = s.val() ? Object.keys(s.val()).length : 1);
        };

        function renderLayerUI() {
            const list = document.getElementById('layer-list');
            list.innerHTML = "";
            roomLayers.forEach(l => {
                const btn = document.createElement('button');
                btn.className = `layer-btn ${activeLayer === l ? 'active' : ''}`;
                btn.innerText = `L${l}`;
                btn.onclick = () => { activeLayer = l; renderLayerUI(); };
                list.appendChild(btn);
            });
        }

        document.getElementById('btn-add-layer').onclick = () => {
            const nextNum = roomLayers.length > 0 ? Math.max(...roomLayers.map(Number)) + 1 : 1;
            set(ref(rtdb, `draws/${activeRoomId}/${nextNum}/_init`), true);
            activeLayer = String(nextNum);
        };

        document.getElementById('btn-del-layer').onclick = () => {
            if(roomLayers.length <= 1) return;
            if(confirm(`レイヤー L${activeLayer} を消しますか？`)) {
                remove(ref(rtdb, `draws/${activeRoomId}/${activeLayer}`));
                activeLayer = roomLayers[0];
            }
        };

        const getPos = (e) => {
            const rect = canvas.getBoundingClientRect();
            const cx = e.touches ? e.touches[0].clientX : e.clientX;
            const cy = e.touches ? e.touches[0].clientY : e.clientY;
            return [(cx - rect.left) * (canvas.width / rect.width), (cy - rect.top) * (canvas.height / rect.height)];
        };

        const start = (e) => { drawing = true; [lx, ly] = getPos(e); currentStroke = []; };
        const move = (e) => {
            if(!drawing) return;
            const [x, y] = getPos(e);
            let color = mode === 'eraser' ? "#ffffff" : (mode === 'rainbow' ? `hsl(${hue},100%,50%)` : document.getElementById('color-picker').value);
            if(mode === 'rainbow') hue += 5;
            const size = document.getElementById('size-range').value;
            const newRef = push(ref(rtdb, `draws/${activeRoomId}/${activeLayer}`), { x1:lx, y1:ly, x2:x, y2:y, color, size });
            currentStroke.push({id: newRef.key, layer: activeLayer});
            [lx, ly] = [x, y];
            if(e.cancelable) e.preventDefault();
        };

        canvas.addEventListener('mousedown', start);
        window.addEventListener('mousemove', move);
        window.addEventListener('mouseup', () => { if(drawing && currentStroke.length > 0) undoStack.push(currentStroke); drawing = false; });
        canvas.addEventListener('touchstart', start);
        canvas.addEventListener('touchmove', move, { passive: false });
        canvas.addEventListener('touchend', () => { if(drawing && currentStroke.length > 0) undoStack.push(currentStroke); drawing = false; });

        document.getElementById('btn-pen').onclick = () => { mode = 'pen'; updateBtn('btn-pen'); };
        document.getElementById('btn-rainbow').onclick = () => { mode = 'rainbow'; updateBtn('btn-rainbow'); };
        document.getElementById('btn-eraser').onclick = () => { mode = 'eraser'; updateBtn('btn-eraser'); };
        function updateBtn(id) { document.querySelectorAll('.tool-btn').forEach(b => b.classList.remove('active')); document.getElementById(id).classList.add('active'); }

        document.getElementById('btn-undo').onclick = () => {
            const lastStroke = undoStack.pop();
            if(lastStroke) lastStroke.forEach(item => remove(ref(rtdb, `draws/${activeRoomId}/${item.layer}/${item.id}`)));
        };
        
        document.getElementById('btn-clear').onclick = () => { if(confirm("全消去？")) set(ref(rtdb, `draws/${activeRoomId}`), null); };

        // 退室ボタン（リロードせずにロビーに戻る）
        document.getElementById('btn-leave').onclick = () => {
            if(activeRoomId) {
                // Firebaseの監視を停止してユーザーリストから削除
                off(ref(rtdb, `draws/${activeRoomId}`));
                off(ref(rtdb, `rooms/${activeRoomId}/users`));
                remove(ref(rtdb, `rooms/${activeRoomId}/users/${myName}`));
            }
            activeRoomId = null;
            document.getElementById('game-page').classList.add('hidden');
            document.getElementById('lobby-page').classList.remove('hidden');
        };
    </script>
</body>
</html>
