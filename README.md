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
            --bg-light-green: #f4f9f4; /* あの画像通りの薄緑 */
        }
        
        /* 背景を画面下まで強制的に伸ばす設定 */
        html {
            height: 100%;
        }
        body {
            min-height: 100%;
            margin: 0;
            padding: 0;
            background-color: var(--bg-light-green); /* 全体の背景色 */
            font-family: sans-serif;
            color: #333;
            display: flex;
            flex-direction: column;
        }

        /* ヘッダー */
        header {
            text-align: center;
            padding: 40px 20px;
            background-color: white;
            border-bottom: 1px solid #e0e0e0;
        }
        .logo-text { font-size: 42px; color: var(--e-green); font-weight: bold; margin: 0; }
        .sub-title { font-size: 20px; color: var(--e-green); margin: 10px 0; font-weight: bold; }
        .tv-icon { width: 80px; height: 80px; margin: 10px auto; background-color: #d9edf7; display: flex; align-items: center; justify-content: center; border-radius: 5px; font-size: 40px; border: 1px solid #cde; }

        .container {
            max-width: 900px;
            margin: 0 auto;
            padding: 20px;
            flex: 1; /* コンテンツが少なくても背景を押し広げる */
            width: 100%;
            box-sizing: border-box;
        }

        /* スレッド一覧画面 */
        .btn-create { background-color: var(--e-green); color: white; border: none; padding: 12px 20px; border-radius: 5px; font-size: 16px; cursor: pointer; margin-bottom: 25px; font-weight: bold; }
        .board-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 15px; }
        .board-card { background: white; border: 1px solid #ddd; border-radius: 8px; padding: 20px; cursor: pointer; box-shadow: 0 2px 5px rgba(0,0,0,0.05); }
        .board-title { font-size: 18px; color: var(--e-purple); font-weight: bold; text-decoration: underline; margin-bottom: 8px; display: block; }
        .post-count { font-size: 13px; color: #666; text-decoration: underline; }

        /* 投稿表示画面（画像IMG_0084再現） */
        #bbs-view { display: none; background: white; padding: 25px; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        .form-row { margin-bottom: 15px; }
        .form-row label { display: block; font-size: 14px; margin-bottom: 5px; font-weight: bold; }
        .form-row input, .form-row textarea { width: 100%; border: 1px solid #ccc; padding: 10px; border-radius: 4px; box-sizing: border-box; font-size: 16px; }
        .button-group { display: flex; gap: 10px; margin-bottom: 30px; }
        .btn-submit, .btn-back { background-color: var(--e-green); color: white; border: none; padding: 10px 25px; border-radius: 4px; cursor: pointer; font-size: 16px; font-weight: bold; }
        .btn-back { background-color: #5cb85c; opacity: 0.9; }

        /* 投稿のリスト表示 */
        .post-item { margin-bottom: 25px; padding-bottom: 15px; border-bottom: 1px solid #f0f0f0; }
        .post-header { font-weight: bold; font-size: 16px; margin-bottom: 5px; }
        .post-content { font-weight: normal; margin-left: 5px; white-space: pre-wrap; }
        .post-footer { display: flex; align-items: center; gap: 10px; font-size: 13px; color: #666; margin-top: 5px; }
        .admin-del { color: #ffaaaa; cursor: pointer; text-decoration: underline; }
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
        <textarea id="content" rows="5"></textarea>
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

    // 時刻フォーマット
    const formatDate = (ts) => {
        const d = new Date(ts);
        return `${d.getFullYear()}-${(d.getMonth()+1)}-${d.getDate()} ${d.getHours()}:${d.getMinutes().toString().padStart(2,'0')}:${d.getSeconds().toString().padStart(2,'0')}`;
    };

    window.createNewBoard = () => {
        const title = prompt("掲示板の名前を入力してください:");
        if (title) push(ref(db, 'boards'), { title: title });
    };

    // 一覧の読み込み
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

    // 掲示板を開く
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
            const timeStr = msg.timestamp ? formatDate(msg.timestamp) : "";
            const div = document.createElement('div');
            div.className = 'post-item';
            div.id = `msg-${msgKey}`;
            div.innerHTML = `
                <div class="post-header">${msg.username}: <span class="post-content">${msg.text}</span></div>
                <div class="post-footer">
                    <span>${timeStr}</span>
                    <span class="admin-del" onclick="adminDelete('${msgKey}')">削除</span>
                </div>
            `;
            container.prepend(div); // 最新を上に
        });
    };

    window.adminDelete = (msgKey) => {
        const pass = prompt("管理者パスワード:");
        if (pass && btoa(pass) === ADMIN_PASS_HASH) {
            remove(ref(db, `messages/${currentBoardKey}/${msgKey}`));
            document.getElementById(`msg-${msgKey}`).remove();
        }
    };

    window.showTop = () => { location.reload(); };

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
