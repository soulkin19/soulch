<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>e ちゃんねる</title>
    <style>
        :root {
            --e-green: #5cb85c;
            --e-purple: #4b0082;
            --bg-light-green: #f4f9f4; /* 薄緑の背景 */
        }
        html, body {
            height: 100%;
            margin: 0;
            background-color: var(--bg-light-green); /* 背景を下に伸ばす */
        }
        body {
            font-family: sans-serif;
            color: #333;
        }

        /* ヘッダー */
        header {
            text-align: center;
            padding: 40px 20px;
            background-color: white;
        }
        .logo-text { font-size: 42px; color: var(--e-green); font-weight: bold; margin: 0; }
        .sub-title { font-size: 20px; color: var(--e-green); margin: 10px 0; font-weight: bold; }
        .tv-icon { width: 80px; height: 80px; margin: 10px auto; background-color: #d9edf7; display: flex; align-items: center; justify-content: center; border-radius: 5px; font-size: 40px; }

        .container {
            max-width: 900px;
            margin: 0 auto;
            padding: 20px;
            min-height: 100%;
        }

        /* スレッド一覧 */
        .btn-create { background-color: var(--e-green); color: white; border: none; padding: 10px 20px; border-radius: 5px; font-size: 16px; cursor: pointer; margin-bottom: 20px; }
        .board-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 15px; }
        .board-card { background: white; border: 1px solid #ddd; border-radius: 8px; padding: 15px; cursor: pointer; }
        .board-title { font-size: 18px; color: var(--e-purple); font-weight: bold; text-decoration: underline; margin-bottom: 5px; display: block; }
        .post-count { font-size: 13px; color: #666; text-decoration: underline; }

        /* 投稿画面（画像IMG_0084を再現） */
        #bbs-view { display: none; background: white; padding: 20px; border-radius: 8px; }
        .form-row { margin-bottom: 15px; }
        .form-row label { display: block; font-size: 14px; margin-bottom: 5px; }
        .form-row input, .form-row textarea { width: 100%; border: 1px solid #ccc; padding: 8px; border-radius: 3px; box-sizing: border-box; }
        .button-group { display: flex; gap: 10px; margin-bottom: 30px; }
        .btn-submit { background-color: var(--e-green); color: white; border: none; padding: 8px 20px; border-radius: 4px; cursor: pointer; font-size: 16px; }
        .btn-back { background-color: var(--e-green); color: white; border: none; padding: 8px 20px; border-radius: 4px; cursor: pointer; font-size: 16px; }

        /* 投稿リスト */
        .post-item { margin-bottom: 20px; font-size: 15px; border-bottom: 1px solid #eee; padding-bottom: 10px; }
        .post-header { font-weight: bold; margin-bottom: 3px; }
        .post-header span { font-weight: normal; }
        .post-time { font-size: 12px; color: #666; margin-bottom: 5px; }
        .admin-del { font-size: 11px; color: #ccc; cursor: pointer; margin-left: 10px; }
    </style>
</head>
<body>

<header id="header-area">
    <h1 class="logo-text">e ちゃんねる</h1>
    <p class="sub-title">掲示板一覧</p>
    <div class="tv-icon">📺</div>
</header>

<div class="container" id="top-view">
    <button class="btn-create" onclick="createNewBoard()">掲示板を作成する</button>
    <div class="board-grid" id="board-list"></div>
</div>

<div class="container" id="bbs-view">
    <div class="form-row">
        <label>名前:</label>
        <input type="text" id="username" value="名無しさん">
    </div>
    <div class="form-row">
        <label>内容:</label>
        <textarea id="content" rows="4"></textarea>
    </div>
    <div class="button-group">
        <button class="btn-submit" id="send-btn">投稿</button>
        <button class="btn-back" onclick="showTop()">戻る</button>
    </div>

    <div id="message-container"></div>
</div>

<script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
    import { getDatabase, ref, push, onChildAdded, onValue, remove, serverTimestamp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-database.js";

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
    const ADMIN_PASS_HASH = "YWRtaW4xMjM="; 
    let currentBoardKey = '';

    // 時刻フォーマット関数
    const formatDate = (timestamp) => {
        const d = new Date(timestamp);
        return `${d.getFullYear()}-${(d.getMonth()+1).toString().padStart(2,'0')}-${d.getDate().toString().padStart(2,'0')} ${d.getHours().toString().padStart(2,'0')}:${d.getMinutes().toString().padStart(2,'0')}:${d.getSeconds().toString().padStart(2,'0')}`;
    };

    window.createNewBoard = () => {
        const title = prompt("掲示板の名前を入力:");
        if (title) push(ref(db, 'boards'), { title: title });
    };

    onChildAdded(ref(db, 'boards'), (snapshot) => {
        const board = snapshot.val();
        const key = snapshot.key;
        const card = document.createElement('div');
        card.className = 'board-card';
        card.onclick = () => openBoard(key, board.title);
        
        onValue(ref(db, `messages/${key}`), (msgSnap) => {
            const count = msgSnap.exists() ? Object.keys(msgSnap.val()).length : 0;
            card.innerHTML = `<span class="board-title">${board.title}</span><div class="post-count">投稿数: ${count}</div>`;
        });
        document.getElementById('board-list').appendChild(card);
    });

    window.openBoard = (key, title) => {
        currentBoardKey = key;
        document.getElementById('top-view').style.display = 'none';
        document.getElementById('header-area').style.display = 'none';
        document.getElementById('bbs-view').style.display = 'block';
        const container = document.getElementById('message-container');
        container.innerHTML = '';

        onChildAdded(ref(db, `messages/${key}`), (snapshot) => {
            const msg = snapshot.val();
            const msgKey = snapshot.key;
            const timeStr = msg.timestamp ? formatDate(msg.timestamp) : "時刻不明";
            const div = document.createElement('div');
            div.className = 'post-item';
            div.id = `msg-${msgKey}`;
            div.innerHTML = `
                <div class="post-header">${msg.username}: <span>${msg.text}</span></div>
                <div class="post-time">${timeStr} <span class="admin-del" onclick="adminDelete('${msgKey}')">[削除]</span></div>
            `;
            container.prepend(div); // 新しい投稿を上に
        });
    };

    window.adminDelete = (msgKey) => {
        const pass = prompt("管理者パスワード:");
        if (pass && btoa(pass) === ADMIN_PASS_HASH) {
            remove(ref(db, `messages/${currentBoardKey}/${msgKey}`)).then(() => {
                document.getElementById(`msg-${msgKey}`).remove();
            });
        }
    };

    window.showTop = () => {
        location.reload(); 
    };

    document.getElementById('send-btn').addEventListener('click', () => {
        const text = document.getElementById('content').value;
        if (!text) return;
        push(ref(db, `messages/${currentBoardKey}`), {
            username: document.getElementById('username').value || "名無しさん",
            text: text,
            timestamp: Date.now()
        });
        document.getElementById('content').value = '';
    });
</script>
</body>
</html>
