<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>e ちゃんねる</title>
    <style>
        :root { --e-green: #5cb85c; --e-purple: #4b0082; --bg-light-green: #f4f9f4; }
        html { height: 100%; }
        body { margin: 0; padding: 0; background-color: var(--bg-light-green); min-height: 100%; font-family: sans-serif; display: flex; flex-direction: column; }
        header { background-color: white; padding: 30px 20px; text-align: center; border-bottom: 1px solid #ddd; flex-shrink: 0; }
        .logo-text { font-size: 40px; color: var(--e-green); font-weight: bold; margin: 0; }
        .container { max-width: 1000px; margin: 0 auto; padding: 20px; width: 100%; box-sizing: border-box; flex: 1; }
        .btn-create { background-color: var(--e-green); color: white; border: none; padding: 10px 18px; border-radius: 4px; cursor: pointer; margin-bottom: 25px; font-weight: bold; }
        #board-list { display: grid; grid-template-columns: repeat(auto-fill, minmax(300px, 1fr)); gap: 15px; }
        .board-card { background: white; border: 1px solid #ddd; border-radius: 8px; padding: 18px; cursor: pointer; display: flex; flex-direction: column; }
        .board-title { font-size: 18px; color: var(--e-purple); font-weight: bold; text-decoration: underline; margin-bottom: 5px; }
        .board-info { font-size: 13px; color: #666; text-decoration: underline; margin-top: 2px; }
        #bbs-view { display: none; background: white; padding: 25px; border-radius: 4px; border: 1px solid #ddd; }
        .form-input, .form-textarea { width: 100%; border: 1px solid #ccc; padding: 10px; margin-bottom: 15px; box-sizing: border-box; border-radius: 3px; }
        .btn-group { display: flex; gap: 10px; margin-bottom: 30px; }
        .btn-green { background-color: var(--e-green); color: white; border: none; padding: 10px 25px; border-radius: 4px; cursor: pointer; font-weight: bold; }
        .post-item { margin-bottom: 25px; padding-bottom: 15px; border-bottom: 1px solid #eee; position: relative; }
        .post-user { font-weight: bold; }
        .post-time { font-size: 12px; color: #888; margin-top: 5px; }
        .admin-menu { font-size: 11px; color: #ccc; cursor: pointer; margin-left: 10px; }
    </style>
</head>
<body>

<header id="header-part">
    <h1 class="logo-text">e ちゃんねる</h1>
    <div style="color:var(--e-green); font-weight:bold;">掲示板一覧</div>
</header>

<div class="container" id="top-view">
    <button class="btn-create" onclick="createNewBoard()">掲示板を作成する</button>
    <div id="board-list"></div>
</div>

<div class="container" id="bbs-view">
    <input type="text" id="username" class="form-input" value="名無しさん">
    <textarea id="content" class="form-textarea" placeholder="内容を入力"></textarea>
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

    const getUserFingerprint = () => {
        let id = localStorage.getItem('user_fingerprint');
        if (!id) {
            id = 'fp_' + Math.random().toString(36).substr(2, 9);
            localStorage.setItem('user_fingerprint', id);
        }
        return id;
    };
    const myId = getUserFingerprint();

    let currentBoardKey = '';

    onValue(ref(db, 'boards'), (snapshot) => {
        const listDiv = document.getElementById('board-list');
        listDiv.innerHTML = '';
        const boards = [];
        snapshot.forEach(child => { boards.push({ key: child.key, ...child.val() }); });
        boards.sort((a, b) => (b.lastUpdated || 0) - (a.lastUpdated || 0));
        boards.forEach(b => {
            const card = document.createElement('div');
            card.className = 'board-card';
            card.onclick = () => openBoard(b.key, b.title);
            
            onValue(ref(db, `messages/${b.key}`), (mSnap) => {
                const count = mSnap.exists() ? Object.keys(mSnap.val()).length : 0;
                card.innerHTML = `
                    <div class="board-title">${b.title}</div>
                    <div class="board-info">投稿数: ${count}</div>
                    <div class="board-info">更新: ${b.lastUpdated ? new Date(b.lastUpdated).toLocaleString() : 'なし'}</div>
                `;
            });
            listDiv.appendChild(card);
        });
    });

    window.createNewBoard = () => {
        const t = prompt("掲示板名:");
        if (t) push(ref(db, 'boards'), { title: t, lastUpdated: Date.now() });
    };

    window.openBoard = (key, title) => {
        currentBoardKey = key;
        document.getElementById('top-view').style.display = 'none';
        document.getElementById('header-part').style.display = 'none';
        document.getElementById('bbs-view').style.display = 'block';
        const container = document.getElementById('message-container');
        container.innerHTML = '';

        onChildAdded(ref(db, `messages/${key}`), (snap) => {
            const m = snap.val();
            const div = document.createElement('div');
            div.className = 'post-item';
            div.id = `msg-${snap.key}`;
            div.innerHTML = `
                <div><span class="post-user">${m.username}</span>: ${m.text}</div>
                <div class="post-time">${new Date(m.timestamp).toLocaleString()} 
                <span class="admin-menu" onclick="adminAction('${snap.key}', '${m.uid}')">[管理]</span></div>
            `;
            container.prepend(div);
        });
    };

    document.getElementById('send-btn').onclick = async () => {
        const txt = document.getElementById('content').value;
        if (!txt) return;

        const bannedSnap = await get(ref(db, `blacklist/${myId}`));
        if (bannedSnap.exists()) {
            alert("投稿できません。");
            return;
        }

        const now = Date.now();
        push(ref(db, `messages/${currentBoardKey}`), {
            username: document.getElementById('username').value || "名無しさん",
            text: txt,
            timestamp: now,
            uid: myId
        });
        update(ref(db, `boards/${currentBoardKey}`), { lastUpdated: now });
        document.getElementById('content').value = '';
    };

    window.adminAction = (msgKey, targetUid) => {
        const p = prompt("管理者パスワード:");
        if (p && btoa(p) === ADMIN_PASS) {
            const action = prompt("操作: 1(削除), 2(アク禁)");
            if (action === "1") {
                remove(ref(db, `messages/${currentBoardKey}/${msgKey}`));
                document.getElementById(`msg-${msgKey}`).remove();
            } else if (action === "2") {
                remove(ref(db, `messages/${currentBoardKey}/${msgKey}`));
                update(ref(db, `blacklist/${targetUid}`), { banned: true, date: Date.now() });
                document.getElementById(`msg-${msgKey}`).remove();
                alert("登録完了");
            }
        }
    };
</script>
</body>
</html>
