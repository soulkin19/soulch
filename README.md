<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>eちゃんねる風 掲示板</title>
    <style>
        :root { --main-green: #5cb85c; --text-purple: #4b0082; --bg-gray: #f9f9f9; }
        body { font-family: sans-serif; background-color: white; margin: 0; color: #333; }
        
        /* ヘッダー */
        header { padding: 20px; text-align: right; max-width: 1000px; margin: 0 auto; }
        .logo-area { display: flex; flex-direction: column; align-items: flex-end; }
        .logo-text { font-size: 2.5rem; color: var(--main-green); font-weight: bold; margin: 0; }
        .sub-title { font-size: 1.2rem; color: var(--main-green); margin-top: 5px; }

        /* コンテンツエリア */
        .container { max-width: 1000px; margin: 0 auto; padding: 20px; }
        
        /* 「掲示板を作成する」ボタン */
        .btn-create { background-color: var(--main-green); color: white; border: none; padding: 10px 20px; border-radius: 5px; font-size: 1rem; cursor: pointer; margin-bottom: 20px; display: inline-block; }

        /* スレッド一覧（カードレイアウト） */
        .board-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(300px, 1fr)); gap: 20px; }
        .board-card { border: 1px solid #ddd; border-radius: 8px; padding: 15px; background: white; box-shadow: 0 2px 4px rgba(0,0,0,0.05); cursor: pointer; transition: 0.2s; }
        .board-card:hover { transform: translateY(-3px); box-shadow: 0 4px 8px rgba(0,0,0,0.1); }
        .board-title { font-size: 1.2rem; color: var(--text-purple); font-weight: bold; margin-bottom: 10px; display: block; text-decoration: none; }
        .post-count { font-size: 0.9rem; color: #666; }

        /* 掲示板内部（非表示時は隠す） */
        #bbs-view { display: none; }
        .back-btn { margin-bottom: 20px; cursor: pointer; color: var(--main-green); }
        .message { border-bottom: 1px solid #eee; padding: 10px 0; }

        /* 入力フォーム */
        .form-area { margin-top: 30px; border-top: 2px solid var(--main-green); padding-top: 20px; }
        .form-area input, .form-area textarea { width: 100%; padding: 10px; margin-bottom: 10px; border: 1px solid #ccc; border-radius: 4px; box-sizing: border-box; }
        .form-area button { background-color: var(--main-green); color: white; border: none; padding: 10px 20px; border-radius: 4px; width: 100%; cursor: pointer; }
    </style>
</head>
<body>

<header>
    <div class="logo-area">
        <h1 class="logo-text">e ちゃんねる</h1>
        <div class="sub-title">掲示板一覧</div>
    </div>
</header>

<div class="container" id="top-view">
    <button class="btn-create" onclick="createNewBoard()">掲示板を作成する</button>
    <div class="board-grid" id="board-list">
        </div>
</div>

<div class="container" id="bbs-view">
    <div class="back-btn" onclick="showTop()">← 掲示板一覧に戻る</div>
    <h2 id="current-board-title" style="color: var(--text-purple);"></h2>
    <div id="messages"></div>
    
    <div class="form-area">
        <input type="text" id="username" placeholder="名前" value="名無しさん">
        <textarea id="content" placeholder="内容を入力してください" rows="4"></textarea>
        <button id="send">書き込む</button>
    </div>
</div>

<script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
    import { getDatabase, ref, push, onChildAdded, onValue, serverTimestamp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-database.js";

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
    let currentBoardKey = '';

    // --- 掲示板作成 ---
    window.createNewBoard = () => {
        const title = prompt("新しい掲示板の名前を入力してください");
        if(title) {
            push(ref(db, 'boards'), { title: title, postCount: 0 });
        }
    };

    // --- 一覧表示 ---
    onChildAdded(ref(db, 'boards'), (snapshot) => {
        const board = snapshot.val();
        const key = snapshot.key;
        
        const card = document.createElement('div');
        card.className = 'board-card';
        card.onclick = () => openBoard(key, board.title);
        
        // 投稿数をリアルタイム監視
        const countRef = ref(db, `messages/${key}`);
        onValue(countRef, (msgSnapshot) => {
            const count = msgSnapshot.exists() ? Object.keys(msgSnapshot.val()).length : 0;
            card.innerHTML = `
                <div class="board-title">${board.title}</div>
                <div class="post-count">投稿数：${count}</div>
            `;
        });
        
        document.getElementById('board-list').appendChild(card);
    });

    // --- 掲示板を開く ---
    window.openBoard = (key, title) => {
        currentBoardKey = key;
        document.getElementById('top-view').style.display = 'none';
        document.getElementById('bbs-view').style.display = 'block';
        document.getElementById('current-board-title').innerText = title;
        document.getElementById('messages').innerHTML = '';
        
        const msgRef = ref(db, `messages/${key}`);
        onChildAdded(msgRef, (snapshot) => {
            const msg = snapshot.val();
            const div = document.createElement('div');
            div.className = 'message';
            div.innerHTML = `<strong>${msg.username}</strong>: ${msg.text}`;
            document.getElementById('messages').appendChild(div);
        });
    };

    window.showTop = () => {
        document.getElementById('top-view').style.display = 'block';
        document.getElementById('bbs-view').style.display = 'none';
    };

    // --- 書き込み ---
    document.getElementById('send').addEventListener('click', () => {
        const text = document.getElementById('content').value;
        if(!text) return;
        push(ref(db, `messages/${currentBoardKey}`), {
            username: document.getElementById('username').value || "名無しさん",
            text: text,
            timestamp: serverTimestamp()
        });
        document.getElementById('content').value = '';
    });
</script>
</body>
</html>
