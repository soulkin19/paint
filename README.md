<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Soulkin Paint Premium</title>
    <style>
        :root { --primary: #6366f1; --danger: #f43f5e; --bg: #f8fafc; --card: #ffffff; --text: #1e293b; }
        body { font-family: 'Inter', -apple-system, sans-serif; background: var(--bg); color: var(--text); margin: 0; display: flex; flex-direction: column; align-items: center; }
        
        .hidden { display: none !important; }
        .card { background: var(--card); padding: 24px; border-radius: 20px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.1); margin: 10px; width: 90%; max-width: 450px; border: 1px solid #e2e8f0; }
        
        /* Input & Buttons */
        input, select { margin: 8px 0; padding: 12px; width: 100%; border: 1px solid #e2e8f0; border-radius: 10px; box-sizing: border-box; font-size: 16px; outline: none; }
        input:focus { border-color: var(--primary); ring: 2px var(--primary); }
        button { padding: 12px 20px; cursor: pointer; background: var(--primary); color: white; border: none; border-radius: 10px; font-weight: 600; font-size: 16px; transition: all 0.2s; width: 100%; }
        button:active { transform: scale(0.98); }
        .btn-outline { background: transparent; border: 1px solid #cbd5e1; color: #64748b; }

        /* Header */
        .header { background: rgba(255,255,255,0.8); backdrop-filter: blur(10px); width: 100%; padding: 12px 20px; display: flex; justify-content: space-between; align-items: center; position: sticky; top: 0; z-index: 100; border-bottom: 1px solid #e2e8f0; box-sizing: border-box; }
        
        /* Room Items */
        .room-item { background: white; margin: 10px 0; padding: 16px; border-radius: 16px; display: flex; justify-content: space-between; align-items: center; border: 1px solid #e2e8f0; transition: transform 0.2s; }
        .room-item:hover { transform: translateY(-2px); border-color: var(--primary); }
        .room-info b { font-size: 1.1em; }
        .room-info div { font-size: 0.8em; color: #64748b; margin-top: 4px; }
        .count-badge { background: #e0e7ff; color: var(--primary); padding: 2px 8px; border-radius: 20px; font-weight: bold; font-size: 0.8em; }

        /* Canvas & Tools */
        #game-page { width: 100%; max-width: 900px; }
        canvas { background: white; touch-action: none; display: block; margin: 0 auto; box-shadow: 0 4px 20px rgba(0,0,0,0.08); border-radius: 4px; }
        .toolbar { display: flex; flex-wrap: wrap; gap: 8px; justify-content: center; padding: 15px; background: white; border-top: 1px solid #e2e8f0; position: fixed; bottom: 0; width: 100%; box-sizing: border-box; }
        .tool-box { display: flex; align-items: center; gap: 10px; background: #f1f5f9; padding: 6px 12px; border-radius: 12px; font-size: 14px; font-weight: bold; }
        .btn-tool { width: auto; padding: 8px 12px; font-size: 14px; }
        .active { background: #1e293b; color: white; }
    </style>
</head>
<body>

    <div id="auth-page" style="margin-top: 60px;">
        <div class="card">
            <h1 style="text-align:center; font-size: 24px; margin-bottom: 24px;">Soulkin Paint 🎨</h1>
            <input type="text" id="username" placeholder="名前を入力">
            <input type="password" id="password" placeholder="パスワードを入力">
            <button id="btn-action">ログイン</button>
            <p style="text-align:center; font-size: 14px; color: #64748b;">
                <span id="toggle-auth" style="color:var(--primary); cursor:pointer; font-weight:bold;">新規登録はこちら</span>
            </p>
        </div>
    </div>

    <div id="lobby-page" class="hidden">
        <div class="header">
            <span>👤 <b id="user-label"></b></span>
            <button onclick="location.reload()" class="btn-outline" style="width:auto; padding:6px 12px;">ログアウト</button>
        </div>
        <div class="card">
            <h3 style="margin-top:0;">部屋を新規作成</h3>
            <input type="text" id="room-name" placeholder="部屋の名前">
            <input type="password" id="room-pass" placeholder="入室パスワード (任意)">
            <input type="password" id="room-del-pass" placeholder="削除・管理用パスワード (必須)">
            <button id="btn-create">作成して入室</button>
        </div>
        <div style="width: 90%; max-width: 450px;">
            <h3>進行中のキャンバス</h3>
            <div id="room-list"></div>
        </div>
    </div>

    <div id="game-page" class="hidden">
        <div class="header">
            <div>
                <b id="room-label"></b> 
                <span class="count-badge">👤 <span id="online-count">1</span></span>
            </div>
            <button id="btn-leave" class="btn-outline" style="width:auto; padding:6px 12px;">退室</button>
        </div>
        
        <canvas id="canvas" width="375" height="550"></canvas>

        <div class="toolbar">
            <div class="tool-box">🎨 <input type="color" id="color-picker" value="#6366f1" style="width:30px; border:none; padding:0; height:24px;"></div>
            <div class="tool-box">太さ <input type="range" id="size-range" min="1" max="50" value="5" style="width:60px;"></div>
            <button id="btn-pen" class="btn-tool active">ペン</button>
            <button id="btn-eraser" class="btn-tool btn-outline">消しゴム</button>
            <button id="btn-clear" class="btn-tool" style="background:var(--danger); display:none;">全消去</button>
        </div>
    </div>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
        import { getFirestore, collection, addDoc, getDocs, query, where, onSnapshot, orderBy, deleteDoc, doc } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";
        import { getDatabase, ref, push, onValue, set, onDisconnect } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-database.js";

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

        let myName = "", isSignup = false, activeRoomId = null, isEraser = false;

        // --- 認証 ---
        document.getElementById('toggle-auth').onclick = () => {
            isSignup = !isSignup;
            document.getElementById('btn-action').innerText = isSignup ? "アカウント作成" : "ログイン";
        };

        document.getElementById('btn-action').onclick = async () => {
            const name = document.getElementById('username').value.trim();
            const pass = document.getElementById('password').value.trim();
            if(!name || !pass) return alert("入力してください");
            
            const usersRef = collection(db, "users");
            if(isSignup) {
                const q = query(usersRef, where("name", "==", name));
                const snap = await getDocs(q);
                if(!snap.empty) return alert("その名前は使用されています");
                await addDoc(usersRef, { name, pass });
                alert("登録完了！"); isSignup = false;
            } else {
                const q = query(usersRef, where("name", "==", name), where("pass", "==", pass));
                const snap = await getDocs(q);
                if(snap.empty) return alert("名前かパスワードが違います");
                myName = name;
                document.getElementById('auth-page').classList.add('hidden');
                document.getElementById('lobby-page').classList.remove('hidden');
                document.getElementById('user-label').innerText = name;
                loadRooms();
            }
        };

        // --- 部屋管理 ---
        function loadRooms() {
            onSnapshot(query(collection(db, "rooms"), orderBy("createdAt", "desc")), (snap) => {
                const list = document.getElementById('room-list');
                list.innerHTML = "";
                snap.forEach(d => {
                    const r = d.data();
                    const item = document.createElement('div');
                    item.className = "room-item";
                    item.innerHTML = `
                        <div class="room-info">
                            <b>${r.name} ${r.pass ? '🔒' : ''}</b>
                            <div>ホスト: ${r.host}</div>
                        </div>
                        <button style="width:auto; font-size:14px;" onclick="window.joinRoom('${d.id}', '${r.name}', '${r.pass}', '${r.host}')">入室</button>
                    `;
                    list.appendChild(item);
                });
            });
        }

        document.getElementById('btn-create').onclick = async () => {
            const name = document.getElementById('room-name').value;
            const delPass = document.getElementById('room-del-pass').value;
            if(!name || !delPass) return alert("部屋名と管理パスは必須です");
            const docRef = await addDoc(collection(db, "rooms"), {
                name, pass: document.getElementById('room-pass').value,
                delPass, host: myName, createdAt: Date.now()
            });
            window.joinRoom(docRef.id, name, "", myName);
        };

        // --- お絵描き & 人数カウント ---
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        let drawing = false, lx = 0, ly = 0;

        window.joinRoom = (id, name, pass, host) => {
            if(pass && pass !== "" && prompt("パスワード") !== pass) return alert("違います");
            
            activeRoomId = id;
            document.getElementById('lobby-page').classList.add('hidden');
            document.getElementById('game-page').classList.remove('hidden');
            document.getElementById('room-label').innerText = name;
            
            // 全消去ボタンの表示制御（作成者のみ）
            document.getElementById('btn-clear').style.display = (host === myName) ? "block" : "none";

            // 人数カウント
            const presenceRef = ref(rtdb, `rooms/${id}/users/${myName}`);
            set(presenceRef, true);
            onDisconnect(presenceRef).remove();

            onValue(ref(rtdb, `rooms/${id}/users`), (snap) => {
                document.getElementById('online-count').innerText = snap.size || 1;
            });

            // 線描画同期
            onValue(ref(rtdb, `draws/${id}`), (snap) => {
                ctx.clearRect(0, 0, canvas.width, canvas.height);
                snap.forEach(c => { const d = c.val(); drawLine(d.x1, d.y1, d.x2, d.y2, d.color, d.size); });
            });
        };

        const getPos = (e) => {
            const rect = canvas.getBoundingClientRect();
            const cx = e.touches ? e.touches[0].clientX : e.clientX;
            const cy = e.touches ? e.touches[0].clientY : e.clientY;
            return [(cx - rect.left) * (canvas.width / rect.width), (cy - rect.top) * (canvas.height / rect.height)];
        };

        const start = (e) => { drawing = true; [lx, ly] = getPos(e); };
        const move = (e) => {
            if(!drawing) return;
            const [x, y] = getPos(e);
            const color = isEraser ? "#ffffff" : document.getElementById('color-picker').value;
            const size = document.getElementById('size-range').value;
            push(ref(rtdb, `draws/${activeRoomId}`), { x1:lx, y1:ly, x2:x, y2:y, color, size });
            [lx, ly] = [x, y];
            if(e.cancelable) e.preventDefault();
        };

        canvas.addEventListener('mousedown', start);
        window.addEventListener('mousemove', move);
        window.addEventListener('mouseup', () => drawing = false);
        canvas.addEventListener('touchstart', start);
        canvas.addEventListener('touchmove', move, { passive: false });

        function drawLine(x1, y1, x2, y2, color, size) {
            ctx.beginPath(); ctx.strokeStyle = color; ctx.lineWidth = size;
            ctx.lineCap = "round"; ctx.moveTo(x1, y1); ctx.lineTo(x2, y2); ctx.stroke();
        }

        // ツール操作
        document.getElementById('btn-pen').onclick = () => { 
            isEraser = false; 
            document.getElementById('btn-pen').classList.add('active'); 
            document.getElementById('btn-eraser').classList.remove('active');
        };
        document.getElementById('btn-eraser').onclick = () => { 
            isEraser = true; 
            document.getElementById('btn-eraser').classList.add('active'); 
            document.getElementById('btn-pen').classList.remove('active');
        };
        document.getElementById('btn-clear').onclick = () => {
            if(confirm("【作成者権限】キャンバスをリセットしますか？")) set(ref(rtdb, `draws/${activeRoomId}`), null);
        };
        document.getElementById('btn-leave').onclick = () => {
            set(ref(rtdb, `rooms/${activeRoomId}/users/${myName}`), null); // 退室時にカウントから消去
            document.getElementById('game-page').classList.add('hidden');
            document.getElementById('lobby-page').classList.remove('hidden');
        };

    </script>
</body>
</html>
