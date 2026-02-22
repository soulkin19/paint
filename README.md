<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>Soulkin Paint - Firestore User Management</title>
    <style>
        body { font-family: sans-serif; background: #f4f7f6; display: flex; flex-direction: column; align-items: center; }
        .card { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); margin: 20px; width: 350px; text-align: center; }
        .hidden { display: none; }
        input { margin: 5px; padding: 10px; width: 80%; }
        button { padding: 10px 20px; cursor: pointer; background: #4a90e2; color: white; border: none; border-radius: 4px; }
        #canvas { background: white; border: 2px solid #333; cursor: crosshair; }
    </style>
</head>
<body>

    <h1>🎨 Soulkin Paint (Firestore DB版)</h1>

    <div id="auth-section" class="card">
        <h3 id="auth-title">ログイン</h3>
        <input type="text" id="auth-email" placeholder="メールアドレス"><br>
        <input type="password" id="auth-pass" placeholder="パスワード"><br>
        <button id="btn-login">ログイン</button>
        <button id="btn-signup" style="background:#2ecc71">新規登録</button>
        <p id="auth-error" style="color:red; font-size:12px;"></p>
    </div>

    <div id="lobby-section" class="hidden">
        <div class="card">
            <p>ようこそ、<span id="display-user"></span> さん</p>
            <button onclick="location.reload()">ログアウト</button>
            <hr>
            <h3>部屋を作る</h3>
            <input type="text" id="room-name" placeholder="部屋名"><br>
            <input type="password" id="room-key" placeholder="部屋の鍵（パスワード）"><br>
            <button id="btn-create">作成</button>
        </div>
        <div class="card">
            <h3>部屋一覧</h3>
            <div id="room-list"></div>
        </div>
    </div>

    <div id="game-section" class="hidden">
        <h2 id="current-room"></h2>
        <canvas id="canvas" width="600" height="400"></canvas>
        <div>
            <input type="color" id="color-picker">
            <button id="btn-clear">全消去</button>
            <button onclick="location.reload()">退出</button>
        </div>
    </div>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
        import { getFirestore, collection, addDoc, getDocs, query, where, onSnapshot, orderBy } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";
        import { getDatabase, ref, push, onChildAdded, set } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-database.js";

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

        let currentUser = null;

        // --- 1. Firestoreを使った独自ログインロジック ---
        
        // 新規登録
        document.getElementById('btn-signup').onclick = async () => {
            const email = document.getElementById('auth-email').value;
            const pass = document.getElementById('auth-pass').value;
            if(!email || !pass) return alert("入力してください");

            // 重複チェック
            const q = query(collection(db, "users"), where("email", "==", email));
            const snap = await getDocs(q);
            if(!snap.empty) return alert("既に登録されています");

            await addDoc(collection(db, "users"), { email: email, password: pass });
            alert("登録完了！ログインしてください");
        };

        // ログイン
        document.getElementById('btn-login').onclick = async () => {
            const email = document.getElementById('auth-email').value;
            const pass = document.getElementById('auth-pass').value;

            const q = query(collection(db, "users"), 
                        where("email", "==", email), 
                        where("password", "==", pass));
            
            const snap = await getDocs(q);
            if(snap.empty) {
                document.getElementById('auth-error').textContent = "メールかパスワードが違います";
            } else {
                currentUser = { email: email };
                startApp();
            }
        };

        function startApp() {
            document.getElementById('auth-section').classList.add('hidden');
            document.getElementById('lobby-section').classList.remove('hidden');
            document.getElementById('display-user').textContent = currentUser.email;
            listenRooms();
        }

        // --- 2. 部屋管理 (Firestore) ---
        document.getElementById('btn-create').onclick = async () => {
            const name = document.getElementById('room-name').value;
            const key = document.getElementById('room-key').value;
            const docRef = await addDoc(collection(db, "rooms"), {
                name: name, key: key, createdAt: Date.now()
            });
            joinRoom(docRef.id, name);
        };

        function listenRooms() {
            const q = query(collection(db, "rooms"), orderBy("createdAt", "desc"));
            onSnapshot(q, (snap) => {
                const list = document.getElementById('room-list');
                list.innerHTML = "";
                snap.forEach(doc => {
                    const r = doc.data();
                    const btn = document.createElement('button');
                    btn.textContent = `${r.name} ${r.key ? '🔒' : ''}`;
                    btn.style.display = "block";
                    btn.style.width = "100%";
                    btn.onclick = () => {
                        if(r.key && prompt("パスワード") !== r.key) return alert("×");
                        joinRoom(doc.id, r.name);
                    };
                    list.appendChild(btn);
                });
            });
        }

        // --- 3. お絵描き同期 (Realtime Database) ---
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        let activeRoomId = null;
        let drawing = false;

        function joinRoom(id, name) {
            activeRoomId = id;
            document.getElementById('lobby-section').classList.add('hidden');
            document.getElementById('game-section').classList.remove('hidden');
            document.getElementById('current-room').textContent = "部屋: " + name;

            onChildAdded(ref(rtdb, `draws/${id}`), (snap) => {
                const d = snap.val();
                draw(d.x1, d.y1, d.x2, d.y2, d.color);
            });
        }

        let lx, ly;
        canvas.onmousedown = (e) => { drawing = true; [lx, ly] = [e.offsetX, e.offsetY]; };
        canvas.onmousemove = (e) => {
            if(!drawing) return;
            const {offsetX: x, offsetY: y} = e;
            const color = document.getElementById('color-picker').value;
            push(ref(rtdb, `draws/${activeRoomId}`), {x1:lx, y1:ly, x2:x, y2:y, color:color});
            [lx, ly] = [x, y];
        };
        window.onmouseup = () => drawing = false;

        function draw(x1, y1, x2, y2, color) {
            ctx.beginPath(); ctx.strokeStyle = color; ctx.lineWidth = 3;
            ctx.moveTo(x1, y1); ctx.lineTo(x2, y2); ctx.stroke();
        }

        document.getElementById('btn-clear').onclick = () => set(ref(rtdb, `draws/${activeRoomId}`), null);
    </script>
</body>
</html>
