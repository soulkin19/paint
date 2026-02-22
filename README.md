<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Soulkin Paint - Simple Auth</title>
    <style>
        :root { --main: #4a90e2; }
        body { font-family: sans-serif; background: #f0f2f5; display: flex; flex-direction: column; align-items: center; margin: 0; }
        .card { background: white; padding: 25px; border-radius: 12px; box-shadow: 0 4px 20px rgba(0,0,0,0.08); margin: 20px; width: 320px; text-align: center; }
        .hidden { display: none; }
        input { margin: 8px 0; padding: 12px; width: 90%; border: 1px solid #ddd; border-radius: 6px; }
        button { padding: 12px; width: 100%; cursor: pointer; background: var(--main); color: white; border: none; border-radius: 6px; font-weight: bold; margin-top: 10px; }
        #canvas { background: white; border: 2px solid #333; cursor: crosshair; touch-action: none; border-radius: 8px; }
        .nav { background: #fff; width: 100%; padding: 10px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); display: flex; justify-content: space-around; align-items: center; }
    </style>
</head>
<body>

    <div id="auth-page">
        <div class="card">
            <h2 id="auth-title">ログイン</h2>
            <input type="text" id="username" placeholder="名前 (Username)">
            <input type="password" id="password" placeholder="パスワード">
            <button id="btn-action">ログインする</button>
            <p style="font-size: 0.8em; color: #666; margin-top: 15px;">
                アカウントがない場合は？ <span id="toggle-auth" style="color:var(--main); cursor:pointer; text-decoration:underline;">新規登録</span>
            </p>
            <p id="msg" style="color:red; font-size: 12px;"></p>
        </div>
    </div>

    <div id="lobby-page" class="hidden">
        <div class="nav">
            <span>👤 <b id="user-label"></b></span>
            <button onclick="location.reload()" style="width:auto; padding:5px 10px; background:#999;">ログアウト</button>
        </div>
        <div class="card">
            <h3>新しい部屋を作る</h3>
            <input type="text" id="room-name" placeholder="部屋の名前">
            <input type="password" id="room-pass" placeholder="部屋の鍵 (任意)">
            <button id="btn-create">作成</button>
        </div>
        <div class="card">
            <h3>参加できる部屋</h3>
            <div id="room-list"></div>
        </div>
    </div>

    <div id="game-page" class="hidden">
        <div class="nav">
            <span id="room-label"></span>
            <button id="btn-leave" style="width:auto; padding:5px 10px; background:#999;">退室</button>
        </div>
        <div style="margin: 20px;">
            <canvas id="canvas" width="650" height="400"></canvas>
            <div style="margin-top:10px; display:flex; gap:10px; justify-content:center;">
                <input type="color" id="color-picker" style="width:50px; padding:0; height:40px;">
                <button id="btn-clear" style="width:auto; background:#e74c3c;">画面をクリア</button>
            </div>
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

        let myName = "";
        let isSignup = false;

        // --- ログイン・登録の切り替え ---
        const toggleBtn = document.getElementById('toggle-auth');
        toggleBtn.onclick = () => {
            isSignup = !isSignup;
            document.getElementById('auth-title').innerText = isSignup ? "新規登録" : "ログイン";
            document.getElementById('btn-action').innerText = isSignup ? "アカウントを作成" : "ログインする";
            toggleBtn.innerText = isSignup ? "ログインはこちら" : "新規登録";
        };

        // --- 認証実行 ---
        document.getElementById('btn-action').onclick = async () => {
            const name = document.getElementById('username').value.trim();
            const pass = document.getElementById('password').value.trim();
            const msg = document.getElementById('msg');
            if(!name || !pass) return alert("名前とパスワードを入力してね");

            const usersRef = collection(db, "users");

            if (isSignup) {
                // 登録処理
                const q = query(usersRef, where("name", "==", name));
                const snap = await getDocs(q);
                if(!snap.empty) return alert("その名前はすでに使われています");
                
                await addDoc(usersRef, { name: name, pass: pass });
                alert("登録完了！そのままログインします。");
                doLogin(name);
            } else {
                // ログイン処理
                const q = query(usersRef, where("name", "==", name), where("pass", "==", pass));
                const snap = await getDocs(q);
                if(snap.empty) {
                    msg.innerText = "名前かパスワードが違います";
                } else {
                    doLogin(name);
                }
            }
        };

        function doLogin(name) {
            myName = name;
            document.getElementById('auth-page').classList.add('hidden');
            document.getElementById('lobby-page').classList.remove('hidden');
            document.getElementById('user-label').innerText = name;
            loadRooms();
        }

        // --- 部屋管理 ---
        document.getElementById('btn-create').onclick = async () => {
            const rName = document.getElementById('room-name').value;
            const rPass = document.getElementById('room-pass').value;
            if(!rName) return alert("部屋の名前を決めてね");
            
            const doc = await addDoc(collection(db, "rooms"), {
                name: rName, pass: rPass, host: myName, createdAt: Date.now()
            });
            joinRoom(doc.id, rName);
        };

        function loadRooms() {
            const q = query(collection(db, "rooms"), orderBy("createdAt", "desc"));
            onSnapshot(q, (snap) => {
                const list = document.getElementById('room-list');
                list.innerHTML = "";
                snap.forEach(doc => {
                    const r = doc.data();
                    const b = document.createElement('button');
                    b.innerHTML = `${r.name} ${r.pass ? '🔒' : ''}`;
                    b.style.marginBottom = "5px";
                    b.onclick = () => {
                        if(r.pass && prompt("部屋のパスワードを入力") !== r.pass) return alert("鍵が違います");
                        joinRoom(doc.id, r.name);
                    };
                    list.appendChild(b);
                });
            });
        }

        // --- お絵描き ---
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        let currentRoomId = null;
        let drawing = false;

        function joinRoom(id, name) {
            currentRoomId = id;
            document.getElementById('lobby-page').classList.add('hidden');
            document.getElementById('game-page').classList.remove('hidden');
            document.getElementById('room-label').innerText = "部屋: " + name;

            onChildAdded(ref(rtdb, `draws/${id}`), (s) => {
                const d = s.val();
                draw(d.x1, d.y1, d.x2, d.y2, d.color);
            });
        }

        let lx, ly;
        canvas.onmousedown = (e) => { drawing = true; [lx, ly] = [e.offsetX, e.offsetY]; };
        canvas.onmousemove = (e) => {
            if(!drawing) return;
            const {offsetX: x, offsetY: y} = e;
            const color = document.getElementById('color-picker').value;
            push(ref(rtdb, `draws/${currentRoomId}`), {x1:lx, y1:ly, x2:x, y2:y, color});
            [lx, ly] = [x, y];
        };
        window.onmouseup = () => drawing = false;

        function draw(x1, y1, x2, y2, color) {
            ctx.beginPath(); ctx.strokeStyle = color; ctx.lineWidth = 3; ctx.lineCap = "round";
            ctx.moveTo(x1, y1); ctx.lineTo(x2, y2); ctx.stroke();
        }

        document.getElementById('btn-clear').onclick = () => {
            if(confirm("全員の画面を消しますか？")) {
                set(ref(rtdb, `draws/${currentRoomId}`), null);
                ctx.clearRect(0, 0, canvas.width, canvas.height);
            }
        };
        document.getElementById('btn-leave').onclick = () => location.reload();

    </script>
</body>
</html>
