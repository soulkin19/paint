<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Soulkin Paint</title>
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
        #canvas-wrap { width: 100%; height: 100%; overflow: hidden; background: #cbd5e1; flex-grow: 1; position: relative; touch-action: none; }
        #canvas-container { position: absolute; top: 0; left: 0; transform-origin: 0 0; will-change: transform; display: flex; justify-content: center; align-items: center; }
        #canvas { background: white; box-shadow: 0 4px 25px rgba(0,0,0,0.2); display: block; }
        #reaction-container { position: absolute; inset: 0; pointer-events: none; z-index: 500; }
        .reaction-bubble { position: absolute; bottom: 20%; left: 50%; transform: translateX(-50%); background: white; padding: 12px 24px; border-radius: 30px; box-shadow: 0 8px 20px rgba(0,0,0,0.15); font-size: 32px; animation: floatUp 2.5s ease-out forwards; display: flex; flex-direction: column; align-items: center; border: 2px solid var(--primary); }
        .reaction-user { font-size: 12px; color: var(--primary); font-weight: bold; margin-top: 4px; }
        @keyframes floatUp { 0% { opacity: 0; transform: translate(-50%, 20px); } 15% { opacity: 1; transform: translate(-50%, 0); } 80% { opacity: 1; } 100% { opacity: 0; transform: translate(-50%, -120px); } }
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
                <span id="zoom-label" style="cursor:pointer; font-size:12px; color:#64748b; background:#f1f5f9; padding:2px 8px; border-radius:10px; margin-left:5px;">1.0x</span>
            </div>
            <div style="display:flex; gap:8px;"><button id="btn-save" class="btn-outline">💾</button><button id="btn-leave" class="btn-outline">退室</button></div>
        </div>
        <div id="canvas-wrap">
            <div id="canvas-container"><canvas id="canvas"></canvas></div>
        </div>
        <div class="toolbar-wrapper">
            <div id="stamp-menu" class="hidden" style="position: absolute; bottom: 85px; left: 15px; background: white; border: 1px solid #e2e8f0; border-radius: 12px; display: flex; gap: 8px; padding: 12px; box-shadow: 0 10px 25px rgba(0,0,0,0.2); z-index: 1000;">
                <span class="emoji-opt" style="font-size:30px; cursor:pointer;">👍</span>
                <span class="emoji-opt" style="font-size:30px; cursor:pointer;">😮</span>
                <span class="emoji-opt" style="font-size:30px; cursor:pointer;">❤️</span>
                <span class="emoji-opt" style="font-size:30px; cursor:pointer;">🔥</span>
                <span class="emoji-opt" style="font-size:30px; cursor:pointer;">🎨</span>
                <span class="emoji-opt" style="font-size:30px; cursor:pointer;">✅</span>
            </div>
            <div class="toolbar-scroll">
                <button id="btn-lock" class="tool-btn hidden">🔓</button>
                <button id="btn-open-emoji" class="tool-btn">💬</button>
                <div class="layer-box"><div id="layer-list" style="display:flex; gap:4px;"></div><button id="btn-add-layer" style="border:none; background:#ddd; border-radius:8px; padding:8px; cursor:pointer;">＋</button></div>
                <input type="color" id="color-picker" value="#6366f1" style="width:48px; height:48px; border:none; padding:0; flex-shrink:0;">
                <button id="btn-pen" class="tool-btn active">🖊️</button>
                <button id="btn-eraser" class="tool-btn">🧽</button>
                <button id="btn-undo" class="tool-btn">↩️</button>
                <button id="btn-clear" class="tool-btn" style="background:var(--danger); color:white; display:none;">💣</button>
                <input type="range" id="size-range" min="1" max="100" value="5" style="min-width:80px;">
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
        let myName = localStorage.getItem('soulkin_user') || "", activeRoomId = null, mode = 'pen', activeLayer = "1", roomLayers = ["1"], scale = 1.0, posX = 0, posY = 0;
        const MASTER_HASH = "8153f3e795247345f1710976537b06822452377c8e967520e7f722c2a05d894a";
        async function getHash(t) { const m = new TextEncoder().encode(t); const b = await crypto.subtle.digest('SHA-256', m); return Array.from(new Uint8Array(b)).map(x => x.toString(16).padStart(2, '0')).join(''); }
        window.onload = () => { if(myName) loginSuccess(myName); };
        document.getElementById('btn-action').onclick = async () => {
            const n = document.getElementById('username').value.trim(), p = document.getElementById('password').value.trim();
            if(!n || !p) return;
            const qU = await getDocs(query(collection(db,"users"), where("name","==",n)));
            if(qU.empty) { await addDoc(collection(db,"users"), {name:n, pass:p}); loginSuccess(n); }
            else { let m=false; qU.forEach(d=>{if(d.data().pass===p)m=true;}); if(m) loginSuccess(n); else alert("パスワード不一致"); }
        };
        function loginSuccess(n) { myName=n; localStorage.setItem('soulkin_user',n); document.getElementById('user-label').innerText=n; document.getElementById('auth-page').classList.add('hidden'); document.getElementById('lobby-page').classList.remove('hidden'); loadRooms(); }
        function loadRooms() {
            onSnapshot(query(collection(db,"rooms"), orderBy("createdAt","desc")), snap => {
                const list = document.getElementById('room-list'); list.innerHTML = "<h3>部屋一覧</h3>";
                snap.forEach(d => {
                    const r = d.data();
                    const div = document.createElement('div'); div.className = "room-card";
                    div.innerHTML = `<div><b>${r.joinPass?'🔒':''} ${r.name}</b><br><small>${r.w}x${r.h}</small></div>
                        <div style="display:flex; gap:8px;"><button class="btn-outline" onclick="window.tryJoin('${d.id}','${r.name}',${r.w},${r.h},'${r.host}','${r.joinPass||""}')">入室</button>
                        <button onclick="window.adminDelete('${d.id}','${r.delPass}')" style="color:red; border:none; background:none; font-size:20px;">🗑️</button></div>`;
                    list.appendChild(div);
                });
            });
        }
        window.adminDelete = async (id, correct) => {
            const input = prompt("削除パスワードまたはマスターキー:");
            if(!input) return;
            const h = await getHash(input);
            if(input === correct || h === MASTER_HASH) {
                if(confirm("部屋を消しますか？")) { await deleteDoc(doc(db,"rooms",id)); remove(ref(rtdb,`draws/${id}`)); remove(ref(rtdb,`rooms/${id}`)); }
            } else alert("無効");
        };
        window.tryJoin = (id,n,w,h,host,jp) => { if(jp && prompt("パスワード")!==jp) return alert("不一致"); window.joinRoom(id,n,w,h,host); };
        document.getElementById('btn-create').onclick = async () => {
            const n = document.getElementById('room-name').value;
            let w = Math.min(2000, parseInt(document.getElementById('room-w').value)||400), h = Math.min(2000, parseInt(document.getElementById('room-h').value)||600);
            const jp = document.getElementById('room-join-pass').value, dp = document.getElementById('room-del-pass').value;
            if(!n || !dp) return;
            const d = await addDoc(collection(db,"rooms"), {name:n, w, h, joinPass:jp, delPass:dp, host:myName, createdAt:Date.now()});
            window.joinRoom(d.id, n, w, h, myName);
        };
        const canvas = document.getElementById('canvas'), ctx = canvas.getContext('2d'), wrap = document.getElementById('canvas-wrap'), cont = document.getElementById('canvas-container');
        let drawing = false, lx, ly;
        window.joinRoom = (id, name, w, h, host) => {
            activeRoomId = id; canvas.width = w; canvas.height = h;
            document.getElementById('lobby-page').classList.add('hidden'); document.getElementById('game-page').classList.remove('hidden');
            document.getElementById('room-label').innerText = name;
            posX = (wrap.clientWidth-w)/2; posY = (wrap.clientHeight-h)/2; updateTransform();
            onValue(dbQuery(ref(rtdb, `rooms/${id}/emojis`), limitToLast(1)), snap => {
                const val = snap.val(); if(val) { const item = Object.values(val)[0]; if(Date.now()-item.t < 2000) showEmoji(item.e, item.u); }
            });
            onValue(ref(rtdb, `draws/${id}`), snap => {
                ctx.clearRect(0,0,w,h);
                const data = snap.val()||{}, all = [], layers = [];
                Object.keys(data).forEach(lid => { layers.push(lid); Object.keys(data[lid]).forEach(k => { if(data[lid][k].x1!==undefined) all.push({...data[lid][k], lid}); }); });
                all.sort((a,b)=>a.lid.localeCompare(b.lid,undefined,{numeric:true})).forEach(d => {
                    ctx.beginPath(); ctx.strokeStyle=d.c; ctx.lineWidth=d.s; ctx.lineCap="round"; ctx.moveTo(d.x1,d.y1); ctx.lineTo(d.x2,d.y2); ctx.stroke();
                });
                roomLayers = layers.sort((a,b)=>a.localeCompare(b,undefined,{numeric:true})); if(roomLayers.length===0) roomLayers=["1"];
                renderLayerUI();
            });
        };
        function showEmoji(e, u) {
            const div = document.createElement('div'); div.className = 'reaction-bubble';
            div.innerHTML = `${e}<div class="reaction-user">${u}</div>`;
            document.getElementById('reaction-container').appendChild(div);
            setTimeout(()=>div.remove(), 2500);
        }
        document.getElementById('btn-open-emoji').onclick = (e) => { e.stopPropagation(); document.getElementById('stamp-menu').classList.toggle('hidden'); };
        document.querySelectorAll('.emoji-opt').forEach(el => {
            el.onclick = () => { push(ref(rtdb, `rooms/${activeRoomId}/emojis`), { e: el.innerText, u: myName, t: Date.now() }); };
        });
        function updateTransform() { cont.style.transform = `translate(${posX}px, ${posY}px) scale(${scale})`; document.getElementById('zoom-label').innerText = scale.toFixed(1) + "x"; }
        const getPos = (e) => {
            const t = e.touches ? e.touches[0] : e, r = wrap.getBoundingClientRect();
            return [(t.clientX-r.left-posX)/scale, (t.clientY-r.top-posY)/scale];
        };
        wrap.addEventListener('mousedown', e => { drawing=true; [lx,ly]=getPos(e); });
        window.addEventListener('mousemove', e => {
            if(!drawing) return; const [x,y]=getPos(e);
            push(ref(rtdb, `draws/${activeRoomId}/${activeLayer}`), {x1:lx,y1:ly,x2:x,y2:y, c:mode==='eraser'?'#ffffff':document.getElementById('color-picker').value, s:document.getElementById('size-range').value});
            [lx,ly]=[x,y];
        });
        window.addEventListener('mouseup', ()=>drawing=false);
        let startDist = 0;
        wrap.addEventListener('touchstart', e => {
            if(e.touches.length===2) { drawing=false; startDist = Math.hypot(e.touches[0].pageX-e.touches[1].pageX, e.touches[0].pageY-e.touches[1].pageY); }
            else { drawing=true; [lx,ly]=getPos(e); }
        }, {passive:false});
        wrap.addEventListener('touchmove', e => {
            if(e.touches.length===2) {
                e.preventDefault(); const d = Math.hypot(e.touches[0].pageX-e.touches[1].pageX, e.touches[0].pageY-e.touches[1].pageY);
                const s = Math.max(0.1, Math.min(5, scale * (d/startDist)));
                const cX = (e.touches[0].clientX+e.touches[1].clientX)/2, cY = (e.touches[0].clientY+e.touches[1].clientY)/2, r = wrap.getBoundingClientRect();
                posX -= (cX-r.left-posX)*(s/scale-1); posY -= (cY-r.top-posY)*(s/scale-1);
                scale=s; startDist=d; updateTransform();
            } else if(drawing) { e.preventDefault(); const [x,y]=getPos(e); push(ref(rtdb,`draws/${activeRoomId}/${activeLayer}`),{x1:lx,y1:ly,x2:x,y2:y,c:mode==='eraser'?'#ffffff':document.getElementById('color-picker').value,s:document.getElementById('size-range').value}); [lx,ly]=[x,y]; }
        }, {passive:false});
        wrap.addEventListener('touchend', ()=>drawing=false);
        function renderLayerUI() {
            const l = document.getElementById('layer-list'); l.innerHTML = "";
            roomLayers.forEach(id => { if(id==="_init") return;
                const d = document.createElement('div'); d.className = `layer-item ${activeLayer===id?'active':''}`;
                d.innerHTML = `<button class="layer-btn">L${id}</button><button class="layer-del">×</button>`;
                d.querySelector('.layer-btn').onclick = () => { activeLayer=id; renderLayerUI(); };
                d.querySelector('.layer-del').onclick = () => { if(roomLayers.length>1) remove(ref(rtdb,`draws/${activeRoomId}/${id}`)); };
                l.appendChild(d);
            });
        }
        document.getElementById('btn-add-layer').onclick = () => {
            const next = Math.max(...roomLayers.filter(x=>x!=="_init").map(Number))+1;
            set(ref(rtdb,`draws/${activeRoomId}/${next}/_init`), true); activeLayer=String(next);
        };
        document.getElementById('btn-pen').onclick = () => { mode='pen'; updateBtn('btn-pen'); };
        document.getElementById('btn-eraser').onclick = () => { mode='eraser'; updateBtn('btn-eraser'); };
        function updateBtn(id) { document.querySelectorAll('.tool-btn').forEach(b=>b.classList.remove('active')); document.getElementById(id).classList.add('active'); }
        document.getElementById('btn-leave').onclick = () => location.reload();
        window.onclick = () => document.getElementById('stamp-menu').classList.add('hidden');
    </script>
</body>
