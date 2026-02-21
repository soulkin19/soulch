<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>e ちゃんねる</title>
    <style>
        :root { --e-green: #5cb85c; --e-purple: #4b0082; --bg-light-green: #f4f9f4; }
        html, body { margin: 0; padding: 0; background-color: var(--bg-light-green); min-height: 100vh; font-family: sans-serif; }
        header { background-color: white; padding: 30px 20px; text-align: center; border-bottom: 1px solid #ddd; }
        .logo-text { font-size: 40px; color: var(--e-green); font-weight: bold; margin: 0; }
        .sub-title { font-size: 20px; color: var(--e-green); margin: 10px 0; font-weight: bold; }
        .container { max-width: 1000px; margin: 0 auto; padding: 20px; }
        .btn-create { background-color: var(--e-green); color: white; border: none; padding: 10px 15px; border-radius: 8px; font-size: 16px; cursor: pointer; margin-bottom: 20px; }
        .board-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(300px, 1fr)); gap: 15px; }
        .board-card { background: white; border: 1px solid #ddd; border-radius: 8px; padding: 15px; cursor: pointer; }
        .board-title { font-size: 18px; color: var(--e-purple); font-weight: bold; text-decoration: underline; margin-bottom: 5px; }
        .post-info { font-size: 13px; color: #666; text-decoration: underline; margin-top: 2px; }
        #bbs-view { display: none; background: white; padding: 20px; border-radius: 4px; border: 1px solid #ddd; }
        .form-label { font-size: 16px; margin-bottom: 5px; display: block; }
        .form-input { width: 100%; border: 1px solid #ccc; padding: 8px; margin-bottom: 15px; box-sizing: border-box; }
        .form-textarea { width: 100%; border: 1px solid #ccc; padding: 8px; margin-bottom: 15px; box-sizing: border-box; height: 100px; }
        .btn-group { display: flex; gap: 10px; margin-bottom: 25px; }
        .btn-green { background-color: var(--e-green); color: white; border: none; padding: 8px 20px; border-radius: 4px; font-size: 16px; cursor: pointer; }
        .post-item { margin-bottom: 20px; font-size: 16px; }
        .post-user { font-weight: bold; }
        .post-time { font-size: 12px; color: #888; margin-top: 2px; }
        .admin-del { font-size: 11px; color: #ccc; cursor: pointer; margin-left: 10px; }
    </style>
</head>
<body>
<header id="header-part">
    <h1 class="logo-text">e ちゃんねる</h1>
    <div class="sub-title">掲示板一覧</div>
    <div style="background:#e8f4f8; width:80px; height:80px; margin:10px auto; border-radius:5px; display:flex; align-items:center; justify-content:center; font-size:40px;">📺</div>
</header>
<div class="container" id="top-view">
    <button class="btn-create" onclick="createNewBoard()">掲示板を作成する</button>
    <div class="board-grid" id="board-list"></div>
</div>
<div class="container" id="bbs-view">
    <label class="form-label">名前:</label>
    <input type="text" id="username" class="form-input" value="名無しさん">
    <label class="form-label">内容:</label>
    <textarea id="content" class="form-textarea"></textarea>
    <div class="btn-group">
        <button class="btn-green" id="send-btn">投稿</button>
        <button class="btn-green" onclick="location.reload()">戻る</button>
    </div>
    <div id="message-container"></div>
</div>
<script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
    import { getDatabase, ref, push, onChildAdded, onValue, remove, update, get } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-database.js";
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
    const db = getDatabase(app);
    const ADMIN_PASS = "ZGVsdGE0Mzc="; 
    const getFp = () => {
        let id = localStorage.getItem('fp');
        if (!id) { id = 'fp_' + Math.random().toString(36).substr(2, 9); localStorage.setItem('fp', id); }
        return id;
    };
    const myId = getFp();
    let currentKey = '';
    onValue(ref(db, 'boards'), (snap) => {
        const list = document.getElementById('board-list');
        list.innerHTML = '';
        const boards = [];
        snap.forEach(c => { boards.push({ key: c.key, ...c.val() }); });
        boards.sort((a, b) => (b.lastUpdated || 0) - (a.lastUpdated || 0));
        boards.forEach(b => {
            const card = document.createElement('div');
            card.className = 'board-card';
            card.onclick = () => openBoard(b.key, b.title);
            onValue(ref(db, `messages/${b.key}`), (mSnap) => {
                const count = mSnap.exists() ? Object.keys(mSnap.val()).length : 0;
                const lastTime = b.lastUpdated ? new Date(b.lastUpdated).toLocaleString() : 'なし';
                card.innerHTML = `<div class="board-title">${b.title}</div>
                                  <div class="post-info">投稿数: ${count}</div>
                                  <div class="post-info">最終投稿: ${lastTime}</div>`;
            });
            list.appendChild(card);
        });
    });
    window.createNewBoard = () => {
        const t = prompt("掲示板名:");
        if (t) push(ref(db, 'boards'), { title: t, lastUpdated: Date.now() });
    };
    window.openBoard = (key, title) => {
        currentKey = key;
        document.getElementById('top-view').style.display = 'none';
        document.getElementById('header-part').style.display = 'none';
        document.getElementById('bbs-view').style.display = 'block';
        const container = document.getElementById('message-container');
        container.innerHTML = '';
        onChildAdded(ref(db, `messages/${key}`), (s) => {
            const m = s.val();
            const div = document.createElement('div');
            div.className = 'post-item';
            div.id = `m-${s.key}`;
            div.innerHTML = `<div><span class="post-user">${m.username}</span>: ${m.text}</div>
                <div class="post-time">${new Date(m.timestamp).toLocaleString()} <span class="admin-del" onclick="admin('${s.key}', '${m.uid}')">[管理]</span></div>`;
            container.prepend(div);
        });
    };
    document.getElementById('send-btn').onclick = async () => {
        const txt = document.getElementById('content').value;
        if (!txt) return;
        const b = await get(ref(db, `blacklist/${myId}`));
        if (b.exists()) { alert("投稿禁止されています"); return; }
        const now = Date.now();
        push(ref(db, `messages/${currentKey}`), { username: document.getElementById('username').value, text: txt, timestamp: now, uid: myId });
        update(ref(db, `boards/${currentKey}`), { lastUpdated: now });
        document.getElementById('content').value = '';
    };
    window.admin = (mKey, uId) => {
        const p = prompt("パスワード:");
        if (p && btoa(p) === ADMIN_PASS) {
            const a = prompt("1:削除 2:アク禁");
            if (a === "1") { remove(ref(db, `messages/${currentKey}/${mKey}`)); document.getElementById(`m-${mKey}`).remove(); }
            else if (a === "2") { update(ref(db, `blacklist/${uId}`), { banned: true }); alert("完了"); }
        }
    };
</script>
</body>
</html>
