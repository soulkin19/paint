<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Soulkin Paint - オンライン共有お絵描き</title>
    <style>
        :root { --primary: #4a90e2; --bg: #f4f7f6; }
        body { font-family: 'Helvetica Neue', Arial, sans-serif; background: var(--bg); margin: 0; display: flex; flex-direction: column; align-items: center; }
        .container { width: 100%; max-width: 800px; padding: 20px; text-align: center; }
        #canvas { background: white; border-radius: 8px; box-shadow: 0 4px 12px rgba(0,0,0,0.1); cursor: crosshair; touch-action: none; }
        .card { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.05); margin-bottom: 20px; }
        .hidden { display: none; }
        button { background: var(--primary); color: white; border: none; padding: 10px 20px; border-radius: 5px; cursor: pointer; font-weight: bold; }
        button:hover { opacity: 0.9; }
        input { padding: 8px; border: 1px solid #ddd; border-radius: 4px; margin: 5px; }
        .room-item { list-style: none; padding: 10px; border-bottom: 1px solid #eee; cursor: pointer; display: flex; justify-content: space-between; }
        .room-item:hover { background: #f9f9f9; }
        .controls { margin-top: 10px; display: flex; gap: 10px; justify-content: center; align-items: center; }
    </style>
</head>
<body>

<div class="container">
    <h1>🎨 Soulkin Paint</h1>

    <div id="auth-section" class="card">
        <p>遊ぶにはログインが必要です</p>
        <button id="btn-login">Googleでログイン</button>
    </div>

    <div id="lobby-section" class="card hidden">
        <h3>部屋を作る</h3>
        <input type="text" id="room-name" placeholder="部屋の名前">
        <input type="password" id="room-pass" placeholder="パスワード（任意）">
        <button id="btn-create">作成して入室</button>
        <hr>
        <h3>部屋を探す</h3>
        <div id="room-list"></div>
    </div>

    <div id="game-section" class="hidden">
        <div class="card">
            <h2 id="current-room-title"></h2>
            <canvas id="canvas" width="700" height="450"></canvas>
            <div class="controls">
                <input type="color" id="color-picker" value="#4a90e2">
                <button id="btn-clear" style="background:#e74c3c">全消去</button>
                <button id="btn-leave" style="background:#95a5a6">退出</button>
            </div>
        </div>
    </div>
</div>

<script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
    import { getAuth, signInWithPopup, GoogleAuthProvider, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-auth.js";
    import { getFirestore, collection, addDoc, onSnapshot, query, orderBy } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";
    import { getDatabase, ref, push, onChildAdded, set } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-database.js";

    // ユーザー提供のConfig
    const firebaseConfig = {
        apiKey: "AIzaSyCwhHspaG94goiCIjVj3h-Un5pBK3JTjMU",
        authDomain: "soulkin-aa3b7.firebaseapp.com",
        databaseURL: "https://soulkin-aa3b7-default-rtdb.firebaseio.com",
        projectId: "soulkin-aa3b7",
        storageBucket: "soulkin-aa3b7.firebasestorage.app",
        messagingSenderId: "358331064206",
        appId: "1:358331064206:web:d7760ea0919259418a4edf",
        measurementId: "G-S5Z8TTWYE2"
    };

    const app = initializeApp(firebaseConfig);
    const auth = getAuth(app);
    const db = getFirestore(app);
    const rtdb = getDatabase(app);

    let user = null;
    let activeRoomId = null;
    const canvas = document.getElementById('canvas');
    const ctx = canvas.getContext('2d');
    let isDrawing = false;

    // --- 認証ロジック ---
    document.getElementById('btn-login').onclick = () => signInWithPopup(auth, new GoogleAuthProvider());
    
    onAuthStateChanged(auth, (u) => {
        if (u) {
            user = u;
            document.getElementById('auth-section').classList.add('hidden');
            document.getElementById('lobby-section').classList.remove('hidden');
            listenRooms();
        }
    });

    // --- 部屋管理 ---
    document.getElementById('btn-create').onclick = async () => {
        const name = document.getElementById('room-name').value;
        const pass = document.getElementById('room-pass').value;
        if(!name) return alert("部屋名を入力してください");
        
        const docRef = await addDoc(collection(db, "rooms"), {
            name, pass, host: user.uid, createdAt: Date.now()
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
                const div = document.createElement('div');
                div.className = 'room-item card';
                div.innerHTML = `<span>${r.name} ${r.pass ? '🔒' : ''}</span><button>入る</button>`;
                div.onclick = () => {
                    if(r.pass && prompt("パスワードを入力") !== r.pass) return alert("違います");
                    joinRoom(doc.id, r.name);
                };
                list.appendChild(div);
            });
        });
    }

    // --- お絵描きロジック ---
    function joinRoom(id, title) {
        activeRoomId = id;
        document.getElementById('lobby-section').classList.add('hidden');
        document.getElementById('game-section').classList.remove('hidden');
        document.getElementById('current-room-title').textContent = title;

        const drawRef = ref(rtdb, `draws/${id}`);
        onChildAdded(drawRef, (snap) => {
            const d = snap.val();
            drawLine(d.x1, d.y1, d.x2, d.y2, d.color);
        });
    }

    let lastX = 0, lastY = 0;
    canvas.onmousedown = (e) => { isDrawing = true; [lastX, lastY] = [e.offsetX, e.offsetY]; };
    canvas.onmousemove = (e) => {
        if (!isDrawing) return;
        const color = document.getElementById('color-picker').value;
        const { offsetX: x, offsetY: y } = e;
        
        push(ref(rtdb, `draws/${activeRoomId}`), { x1: lastX, y1: lastY, x2: x, y2: y, color });
        [lastX, lastY] = [x, y];
    };
    window.onmouseup = () => isDrawing = false;

    function drawLine(x1, y1, x2, y2, color) {
        ctx.beginPath();
        ctx.strokeStyle = color;
        ctx.lineWidth = 3;
        ctx.lineCap = "round";
        ctx.moveTo(x1, y1);
        ctx.lineTo(x2, y2);
        ctx.stroke();
    }

    document.getElementById('btn-clear').onclick = () => {
        if(confirm("キャンバスをリセットしますか？")) set(ref(rtdb, `draws/${activeRoomId}`), null).then(() => {
            ctx.clearRect(0, 0, canvas.width, canvas.height);
        });
    };
    document.getElementById('btn-leave').onclick = () => location.reload();

</script>
</body>
</html>
