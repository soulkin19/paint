<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Soulkin Paint Pro</title>
    <style>
        :root { --primary: #6366f1; --danger: #f43f5e; --success: #10b981; --bg: #f1f5f9; --card-bg: #ffffff; --text: #1e293b; }
        body { font-family: 'Inter', sans-serif; background: var(--bg); color: var(--text); margin: 0; overflow: hidden; height: 100vh; display: flex; flex-direction: column; }
        .hidden { display: none !important; }

        /* ヘッダー */
        .header { background: #fff; padding: 10px 16px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #e2e8f0; z-index: 100; }
        .btn-outline { background: #fff; border: 1px solid #e2e8f0; color: #64748b; padding: 6px 12px; border-radius: 8px; cursor: pointer; font-size: 13px; }

        /* キャンバスエリア */
        #viewport { flex: 1; position: relative; overflow: hidden; background: #cbd5e1; display: flex; align-items: center; justify-content: center; touch-action: none; }
        #canvas-container { position: relative; box-shadow: 0 10px 30px rgba(0,0,0,0.2); background: white; transform-origin: center; }
        canvas { display: block; image-rendering: pixelated; }

        /* サイドパネル（レイヤー） */
        #layer-panel { position: absolute; right: 10px; top: 70px; width: 180px; background: rgba(255,255,255,0.9); backdrop-filter: blur(10px); border-radius: 12px; padding: 10px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); z-index: 50; max-height: 60vh; overflow-y: auto; }
        .layer-item { display: flex; flex-direction: column; gap: 4px; padding: 8px; border-bottom: 1px solid #eee; font-size: 12px; }
        .layer-item.active { background: #e0e7ff; border-radius: 8px; }
        .layer-row { display: flex; align-items: center; gap: 5px; }
        .layer-name { flex: 1; font-weight: bold; cursor: pointer; }

        /* ツールバー */
        .toolbar { background: #fff; border-top: 1px solid #e2e8f0; padding: 8px; display: flex; flex-direction: column; gap: 8px; z-index: 100; }
        .tool-row { display: flex; overflow-x: auto; gap: 8px; padding-bottom: 4px; align-items: center; }
        .tool-row::-webkit-scrollbar { display: none; }
        .tool-btn { flex: 0 0 auto; width: 44px; height: 44px; display: flex; align-items: center; justify-content: center; border-radius: 10px; border: 1px solid #e2e8f0; background: #f8fafc; cursor: pointer; font-size: 20px; transition: 0.2s; }
        .tool-btn.active { background: var(--primary); color: white; border-color: var(--primary); }
        .tool-sep { width: 1px; height: 24px; background: #e2e8f0; flex-shrink: 0; }

        /* チャット */
        #chat-box { position: absolute; left: 10px; top: 70px; width: 200px; pointer-events: none; z-index: 50; }
        .chat-msg { background: rgba(0,0,0,0.6); color: white; padding: 4px 10px; border-radius: 10px; margin-bottom: 4px; font-size: 12px; animation: fadeOut 5s forwards; }
        @keyframes fadeOut { 0% { opacity:1; } 80% { opacity:1; } 100% { opacity:0; } }

        /* ログイン・ロビーカード */
        .overlay-page { position: fixed; inset: 0; background: var(--bg); display: flex; justify-content: center; align-items: center; z-index: 1000; overflow-y: auto; }
        .card { background: #fff; padding: 24px; border-radius: 20px; width: 90%; max-width: 400px; box-shadow: 0 20px 25px -5px rgba(0,0,0,0.1); }
        input { width: 100%; padding: 12px; margin: 8px 0; border: 1px solid #ddd; border-radius: 10px; box-sizing: border-box; }
        .room-card { background: #fff; border: 1px solid #e2e8f0; padding: 15px; border-radius: 12px; margin-bottom: 10px; display: flex; justify-content: space-between; align-items: center; }
    </style>
</head>
<body>

    <div id="auth-page" class="overlay-page">
        <div class="card">
            <h2 style="text-align:center; color:var(--primary);">Soulkin Paint Pro</h2>
            <input type="text" id="username" placeholder="名前">
            <input type="password" id="password" placeholder="パスワード">
            <button id="btn-action" style="width:100%; padding:14px; background:var(--primary); color:white; border:none; border-radius:10px; font-weight:bold;">ログイン</button>
            <p style="text-align:center; font-size:14px; cursor:pointer;" onclick="isSignup=!isSignup; this.innerText=isSignup?'ログインへ戻る':'新規登録はこちら'">新規登録はこちら</p>
        </div>
    </div>

    <div id="lobby-page" class="overlay-page hidden">
        <div style="width:90%; max-width:500px; padding: 20px 0;">
            <div class="card" style="margin-bottom:20px;">
                <h3>部屋を作る</h3>
                <input type="text" id="room-name" placeholder="部屋の名前">
                <div style="display:flex; gap:10px;">
                    <input type="number" id="room-w" value="800" placeholder="幅">
                    <input type="number" id="room-h" value="800" placeholder="高">
                </div>
                <input type="password" id="room-del-pass" placeholder="削除用パスワード">
                <button id="btn-create" style="width:100%; padding:12px; background:var(--success); color:white; border:none; border-radius:10px;">作成して入室</button>
            </div>
            <div id="room-list"></div>
            <button onclick="localStorage.removeItem('soulkin_user'); location.reload()" class="btn-outline" style="width:100%; margin-top:20px;">ログアウト</button>
        </div>
    </div>

    <div id="game-page" class="hidden" style="display:flex; flex-direction:column; height:100vh;">
        <div class="header">
            <div><b id="room-label"></b> <small id="online-count">1</small></div>
            <div style="display:flex; gap:8px;">
                <button id="btn-save" class="btn-outline">💾 保存</button>
                <button id="btn-leave" class="btn-outline">退室</button>
            </div>
        </div>

        <div id="viewport">
            <div id="chat-box"></div>
            <div id="canvas-container">
                <canvas id="canvas"></canvas>
            </div>
        </div>

        <div id="layer-panel">
            <div style="display:flex; justify-content:space-between; margin-bottom:8px;">
                <span style="font-weight:bold; font-size:12px;">レイヤー</span>
                <button id="btn-add-layer" style="border:none; background:var(--primary); color:white; border-radius:4px; padding:2px 8px;">+</button>
            </div>
            <div id="layer-list"></div>
        </div>

        <div class="toolbar">
            <div class="tool-row">
                <input type="color" id="color-picker" value="#6366f1" style="width:44px; height:44px; border:none; padding:0; background:none;">
                <div class="tool-sep"></div>
                <button id="btn-pen" class="tool-btn active">🖊️</button>
                <button id="btn-eraser" class="tool-btn">🧽</button>
                <button id="btn-dropper" class="tool-btn">🧪</button>
                <button id="btn-rainbow" class="tool-btn">🌈</button>
                <div class="tool-sep"></div>
                <button id="btn-undo" class="tool-btn">↩️</button>
                <button id="btn-clear" class="tool-btn" style="color:var(--danger)">💣</button>
                <div class="tool-sep"></div>
                <input type="text" id="chat-input" placeholder="チャット..." style="width:120px; margin:0; padding:8px;">
            </div>
            <div class="tool-row">
                <span style="font-size:12px; min-width:30px;">太さ</span>
                <input type="range" id="size-range" min="1" max="100" value="5" style="flex:1;">
                <span style="font-size:12px; min-width:30px;">透明</span>
                <input type="range" id="opacity-range" min="1" max="100" value="100" style="flex:1;">
            </div>
        </div>
    </div>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
        import { getFirestore, collection, addDoc, getDocs, query, where, onSnapshot, orderBy, doc, deleteDoc } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";
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

        let myName = localStorage.getItem('soulkin_user') || "";
        let isSignup = false, activeRoomId = null;
        let mode = 'pen', activeLayer = "1", hue = 0, undoStack = [];
        let roomLayers = {}, layerOrder = ["1"];
        let zoom = 1, offsetX = 0, offsetY = 0;

        // --- 認証系 ---
        if(myName) loginSuccess(myName);
        document.getElementById('btn-action').onclick = async () => {
            const n = document.getElementById('username').value, p = document.getElementById('password').value;
            if(!n || !p) return;
            if(isSignup) { await addDoc(collection(db,"users"),{name:n,pass:p}); alert("登録完了"); }
            else {
                const s = await getDocs(query(collection(db,"users"),where("name","==",n),where("pass","==",p)));
                if(s.empty) return alert("認証失敗");
                localStorage.setItem('soulkin_user', n); loginSuccess(n);
            }
        };
        function loginSuccess(n) { myName = n; document.getElementById('auth-page').classList.add('hidden'); document.getElementById('lobby-page').classList.remove('hidden'); loadRooms(); }

        // --- ロビー ---
        function loadRooms() {
            onSnapshot(query(collection(db,"rooms"),orderBy("createdAt","desc")), s => {
                const list = document.getElementById('room-list'); list.innerHTML = "";
                s.forEach(d => {
                    const r = d.data();
                    const div = document.createElement('div'); div.className="room-card";
                    div.innerHTML = `<div><b>${r.name}</b><br><small>${r.w}x${r.h} by ${r.host}</small></div>
                        <div><button class="btn-outline" onclick="window.joinRoom('${d.id}','${r.name}',${r.w},${r.h},'${r.host}')">入室</button>
                        <button onclick="window.delRoom('${d.id}','${r.delPass}')" style="border:none; background:none;">🗑️</button></div>`;
                    list.appendChild(div);
                });
            });
        }
        window.delRoom = async (id, p) => { if(prompt("削除パス")===p) { await deleteDoc(doc(db,"rooms",id)); remove(ref(rtdb,`draws/${id}`)); } };
        document.getElementById('btn-create').onclick = async () => {
            const n = document.getElementById('room-name').value, w = document.getElementById('room-w').value, h = document.getElementById('room-h').value, dp = document.getElementById('room-del-pass').value;
            if(!n || !dp) return alert("入力不足");
            const d = await addDoc(collection(db,"rooms"),{name:n,w,h,delPass:dp,host:myName,createdAt:Date.now()});
            window.joinRoom(d.id, n, w, h, myName);
        };

        // --- エディタ本体 ---
        const canvas = document.getElementById('canvas'), ctx = canvas.getContext('2d', {willReadFrequently: true});
        let drawing = false, lx, ly;

        window.joinRoom = (id, name, w, h, host) => {
            activeRoomId = id; canvas.width = w; canvas.height = h;
            document.getElementById('lobby-page').classList.add('hidden');
            document.getElementById('game-page').classList.remove('hidden');
            document.getElementById('room-label').innerText = name;
            
            // ユーザー参加通知
            const pRef = ref(rtdb, `rooms/${id}/users/${myName}`);
            set(pRef, true); onDisconnect(pRef).remove();
            onValue(ref(rtdb, `rooms/${id}/users`), s => document.getElementById('online-count').innerText = s.val() ? Object.keys(s.val()).length : 1);

            // 描画同期 & レイヤー管理
            onValue(ref(rtdb, `draws/${id}`), snap => {
                ctx.clearRect(0,0,w,h);
                const data = snap.val() || {};
                const layers = Object.keys(data).sort((a,b) => a.localeCompare(b, undefined, {numeric:true}));
                layerOrder = layers;
                layers.forEach(lId => {
                    const lData = data[lId];
                    if(!lData || lData.hidden) return;
                    ctx.globalAlpha = (lData.opacity || 100) / 100;
                    Object.keys(lData).forEach(k => {
                        const d = lData[k]; if(!d.x1) return;
                        ctx.beginPath(); ctx.strokeStyle = d.c; ctx.lineWidth = d.s; ctx.lineCap = "round";
                        ctx.moveTo(d.x1, d.y1); ctx.lineTo(d.x2, d.y2); ctx.stroke();
                    });
                });
                ctx.globalAlpha = 1.0;
                renderLayerUI(data);
            });

            // チャット同期
            onValue(ref(rtdb, `chats/${id}`), s => {
                const val = s.val(); if(!val) return;
                const last = Object.values(val).pop();
                showChat(last.u, last.m);
            });
        };

        function renderLayerUI(data) {
            const list = document.getElementById('layer-list'); list.innerHTML = "";
            [...layerOrder].reverse().forEach(lId => {
                const lInfo = data[lId] || {};
                const item = document.createElement('div');
                item.className = `layer-item ${activeLayer === lId ? 'active' : ''}`;
                item.innerHTML = `
                    <div class="layer-row">
                        <span class="layer-name" onclick="window.setActiveLayer('${lId}')">L${lId}</span>
                        <button onclick="window.toggleLayer('${lId}', ${!lInfo.hidden})">${lInfo.hidden?'👁️‍🗨️':'👁️'}</button>
                    </div>
                    <input type="range" min="0" max="100" value="${lInfo.opacity||100}" onchange="window.setLayerOpacity('${lId}', this.value)">
                `;
                list.appendChild(item);
            });
        }
        window.setActiveLayer = (id) => { activeLayer = id; };
        window.toggleLayer = (id, hide) => set(ref(rtdb, `draws/${activeRoomId}/${id}/hidden`), hide);
        window.setLayerOpacity = (id, val) => set(ref(rtdb, `draws/${activeRoomId}/${id}/opacity`), parseInt(val));
        document.getElementById('btn-add-layer').onclick = () => {
            const next = layerOrder.length > 0 ? Math.max(...layerOrder.map(Number)) + 1 : 1;
            set(ref(rtdb, `draws/${activeRoomId}/${next}/_init`), true);
            activeLayer = String(next);
        };
        document.getElementById('btn-del-layer').onclick = () => { if(layerOrder.length>1) remove(ref(rtdb, `draws/${activeRoomId}/${activeLayer}`)); };

        // --- 描画ロジック ---
        const getPos = (e) => {
            const rect = canvas.getBoundingClientRect();
            const cx = e.touches ? e.touches[0].clientX : e.clientX;
            const cy = e.touches ? e.touches[0].clientY : e.clientY;
            return [(cx - rect.left) * (canvas.width / rect.width), (cy - rect.top) * (canvas.height / rect.height)];
        };

        const start = (e) => {
            if(e.touches && e.touches.length > 1) return; // ズーム中は描画しない
            drawing = true; [lx, ly] = getPos(e);
            if(mode === 'dropper') {
                const [x, y] = getPos(e);
                const p = ctx.getImageData(x, y, 1, 1).data;
                const hex = "#" + ("000000" + ((p[0] << 16) | (p[1] << 8) | p[2]).toString(16)).slice(-6);
                document.getElementById('color-picker').value = hex;
                mode = 'pen'; updateBtn('btn-pen');
                drawing = false;
            }
            undoStack.push([]);
        };

        const move = (e) => {
            if(!drawing || (e.touches && e.touches.length > 1)) return;
            const [x, y] = getPos(e);
            let color = mode === 'eraser' ? "#ffffff" : (mode === 'rainbow' ? `hsl(${hue},100%,50%)` : document.getElementById('color-picker').value);
            if(mode === 'rainbow') hue += 5;
            const op = document.getElementById('opacity-range').value;
            // 簡易不透明度（100未満なら少し色を薄くして送信）
            const finalColor = op < 100 ? color + Math.floor(op*2.55).toString(16).padStart(2,'0') : color;
            
            const r = push(ref(rtdb, `draws/${activeRoomId}/${activeLayer}`), { x1:lx, y1:ly, x2:x, y2:y, c:finalColor, s:document.getElementById('size-range').value });
            undoStack[undoStack.length-1].push({l:activeLayer, k:r.key});
            [lx, ly] = [x, y];
            if(e.cancelable) e.preventDefault();
        };

        canvas.addEventListener('mousedown', start);
        window.addEventListener('mousemove', move);
        window.addEventListener('mouseup', () => drawing = false);
        canvas.addEventListener('touchstart', start);
        canvas.addEventListener('touchmove', move, { passive: false });
        canvas.addEventListener('touchend', () => drawing = false);

        // --- ツール機能 ---
        document.getElementById('btn-pen').onclick = () => { mode = 'pen'; updateBtn('btn-pen'); };
        document.getElementById('btn-eraser').onclick = () => { mode = 'eraser'; updateBtn('btn-eraser'); };
        document.getElementById('btn-dropper').onclick = () => { mode = 'dropper'; updateBtn('btn-dropper'); };
        document.getElementById('btn-rainbow').onclick = () => { mode = 'rainbow'; updateBtn('btn-rainbow'); };
        function updateBtn(id) { document.querySelectorAll('.tool-btn').forEach(b => b.classList.remove('active')); document.getElementById(id).classList.add('active'); }
        
        document.getElementById('btn-undo').onclick = () => {
            const last = undoStack.pop();
            if(last) last.forEach(i => remove(ref(rtdb, `draws/${activeRoomId}/${i.l}/${i.k}`)));
        };
        document.getElementById('btn-clear').onclick = () => { if(confirm("全消去？")) set(ref(rtdb, `draws/${activeRoomId}`), null); };
        document.getElementById('btn-save').onclick = () => {
            const link = document.createElement('a');
            link.download = 'soulkin_paint.png';
            link.href = canvas.toDataURL();
            link.click();
        };

        // --- チャット ---
        const chatInput = document.getElementById('chat-input');
        chatInput.onkeypress = (e) => {
            if(e.key === 'Enter' && chatInput.value) {
                push(ref(rtdb, `chats/${activeRoomId}`), { u: myName, m: chatInput.value });
                chatInput.value = "";
            }
        };
        function showChat(u, m) {
            const box = document.getElementById('chat-box');
            const d = document.createElement('div'); d.className = 'chat-msg';
            d.innerText = `${u}: ${m}`;
            box.appendChild(d);
            setTimeout(() => d.remove(), 5000);
        }

        // --- ズーム・移動 (Viewport) ---
        const container = document.getElementById('canvas-container');
        window.addEventListener('wheel', e => {
            if(e.ctrlKey) {
                e.preventDefault(); zoom *= e.deltaY > 0 ? 0.9 : 1.1;
                container.style.transform = `scale(${zoom})`;
            }
        }, {passive:false});

        // --- ショートカット ---
        window.addEventListener('keydown', e => {
            if(document.activeElement.tagName === 'INPUT') return;
            if(e.ctrlKey && e.key === 'z') document.getElementById('btn-undo').click();
            if(e.key === 'b') document.getElementById('btn-pen').click();
            if(e.key === 'e') document.getElementById('btn-eraser').click();
            if(e.key === 'i') document.getElementById('btn-dropper').click();
        });

        document.getElementById('btn-leave').onclick = () => location.reload();
    </script>
</body>
</html>
