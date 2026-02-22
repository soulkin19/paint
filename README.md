<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Soulkin Paint Ultra</title>
    <style>
        :root { --primary: #6366f1; --danger: #f43f5e; --bg: #f8fafc; --text: #1e293b; --card-bg: #ffffff; }
        body { font-family: 'Inter', -apple-system, sans-serif; background: var(--bg); color: var(--text); margin: 0; display: flex; flex-direction: column; align-items: center; overflow-x: hidden; }
        .hidden { display: none !important; }

        /* 共通カード */
        .card { background: var(--card-bg); padding: 24px; border-radius: 20px; box-shadow: 0 10px 15px -3px rgba(0,0,0,0.05); margin: 10px; width: 90%; max-width: 450px; box-sizing: border-box; }
        
        /* 入力・ボタン */
        input, select { margin: 8px 0; padding: 14px; width: 100%; border: 1px solid #e2e8f0; border-radius: 12px; box-sizing: border-box; font-size: 16px; transition: border-color 0.2s; }
        input:focus { border-color: var(--primary); outline: none; }
        button { padding: 14px; cursor: pointer; background: var(--primary); color: white; border: none; border-radius: 12px; font-weight: 700; transition: all 0.2s; font-size: 16px; }
        button:active { transform: scale(0.96); opacity: 0.9; }
        .btn-outline { background: #fff; border: 1px solid #e2e8f0; color: #64748b; width: auto; padding: 8px 16px; font-size: 14px; }

        /* ヘッダー */
        .header { background: rgba(255,255,255,0.85); backdrop-filter: blur(12px); width: 100%; padding: 12px 20px; display: flex; justify-content: space-between; align-items: center; border-bottom: 1px solid #e2e8f0; position: sticky; top:0; z-index:100; box-sizing: border-box; }
        
        /* 刷新された部屋リストUI */
        .room-container { width: 95%; max-width: 500px; padding-bottom: 120px; }
        .room-card { background: white; margin-bottom: 12px; padding: 18px; border-radius: 16px; display: flex; justify-content: space-between; align-items: center; border: 1px solid #e2e8f0; transition: all 0.2s; cursor: pointer; }
        .room-card:hover { border-color: var(--primary); box-shadow: 0 4px 12px rgba(99, 102, 241, 0.1); }
        .room-card b { font-size: 1.1rem; display: block; margin-bottom: 4px; color: var(--text); }
        .room-card small { color: #94a3b8; font-size: 0.85rem; }
        .room-status { text-align: right; }
        .badge-lock { color: #f59e0b; font-size: 1.2rem; margin-right: 8px; }

        /* キャンバス領域 */
        #canvas { background: white; display: block; margin: 15px auto; border-radius: 12px; box-shadow: 0 4px 25px rgba(0,0,0,0.06); touch-action: none; border: 1px solid #e2e8f0; }
        
        /* モバイル対応ツールバー (横スクロール可能) */
        .toolbar-wrapper { position: fixed; bottom: 0; width: 100%; background: white; border-top: 1px solid #e2e8f0; padding: 12px 0; z-index: 200; }
        .toolbar-scroll { display: flex; overflow-x: auto; padding: 0 15px; gap: 10px; -webkit-overflow-scrolling: touch; scrollbar-width: none; /* Firefox用 */ }
        .toolbar-scroll::-webkit-scrollbar { display: none; /* Chrome用 */ }
        .tool-btn { flex: 0 0 auto; min-width: 50px; height: 50px; display: flex; align-items: center; justify-content: center; font-size: 20px; background: #f1f5f9; border-radius: 14px; border: 2px solid transparent; transition: 0.2s; }
        .tool-btn.active { border-color: var(--primary); background: #e0e7ff; }
        .tool-sep { width: 1px; background: #e2e8f0; margin: 0 5px; flex: 0 0 auto; }
        
        /* スライダー調整 */
        .size-box { display: flex; align-items: center; gap: 10px; background: #f1f5f9; padding: 0 15px; border-radius: 14px; flex: 0 0 auto; font-size: 14px; font-weight: 600; }

        /* 参加者リストポップアップ */
        #user-list-modal { position: fixed; top: 70px; right: 20px; background: white; border-radius: 16px; box-shadow: 0 10px 30px rgba(0,0,0,0.15); border: 1px solid #e2e8f0; z-index: 1000; width: 180px; padding: 12px; }
    </style>
</head>
<body>

    <div id="auth-page" style="margin-top: 60px;">
        <div class="card">
            <h2 style="text-align:center; margin:0 0 20px 0; color:var(--primary);">Soulkin Paint</h2>
            <input type="text" id="username" placeholder="ユーザー名">
            <input type="password" id="password" placeholder="パスワード">
            <button id="btn-action" style="width:100%;">ログインする</button>
            <p style="text-align:center; font-size:14px; margin-top:20px; color:#64748b;">
                はじめてですか？ <span id="toggle-auth" style="color:var(--primary); cursor:pointer; font-weight:bold;">新規登録</span>
            </p>
        </div>
    </div>

    <div id="lobby-page" class="hidden">
        <div class="header">
            <span>👤 <b id="user-label"></b></span>
            <button onclick="location.reload()" class="btn-outline">ログアウト</button>
        </div>
        <div class="card">
            <h3 style="margin-top:0">新しいキャンバスを作る</h3>
            <input type="text" id="room-name" placeholder="部屋の名前">
            <input type="password" id="room-pass" placeholder="パスワード (任意)">
            <input type="password" id="room-del-pass" placeholder="削除用パスワード (必須)">
            <button id="btn-create" style="width:100%">作成して入室</button>
        </div>
        <div class="room-container">
            <h3 style="padding-left:10px; margin-bottom:15px; color:#475569;">キャンバス一覧</h3>
            <div id="room-list"></div>
        </div>
    </div>

    <div id="game-page" class="hidden">
        <div class="header">
            <div>
                <b id="room-label"></b> 
                <small id="online-count-badge" style="background:#e0e7ff; color:var(--primary); padding:4px 10px; border-radius:20px; font-weight:bold; cursor:pointer; margin-left:8px;">👤 <span id="online-count">1</span></small>
            </div>
            <button id="btn-leave" class="btn-outline">退室</button>
        </div>

        <div id="user-list-modal" class="hidden">
            <div style="font-size:12px; color:#94a3b8; margin-bottom:8px; font-weight:600;">オンライン</div>
            <div id="user-names-list"></div>
        </div>

        <canvas id="canvas" width="360" height="520"></canvas>

        <div class="toolbar-wrapper">
            <div class="toolbar-scroll">
                <input type="color" id="color-picker" value="#6366f1" style="flex:0 0 50px; height:50px; padding:0; border:none; border-radius:14px;">
                <button id="btn-pen" class="tool-btn active">🖊️</button>
                <button id="btn-rainbow" class="tool-btn">🌈</button>
                <button id="btn-stamp" class="tool-btn">⭐</button>
                <button id="btn-eraser" class="tool-btn">🧽</button>
                <div class="tool-sep"></div>
                <button id="btn-undo" class="tool-btn">↩️</button>
                <button id="btn-clear" class="tool-btn" style="background:var(--danger); color:white; display:none;">💣</button>
                <div class="size-box">
                    <span>太さ</span>
                    <input type="range" id="size-range" min="1" max="50" value="5" style="width:80px; margin:0;">
                </div>
            </div>
        </div>
    </div>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
        import { getFirestore, collection, addDoc, getDocs, query, where, onSnapshot, orderBy } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";
        import { getDatabase, ref, push, onValue, set, onDisconnect, remove } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-database.js";

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
        let mode = 'pen', hue = 0, currentStroke = [], undoStack = [];

        // 認証
        document.getElementById('toggle-auth').onclick = () => {
            isSignup = !isSignup;
            document.getElementById('btn-action').innerText = isSignup ? "新規作成する" : "ログインする";
        };

        document.getElementById('btn-action').onclick = async () => {
            const name = document.getElementById('username').value.trim();
            const pass = document.getElementById('password').value.trim();
            if(!name || !pass) return alert("入力してね");
            const usersRef = collection(db, "users");
            if(isSignup) {
                const snap = await getDocs(query(usersRef, where("name", "==", name)));
                if(!snap.empty) return alert("その名前は使われています");
                await addDoc(usersRef, { name, pass });
                alert("登録完了！"); isSignup = false; document.getElementById('btn-action').innerText = "ログインする";
            } else {
                const snap = await getDocs(query(usersRef, where("name", "==", name), where("pass", "==", pass)));
                if(snap.empty) return alert("名前かパスワードが違います");
                myName = name;
                document.getElementById('auth-page').classList.add('hidden');
                document.getElementById('lobby-page').classList.remove('hidden');
                document.getElementById('user-label').innerText = name;
                loadRooms();
            }
        };

        // 部屋リスト
        function loadRooms() {
            onSnapshot(query(collection(db, "rooms"), orderBy("createdAt", "desc")), (snap) => {
                const list = document.getElementById('room-list');
                list.innerHTML = "";
                snap.forEach(d => {
                    const r = d.data();
                    const card = document.createElement('div');
                    card.className = "room-card";
                    card.innerHTML = `
                        <div>
                            <b>${r.name}</b>
                            <small>ホスト: ${r.host}</small>
                        </div>
                        <div class="room-status">
                            ${r.pass ? '<span class="badge-lock">🔒</span>' : ''}
                            <button class="btn-outline">入室</button>
                        </div>
                    `;
                    card.onclick = () => window.joinRoom(d.id, r.name, r.pass, r.host);
                    list.appendChild(card);
                });
            });
        }

        document.getElementById('btn-create').onclick = async () => {
            const name = document.getElementById('room-name').value;
            const delPass = document.getElementById('room-del-pass').value;
            if(!name || !delPass) return alert("必須項目を埋めてね");
            const doc = await addDoc(collection(db, "rooms"), { name, pass: document.getElementById('room-pass').value, delPass, host: myName, createdAt: Date.now() });
            window.joinRoom(doc.id, name, "", myName);
        };

        // お絵描き
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        let drawing = false, lx = 0, ly = 0;

        window.joinRoom = (id, name, pass, host) => {
            if(pass && pass !== "" && prompt("パスワード") !== pass) return alert("拒否されました");
            activeRoomId = id;
            document.getElementById('lobby-page').classList.add('hidden');
            document.getElementById('game-page').classList.remove('hidden');
            document.getElementById('room-label').innerText = name;
            document.getElementById('btn-clear').style.display = (host === myName) ? "flex" : "none";

            const pRef = ref(rtdb, `rooms/${id}/users/${myName}`);
            set(pRef, true); onDisconnect(pRef).remove();
            onValue(ref(rtdb, `rooms/${id}/users`), s => {
                const users = s.val() ? Object.keys(s.val()) : [];
                document.getElementById('online-count').innerText = users.length;
                const list = document.getElementById('user-names-list');
                list.innerHTML = "";
                users.forEach(u => {
                    const div = document.createElement('div');
                    div.style.padding = "5px 0";
                    div.style.fontSize = "14px";
                    div.innerText = (u === host ? "👑 " : "") + u;
                    list.appendChild(div);
                });
            });

            onValue(ref(rtdb, `draws/${id}`), (snap) => {
                ctx.clearRect(0, 0, canvas.width, canvas.height);
                snap.forEach(c => {
                    const d = c.val();
                    if(d.type === 'stamp') { ctx.font = `${d.size*2}px serif`; ctx.fillText(d.txt, d.x, d.y); }
                    else { ctx.beginPath(); ctx.strokeStyle = d.color; ctx.lineWidth = d.size; ctx.lineCap = "round"; ctx.moveTo(d.x1, d.y1); ctx.lineTo(d.x2, d.y2); ctx.stroke(); }
                });
            });
        };

        document.getElementById('online-count-badge').onclick = () => document.getElementById('user-list-modal').classList.toggle('hidden');

        const getPos = (e) => {
            const rect = canvas.getBoundingClientRect();
            const cx = e.touches ? e.touches[0].clientX : e.clientX;
            const cy = e.touches ? e.touches[0].clientY : e.clientY;
            return [(cx - rect.left) * (canvas.width / rect.width), (cy - rect.top) * (canvas.height / rect.height)];
        };

        const start = (e) => {
            drawing = true; [lx, ly] = getPos(e);
            currentStroke = [];
            if(mode === 'stamp') {
                const id = push(ref(rtdb, `draws/${activeRoomId}`), { type:'stamp', txt:'⭐', x:lx, y:ly, size:document.getElementById('size-range').value }).key;
                undoStack.push([id]); drawing = false;
            }
        };

        const move = (e) => {
            if(!drawing) return;
            const [x, y] = getPos(e);
            let color = document.getElementById('color-picker').value;
            if(mode === 'rainbow') { color = `hsl(${hue}, 100%, 50%)`; hue += 5; }
            if(mode === 'eraser') color = "#ffffff";
            const size = document.getElementById('size-range').value;
            const newRef = push(ref(rtdb, `draws/${activeRoomId}`), { x1:lx, y1:ly, x2:x, y2:y, color, size });
            currentStroke.push(newRef.key); [lx, ly] = [x, y];
            if(e.cancelable) e.preventDefault();
        };

        canvas.addEventListener('mousedown', start);
        window.addEventListener('mousemove', move);
        window.addEventListener('mouseup', () => { if(drawing && currentStroke.length > 0) undoStack.push(currentStroke); drawing = false; });
        canvas.addEventListener('touchstart', start);
        canvas.addEventListener('touchmove', move, { passive: false });
        canvas.addEventListener('touchend', () => { if(drawing && currentStroke.length > 0) undoStack.push(currentStroke); drawing = false; });

        const setMode = (m, bid) => { mode = m; document.querySelectorAll('.tool-btn').forEach(b => b.classList.remove('active')); document.getElementById(bid).classList.add('active'); };
        document.getElementById('btn-pen').onclick = () => setMode('pen', 'btn-pen');
        document.getElementById('btn-rainbow').onclick = () => setMode('rainbow', 'btn-rainbow');
        document.getElementById('btn-stamp').onclick = () => setMode('stamp', 'btn-stamp');
        document.getElementById('btn-eraser').onclick = () => setMode('eraser', 'btn-eraser');
        document.getElementById('btn-undo').onclick = () => { const ids = undoStack.pop(); if(ids) ids.forEach(id => remove(ref(rtdb, `draws/${activeRoomId}/${id}`))); };
        document.getElementById('btn-clear').onclick = () => { if(confirm("全消去しますか？")) set(ref(rtdb, `draws/${activeRoomId}`), null); };
        document.getElementById('btn-leave').onclick = () => { set(ref(rtdb, `rooms/${activeRoomId}/users/${myName}`), null); document.getElementById('game-page').classList.add('hidden'); document.getElementById('lobby-page').classList.remove('hidden'); document.getElementById('user-list-modal').classList.add('hidden'); };
    </script>
</body>
</html>
