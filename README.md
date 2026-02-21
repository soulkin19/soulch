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
            --bg-light: #f4f9f4;
        }
        body {
            font-family: "Helvetica Neue", Arial, "Hiragino Kaku Gothic ProN", "Hiragino Sans", Meiryo, sans-serif;
            background-color: var(--bg-light);
            margin: 0;
            padding: 0;
            color: #333;
        }

        /* ヘッダーエリア */
        header {
            text-align: center;
            padding: 40px 20px 20px;
            background-color: white;
        }
        .logo-text {
            font-size: 48px;
            color: var(--e-green);
            font-weight: bold;
            margin: 0;
            letter-spacing: 2px;
        }
        .sub-title {
            font-size: 24px;
            color: var(--e-green);
            margin: 10px 0;
            font-weight: bold;
        }
        .tv-icon {
            width: 80px;
            height: 80px;
            margin: 10px auto;
            background-color: #d9edf7;
            display: flex;
            align-items: center;
            justify-content: center;
            border-radius: 5px;
            font-size: 40px;
        }

        /* コンテンツエリア */
        .container {
            max-width: 900px;
            margin: 0 auto;
            padding: 20px;
        }

        .btn-create {
            background-color: var(--e-green);
            color: white;
            border: none;
            padding: 12px 24px;
            border-radius: 5px;
            font-size: 18px;
            font-weight: bold;
            cursor: pointer;
            margin-bottom: 30px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }

        /* カードレイアウト */
        .board-grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
            gap: 15px;
        }
        .board-card {
            background: white;
            border: 1px solid #ddd;
            border-radius: 10px;
            padding: 20px;
            text-decoration: none;
            color: inherit;
            box-shadow: 0 2px 8px rgba(0,0,0,0.05);
            transition: transform 0.2s;
        }
        .board-card:hover {
            transform: translateY(-3px);
            border-color: var(--e-green);
        }
        .board-title {
            font-size: 18px;
            color: var(--e-purple);
            font-weight: bold;
            margin-bottom: 10px;
            display: block;
            text-decoration: underline;
        }
        .post-count {
            font-size: 14px;
            color: #777;
            border-bottom: 1px solid #eee;
            padding-bottom: 5px;
        }

        /* 掲示板表示時 */
        #bbs-view { display: none; background: white; border-radius: 10px; padding: 20px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
        .back-link { color: var(--e-green); cursor: pointer; margin-bottom: 20px; display: inline-block; }
        .message-item { padding: 15px; border-bottom: 1px solid #eee; }
        .message-header { font-size: 14px; color: #666; margin-bottom: 5px; }

        /* 書き込みフォーム */
        .form-area {
            margin-top: 30px;
            padding: 20px;
            background: #f9f9f9;
            border-top: 4px solid var(--e-green);
            border-radius: 0 0 10px 10px;
        }
        .form-area input, .form-area textarea {
            width: 100%;
            padding: 12px;
            margin-bottom: 10px;
            border: 1px solid #ccc;
            border-radius: 5px;
            box-sizing: border-box;
            font-size: 16px;
        }
        .btn-post {
            background-color: var(--e-green);
            color: white;
            border: none;
            padding: 15px;
            width: 100%;
            border-radius: 5px;
            font-size: 18px;
            font-weight: bold;
            cursor: pointer;
        }
    </style>
</head>
<body>

<header>
    <h1 class="logo-text">e ちゃんねる</h1>
    <p class="sub-title">掲示板一覧</p>
    <div class="tv-icon">📺</div> </header>

<div class="container" id="top-view">
    <button class="btn-create" onclick="createNewBoard()">掲示板を作成する</button>
    <div class="board-grid" id="board-list">
        </div>
</div>

<div class="container" id="bbs-view">
    <span class="back-link" onclick="showTop()">← 掲示板一覧に戻る</span>
    <h2 id="current-board-title" style="color: var(--e-purple);"></h2>
    <div id="message-container"></div>

    <div class="form-area">
        <input type="text" id="username" placeholder="名前" value="名無しさん">
        <textarea id="content" placeholder="内容を入力してください" rows="4"></textarea>
        <button class="btn-post" id="send-btn">書き込む</button>
    </div>
</div>

<script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
    import { getDatabase, ref, push, onChildAdded, onValue, serverTimestamp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-database.js";

    // あなたのFirebase設定
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

    // --- スレッド作成 ---
    window.createNewBoard = () => {
        const title = prompt("新しい掲示板の名前を入力してください");
        if (title) {
            push(ref(db, 'boards'), {
                title: title,
                createdAt: serverTimestamp()
            });
        }
    };

    // --- スレッド一覧読み込み ---
    onChildAdded(ref(db, 'boards'), (snapshot) => {
        const board = snapshot.val();
        const key = snapshot.key;

        const card = document.createElement('div');
        card.className = 'board-card';
        card.onclick = () => openBoard(key, board.title);

        // 投稿数を監視
        const msgRef = ref(db, `messages/${key}`);
        onValue(msgRef, (msgSnapshot) => {
            const count = msgSnapshot.exists() ? Object.keys(msgSnapshot.val()).length : 0;
            card.innerHTML = `
                <span class="board-title">${board.title}</span>
                <div class="post-count">投稿数: ${count}</div>
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
        document.getElementById('message-container').innerHTML = '';

        const msgRef = ref(db, `messages/${key}`);
        onChildAdded(msgRef, (snapshot) => {
            const msg = snapshot.val();
            const div = document.createElement('div');
            div.className = 'message-item';
            div.innerHTML = `
                <div class="message-header">投稿者: ${msg.username}</div>
                <div class="message-body">${msg.text}</div>
            `;
            document.getElementById('message-container').appendChild(div);
            window.scrollTo(0, document.body.scrollHeight);
        });
    };

    window.showTop = () => {
        document.getElementById('top-view').style.display = 'block';
        document.getElementById('bbs-view').style.display = 'none';
        location.reload(); // 簡易的なリセット
    };

    // --- 書き込み ---
    document.getElementById('send-btn').addEventListener('click', () => {
        const text = document.getElementById('content').value;
        if (!text) return;
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
