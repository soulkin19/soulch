<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>e ちゃんねる - 掲示板一覧</title>
    <style>
        :root {
            --e-green: #5cb85c;
            --e-green-hover: #4cae4c;
            --e-purple: #4b0082;
            --bg-gray: #f4f7f4;
            --card-shadow: 0 2px 10px rgba(0,0,0,0.08);
        }
        body {
            font-family: "Helvetica Neue", Arial, "Hiragino Kaku Gothic ProN", "Hiragino Sans", sans-serif;
            background-color: var(--bg-gray);
            margin: 0;
            color: #333;
            line-height: 1.6;
        }

        /* ヘッダー */
        header {
            background: white;
            padding: 40px 20px;
            text-align: center;
            border-bottom: 1px solid #ddd;
        }
        .logo-text {
            font-size: 3rem;
            color: var(--e-green);
            font-weight: bold;
            margin: 0;
            letter-spacing: 3px;
        }
        .sub-title {
            font-size: 1.5rem;
            color: var(--e-green);
            margin: 10px 0;
        }
        .tv-icon-box {
            width: 90px;
            height: 90px;
            background-color: #e8f4f8;
            margin: 15px auto;
            display: flex;
            align-items: center;
            justify-content: center;
            border-radius: 12px;
            font-size: 50px;
            border: 1px solid #cde;
        }

        .container {
            max-width: 1000px;
            margin: 0 auto;
            padding: 30px 20px;
        }

        /* 掲示板作成ボタン */
        .btn-create {
            background-color: var(--e-green);
            color: white;
            border: none;
            padding: 12px 28px;
            border-radius: 6px;
            font-size: 1.1rem;
            font-weight: bold;
            cursor: pointer;
            margin-bottom: 30px;
            transition: 0.2s;
        }
        .btn-create:hover { background-color: var(--e-green-hover); }

        /* スレッドカード */
        .board-grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
            gap: 20px;
        }
        .board-card {
            background: white;
            border-radius: 12px;
            padding: 25px;
            box-shadow: var(--card-shadow);
            border: 1px solid #eee;
            cursor: pointer;
            transition: 0.3s;
        }
        .board-card:hover { transform: translateY(-5px); border-color: var(--e-green); }
        .board-title {
            font-size: 1.25rem;
            color: var(--e-purple);
            font-weight: bold;
            margin-bottom: 12px;
            text-decoration: underline;
            display: block;
        }
        .post-count { color: #888; font-size: 0.95rem; }

        /* 掲示板内部表示 */
        #bbs-view { display: none; background: white; border-radius: 15px; padding: 30px; box-shadow: var(--card-shadow); }
        .back-link { color: var(--e-green); cursor: pointer; font-weight: bold; margin-bottom: 25px; display: inline-block; }
        
        .message-item {
            padding: 20px;
            border-bottom: 1px solid #f0f0f0;
            position: relative;
        }
        .message-header {
            font-size: 0.9rem;
            color: #666;
            margin-bottom: 8px;
            display: flex;
            justify-content: space-between;
        }
        .user-name { color: #2e7d32; font-weight: bold; }
        .message-body { font-size: 1.05rem; white-space: pre-wrap; color: #444; }
        
        .admin-delete {
            color: #ff4444;
            cursor: pointer;
            font-size: 0.8rem;
            background: #fff0f0;
            border: 1px solid #ffcccc;
            padding: 2px 8px;
            border-radius: 4px;
        }

        /* フォーム改善 */
        .form-area {
            margin-top: 40px;
            padding: 30px;
            background: #fdfdfd;
            border: 2px solid var(--e-green);
            border-radius: 12px;
        }
        .form-label { font-weight: bold; margin-bottom: 5px; display: block; color: #555; }
        .form-area input, .form-area textarea {
            width: 100%;
            padding: 12px;
            margin-bottom: 15px;
            border: 1px solid #ccc;
            border-radius: 8px;
            box-sizing: border-box;
            font-size: 1rem;
        }
        .btn-post {
            background-color: var(--e-green);
            color: white;
            border: none;
            padding: 15px;
            width: 100%;
            border-radius: 8px;
            font-size: 1.2rem;
            font-weight: bold;
            cursor: pointer;
        }
    </style>
</head>
<body>

<header>
    <h1 class="logo-text">e ちゃんねる</h1>
    <p class="sub-title">掲示板一覧</p>
    <div class="tv-icon-box">📺</div>
</header>

<div class="container" id="top-view">
    <button class="btn-create" onclick="createNewBoard()">掲示板を作成する</button>
    <div class="board-grid" id="board-list"></div>
</div>

<div class="container" id="bbs-view">
    <span class="back-link" onclick="showTop()">← 一覧に戻る</span>
    <h2 id="current-board-title" style="color: var(--e-purple); margin-top: 0;"></h2>
    <div id="message-container"></div>

    <div class="form-area">
        <label class="form-label">名前</label>
        <input type="text" id="username" value="名無しさん">
        <label class="form-label">メッセージ</label>
        <textarea id="content" placeholder="ここに内容を書いてください" rows="5"></textarea>
        <button class="btn-post" id="send-btn">書き込む</button>
    </div>
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
    const ADMIN_PASS_HASH = "YWRtaW4xMjM="; // admin123
    let currentBoardKey = '';

    // --- 掲示板作成 ---
    window.createNewBoard = () => {
        const title = prompt("新しい掲示板の名前は？");
        if (title) push(ref(db, 'boards'), { title: title });
    };

    // --- 一覧読み込み ---
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

    // --- 掲示板を開く ---
    window.openBoard = (key, title) => {
        currentBoardKey = key;
        document.getElementById('top-view').style.display = 'none';
        document.getElementById('bbs-view').style.display = 'block';
        document.getElementById('current-board-title').innerText = title;
        const container = document.getElementById('message-container');
        container.innerHTML = '';

        onChildAdded(ref(db, `messages/${key}`), (snapshot) => {
            const msg = snapshot.val();
            const msgKey = snapshot.key;
            const div = document.createElement('div');
            div.className = 'message-item';
            div.id = `msg-${msgKey}`;
            div.innerHTML = `
                <div class="message-header">
                    <span>投稿者: <span class="user-name">${msg.username}</span></span>
                    <span class="admin-delete" onclick="event.stopPropagation(); adminDelete('${msgKey}')">削除</span>
                </div>
                <div class="message-body">${msg.text}</div>
            `;
            container.appendChild(div);
        });
    };

    // --- 削除機能 ---
    window.adminDelete = (msgKey) => {
        const pass = prompt("管理者パスワードを入力してください");
        if (pass && btoa(pass) === ADMIN_PASS_HASH) {
            remove(ref(db, `messages/${currentBoardKey}/${msgKey}`))
                .then(() => {
                    document.getElementById(`msg-${msgKey}`).remove();
                    alert("削除しました");
                });
        } else {
            alert("パスワードが違います");
        }
    };

    window.showTop = () => {
        document.getElementById('top-view').style.display = 'block';
        document.getElementById('bbs-view').style.display = 'none';
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
