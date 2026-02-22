<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Soulkin Paint Pro+</title>
    <style>
        :root { --primary: #6366f1; --primary-hover: #4f46e5; --danger: #ef4444; --bg: #f8fafc; --text: #1e293b; }
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background: var(--bg); color: var(--text); margin: 0; display: flex; flex-direction: column; align-items: center; }
        
        /* 共通カードデザイン */
        .card { background: white; padding: 24px; border-radius: 16px; box-shadow: 0 10px 15px -3px rgba(0,0,0,0.1); margin: 10px; width: 90%; max-width: 450px; }
        .hidden { display: none !important; }
        
        /* 入力フォーム */
        input, select { margin: 8px 0; padding: 12px; width: 100%; border: 1px solid #e2e8f0; border-radius: 8px; box-sizing: border-box; font-size: 16px; }
        button { padding: 12px 20px; cursor: pointer; background: var(--primary); color: white; border: none; border-radius: 8px; font-weight: 600; font-size: 16px; transition: 0.2s; width: 100%; }
        button:hover { background: var(--primary-hover); }
        
        /* ナビゲーション */
        .header { background: white; width: 100%; padding: 15px 20px; display: flex; justify-content: space-between; align-items: center; box-shadow: 0 1px 3px rgba(0,0,0,0.1); box-sizing: border-box; }
        
        /* 部屋リスト */
        .room-item { background: white; margin: 10px 0; padding: 15px; border-radius: 12px; display: flex; justify-content: space-between; align-items: center; border: 1px solid #e2e8f0; }
        .room-info b { font-size: 1.1em; color: var(--primary); }
        .room-info div { font-size: 0.85em; color: #64748b; margin-top: 4px; }
        .btn-delete { background: #fee2e2; color: var(--danger); width: auto; padding: 8px; font-size: 12px; margin-left: 10px; }

        /* お絵描き画面 */
        #game-page { width: 100%; max-width: 800px; text-align: center; }
        #canvas-container { position: relative; margin: 10px; background: white; line-height: 0; border-radius: 12px; overflow: hidden; box-shadow: 0 4px 6px -1px rgba(0,0,0,0.1); }
        canvas { background: white; touch-action: none; border: 1px solid #ddd; }
        
        /* ツールバー */
        .toolbar { display: flex; flex-wrap: wrap; gap: 10px; justify-content: center; padding: 15px; background: #f1f5f9; border-top: 1px solid #e2e8f0; position: sticky; bottom: 0; width: 100%; box-sizing: border-box; }
        .tool { background: white; padding: 5px 10px; border-radius: 8px; display: flex; align-items: center; gap: 8px; border: 1px solid #e2e8f0; font-size: 14px; }
        .active-tool { border: 2px solid var(--primary); }
    </style>
</head>
<body>

    <div id="auth-page" style="margin-top: 50px;">
        <div class="card">
            <h2 id="auth-title" style="margin-top:0;">🎨 Soulkin Paint</h2>
            <input type="text" id="username" placeholder="ユーザー名">
            <input type="password" id="password" placeholder="パスワード">
            <button id="btn-action">ログイン</button>
            <p style="font-size: 0.85em; margin-top: 20px;">
                <span id="toggle-auth" style="color:var(--primary); cursor:pointer; font-weight:600;">新規登録はこちら</span>
            </p>
        </div>
    </div>

    <div id="lobby-page" class="hidden">
        <div class="header">
            <span>👤 <b id="user-label"></b></span>
            <button onclick="location.reload()" style="width:auto; padding:8px 15px; background:#64748b;">終了</button>
        </div>
        
        <div class="card">
            <h3 style="margin-top:0;">部屋を作る</h3>
            <input type="text" id="room-name" placeholder="部屋の名前">
            <input type="password" id="room-pass" placeholder="入室パスワード (空なら公開)">
            <input type="password" id="room-del-pass" placeholder="削除用パスワード (必須)">
            <button id="btn-create">作成して入室</button>
        </div>

        <div style="width: 90%; max-width: 450px;">
            <h3 style="padding-left:10px;">公開中の部屋</h3>
            <div id="room-list"></div>
        </div>
    </div>

    <div id="game-page" class="hidden">
        <div class="header">
            <span id="room-label" style="font-weight:bold;"></span>
            <button id="btn-leave" style="width:auto; padding:8px 15px; background:#64748b;">退室</button>
        </div>
        
        <div id="canvas-container">
            <canvas id="canvas" width="375" height="550"></canvas>
        </div>

        <div class="toolbar">
            <div class="tool">🎨 <input type="color" id="color-picker" value="#6366f1" style="width:40px; height:30px; padding:0; border:none;"></div>
            <div class="tool">太さ <input type="range" id="size-range" min="1" max="50" value="5" style="width:70px;"></div>
            <button id="btn-pen" style="width:auto; background:#475569;">ペン</button>
            <button id="btn-eraser" style="width:auto; background:#94a3b8;">消しゴム</button>
            <button id="btn-clear" style="width:auto; background:var(--danger);">全消去</button>
        </div>
    </div>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
        import { getFirestore, collection, addDoc, getDocs, query, where, onSnapshot, orderBy, deleteDoc, doc } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";
        import { getDatabase, ref, push, onValue, set } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-database.js";

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

        // --- 認証機能 ---
        document.getElementById('toggle-auth').onclick = () => {
            isSignup = !isSignup;
            document.getElementById('auth-title').innerText = isSignup ? "新規作成" : "🎨 Soulkin Paint";
            document.getElementById('btn-action').innerText = isSignup ? "アカウントを作成" : "ログイン";
        };

        document.getElementById('btn-action').onclick = async () => {
            const name = document.getElementById('username').value.trim();
            const pass = document.getElementById('password').value.trim();
            if(!name || !pass) return alert("名前とパスワードを入力してください");
            
            const usersRef = collection(db, "users");
            if(isSignup) {
                const q = query(usersRef, where("name", "==", name));
                const snap = await getDocs(q);
                if(!snap.empty) return alert("その名前はすでに存在します");
                await addDoc(usersRef, { name, pass });
                alert("登録しました！ログインしてください");
                isSignup = false;
                document.getElementById('btn-action').innerText = "ログイン";
            } else {
                const q = query(usersRef, where("name", "==", name), where("pass", "==", pass));
                const snap = await getDocs(q);
                if(snap.empty) return alert("ログイン情報が正しくありません");
                
                myName = name;
                document.getElementById('auth-page').classList.add('hidden');
                document.getElementById('lobby-page').classList.remove('hidden');
                document.getElementById('user-label').innerText = name;
                loadRooms();
            }
        };

        // --- ロビー & 部屋管理 ---
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
                            <div>作成者: ${r.host}</div>
                        </div>
                        <div style="display:flex; align-items:center;">
                            <button style="width:auto; padding:8px 12px;" onclick="window.joinRoom('${d.id}', '${r.name}', '${r.pass}')">入室</button>
                            <button class="btn-delete" onclick="window.deleteRoom('${d.id}', '${r.delPass}')">削除</button>
                        </div>
                    `;
                    list.appendChild(item);
                });
            });
        }

        document.getElementById('btn-create').onclick = async () => {
            const name = document.getElementById('room-name').value;
            const delPass = document.getElementById('room-del-pass').value;
            if(!name || !delPass) return alert("部屋名と削除パスワードは必須です");
            
            const docRef = await addDoc(collection(db, "rooms"), {
                name, pass: document.getElementById('room-pass').value,
                delPass: delPass, host: myName, createdAt: Date.now()
            });
            window.joinRoom(docRef.id, name);
        };

        window.deleteRoom = async (id, correctPass) => {
            const input = prompt("削除用パスワードを入力してください");
            if(input === correctPass) {
                await deleteDoc(doc(db, "rooms", id));
                set(ref(rtdb, `draws/${id}`), null); // お絵描きデータも消去
                alert("削除しました");
            } else if(input !== null) {
                alert("パスワードが違います");
            }
        };

        // --- お絵描き機能 ---
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        let drawing = false, lx = 0, ly = 0;

        window.joinRoom = (id, name, pass) => {
            if(pass && pass !== "undefined" && pass !== "" && prompt("部屋のパスワードを入力") !== pass) return alert("パスワードが違います");
            
            activeRoomId = id;
            document.getElementById('lobby-page').classList.add('hidden');
            document.getElementById('game-page').classList.remove('hidden');
            document.getElementById('room-label').innerText = name;
            
            // 既存データの読み込み
            onValue(ref(rtdb, `draws/${id}`), (snap) => {
                ctx.clearRect(0, 0, canvas.width, canvas.height);
                snap.forEach(child => {
                    const d = child.val();
                    draw(d.x1, d.y1, d.x2, d.y2, d.color, d.size);
                });
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
        canvas.addEventListener('touchend', () => drawing = false);

        function draw(x1, y1, x2, y2, color, size) {
            ctx.beginPath(); ctx.strokeStyle = color; ctx.lineWidth = size;
            ctx.lineCap = "round"; ctx.moveTo(x1, y1); ctx.lineTo(x2, y2); ctx.stroke();
        }

        document.getElementById('btn-pen').onclick = () => { isEraser = false; };
        document.getElementById('btn-eraser').onclick = () => { isEraser = true; };
        document.getElementById('btn-clear').onclick = () => { if(confirm("全消去しますか？")) set(ref(rtdb, `draws/${activeRoomId}`), null); };
        document.getElementById('btn-leave').onclick = () => {
            document.getElementById('game-page').classList.add('hidden');
            document.getElementById('lobby-page').classList.remove('hidden');
        };

    </script>
</body>
</html>
