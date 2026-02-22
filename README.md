<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Soulkin Paint Pro</title>
    <style>
        :root { --main: #4a90e2; --bg: #f0f2f5; }
        body { font-family: sans-serif; background: var(--bg); margin: 0; display: flex; flex-direction: column; align-items: center; overflow-x: hidden; }
        .card { background: white; padding: 20px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.1); margin: 10px; width: 90%; max-width: 400px; text-align: center; }
        .hidden { display: none; }
        input, select { margin: 5px 0; padding: 12px; width: 95%; border: 1px solid #ddd; border-radius: 8px; font-size: 16px; }
        button { padding: 12px; cursor: pointer; background: var(--main); color: white; border: none; border-radius: 8px; font-weight: bold; margin-top: 5px; font-size: 16px; }
        
        /* キャンバスエリア */
        #game-page { width: 100%; display: flex; flex-direction: column; align-items: center; }
        canvas { background: white; border: 2px solid #333; touch-action: none; border-radius: 8px; max-width: 95%; height: auto; }
        
        /* ツールバー */
        .toolbar { display: flex; flex-wrap: wrap; gap: 8px; justify-content: center; padding: 10px; background: #ddd; width: 100%; position: sticky; bottom: 0; }
        .tool-group { display: flex; align-items: center; gap: 5px; background: white; padding: 5px 10px; border-radius: 5px; }
        
        .nav { background: #fff; width: 100%; padding: 10px; display: flex; justify-content: space-between; align-items: center; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
        .room-item { background: white; margin: 5px; padding: 15px; border-radius: 8px; display: flex; justify-content: space-between; align-items: center; box-shadow: 0 2px 4px rgba(0,0,0,0.05); }
    </style>
</head>
<body>

    <div id="auth-page">
        <div class="card">
            <h2 id="auth-title">ログイン</h2>
            <input type="text" id="username" placeholder="名前">
            <input type="password" id="password" placeholder="パスワード">
            <button id="btn-action">ログイン</button>
            <p style="font-size: 0.8em;"><span id="toggle-auth" style="color:var(--main); cursor:pointer;">新規登録はこちら</span></p>
        </div>
    </div>

    <div id="lobby-page" class="hidden">
        <div class="nav">
            <span>👤 <b id="user-label"></b></span>
            <button onclick="location.reload()" style="background:#999; width:auto;">終了</button>
        </div>
        <div class="card">
            <h3>部屋を新しく作る</h3>
            <input type="text" id="room-name" placeholder="部屋の名前">
            <input type="password" id="room-pass" placeholder="パスワード (なしでもOK)">
            <select id="room-cat">
                <option value="自由">カテゴリ：自由</option>
                <option value="動物">カテゴリ：動物</option>
                <option value="キャラ">カテゴリ：キャラ</option>
            </select>
            <select id="room-limit">
                <option value="5">最大 5人</option>
                <option value="10">最大 10人</option>
            </select>
            <button id="btn-create">作成して入室</button>
        </div>
        <div id="room-list" style="width:100%; max-width:450px;"></div>
    </div>

    <div id="game-page" class="hidden">
        <div class="nav">
            <span id="room-label"></span>
            <button id="btn-leave" style="background:#999; width:auto;">退室</button>
        </div>
        
        <canvas id="canvas" width="375" height="500"></canvas>

        <div class="toolbar">
            <div class="tool-group">
                色 <input type="color" id="color-picker" value="#000000">
            </div>
            <div class="tool-group">
                太さ <input type="range" id="size-range" min="1" max="50" value="3" style="width:80px;">
            </div>
            <button id="btn-eraser" style="background:#555;">消しゴム</button>
            <button id="btn-pen" style="background:var(--main);">ペン</button>
            <button id="btn-clear" style="background:#e74c3c;">全消去</button>
        </div>
    </div>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
        import { getFirestore, collection, addDoc, getDocs, query, where, onSnapshot, orderBy } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";
        import { getDatabase, ref, push, onChildAdded, set, onValue } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-database.js";

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

        let myName = "", isSignup = false, activeRoomId = null, unsub = null;
        let isEraser = false;

        // --- 認証 ---
        document.getElementById('toggle-auth').onclick = () => {
            isSignup = !isSignup;
            document.getElementById('auth-title').innerText = isSignup ? "新規登録" : "ログイン";
            document.getElementById('btn-action').innerText = isSignup ? "作成" : "ログイン";
        };

        document.getElementById('btn-action').onclick = async () => {
            const name = document.getElementById('username').value.trim();
            const pass = document.getElementById('password').value.trim();
            if(!name || !pass) return alert("入力してね");
            const usersRef = collection(db, "users");

            if(isSignup) {
                const q = query(usersRef, where("name", "==", name));
                const snap = await getDocs(q);
                if(!snap.empty) return alert("その名前は使われています");
                await addDoc(usersRef, { name, pass });
                alert("登録完了");
            } else {
                const q = query(usersRef, where("name", "==", name), where("pass", "==", pass));
                const snap = await getDocs(q);
                if(snap.empty) return alert("間違いです");
            }
            myName = name;
            document.getElementById('auth-page').classList.add('hidden');
            document.getElementById('lobby-page').classList.remove('hidden');
            document.getElementById('user-label').innerText = name;
            loadRooms();
        };

        // --- ロビー ---
        function loadRooms() {
            onSnapshot(query(collection(db, "rooms"), orderBy("createdAt", "desc")), (snap) => {
                const list = document.getElementById('room-list');
                list.innerHTML = "";
                snap.forEach(doc => {
                    const r = doc.data();
                    const item = document.createElement('div');
                    item.className = "room-item";
                    item.innerHTML = `<div><b>${r.name}</b><br><small>${r.cat} | 定員${r.limit}</small></div>
                                      <button style="width:auto;">${r.pass ? '鍵付' : '入室'}</button>`;
                    item.onclick = () => {
                        if(r.pass && prompt("パスワード") !== r.pass) return alert("違うよ");
                        joinRoom(doc.id, r.name);
                    };
                    list.appendChild(item);
                });
            });
        }

        document.getElementById('btn-create').onclick = async () => {
            const name = document.getElementById('room-name').value;
            if(!name) return alert("部屋名なし");
            const doc = await addDoc(collection(db, "rooms"), {
                name, pass: document.getElementById('room-pass').value,
                cat: document.getElementById('room-cat').value,
                limit: document.getElementById('room-limit').value,
                createdAt: Date.now()
            });
            joinRoom(doc.id, name);
        };

        // --- お絵描き(メインロジック) ---
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        let drawing = false, lx = 0, ly = 0;

        function joinRoom(id, name) {
            activeRoomId = id;
            document.getElementById('lobby-page').classList.add('hidden');
            document.getElementById('game-page').classList.remove('hidden');
            document.getElementById('room-label').innerText = name;
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            // リアルタイム同期（既存の線をロード＋新規を監視）
            const roomRef = ref(rtdb, `draws/${id}`);
            // すでに描いてあるものも含む全ての線を描画
            unsub = onValue(roomRef, (snap) => {
                ctx.clearRect(0, 0, canvas.width, canvas.height);
                snap.forEach(child => {
                    const d = child.val();
                    draw(d.x1, d.y1, d.x2, d.y2, d.color, d.size);
                });
            });
        }

        // 入力イベント（マウス ＆ タッチ）
        const getPos = (e) => {
            const rect = canvas.getBoundingClientRect();
            const clientX = e.touches ? e.touches[0].clientX : e.clientX;
            const clientY = e.touches ? e.touches[0].clientY : e.clientY;
            return [ (clientX - rect.left) * (canvas.width / rect.width), (clientY - rect.top) * (canvas.height / rect.height) ];
        };

        const start = (e) => { drawing = true; [lx, ly] = getPos(e); };
        const move = (e) => {
            if(!drawing) return;
            const [x, y] = getPos(e);
            const color = isEraser ? "#ffffff" : document.getElementById('color-picker').value;
            const size = document.getElementById('size-range').value;
            
            push(ref(rtdb, `draws/${activeRoomId}`), { x1:lx, y1:ly, x2:x, y2:y, color, size });
            [lx, ly] = [x, y];
            e.preventDefault();
        };
        const end = () => drawing = false;

        canvas.addEventListener('mousedown', start);
        canvas.addEventListener('mousemove', move);
        window.addEventListener('mouseup', end);
        canvas.addEventListener('touchstart', start);
        canvas.addEventListener('touchmove', move);
        canvas.addEventListener('touchend', end);

        function draw(x1, y1, x2, y2, color, size) {
            ctx.beginPath();
            ctx.strokeStyle = color;
            ctx.lineWidth = size;
            ctx.lineCap = "round";
            ctx.moveTo(x1, y1);
            ctx.lineTo(x2, y2);
            ctx.stroke();
        }

        // ツール
        document.getElementById('btn-eraser').onclick = () => { isEraser = true; document.getElementById('btn-eraser').style.background = "#main"; };
        document.getElementById('btn-pen').onclick = () => { isEraser = false; };
        document.getElementById('btn-clear').onclick = () => { if(confirm("消去？")) set(ref(rtdb, `draws/${activeRoomId}`), null); };
        
        // 退室（ログアウトしない）
        document.getElementById('btn-leave').onclick = () => {
            activeRoomId = null;
            document.getElementById('game-page').classList.add('hidden');
            document.getElementById('lobby-page').classList.remove('hidden');
        };

    </script>
</body>
</html>
