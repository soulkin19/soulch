<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>e ちゃんねる</title>
    <style>
        /* 画像に基づいたカラー設定 */
        :root {
            --e-green: #5cb85c;
            --e-purple: #4b0082;
            --bg-light-green: #f4f9f4; 
        }

        /* 画面全体を薄緑の背景に */
        html, body {
            margin: 0;
            padding: 0;
            background-color: var(--bg-light-green);
            min-height: 100vh;
            font-family: sans-serif;
        }

        /* ヘッダー部分は白背景 */
        header {
            background-color: white;
            padding: 30px 20px;
            text-align: center;
            border-bottom: 1px solid #ddd;
        }
        .logo-text { font-size: 40px; color: var(--e-green); font-weight: bold; margin: 0; }
        .sub-title { font-size: 20px; color: var(--e-green); margin: 10px 0; font-weight: bold; }
        .tv-icon-img { width: 80px; height: 80px; margin: 10px auto; display: block; }

        /* コンテンツ幅の制限 */
        .container {
            max-width: 1000px;
            margin: 0 auto;
            padding: 20px;
        }

        /* ボタンデザイン */
        .btn-create {
            background-color: var(--e-green);
            color: white;
            border: none;
            padding: 10px 15px;
            border-radius: 4px;
            font-size: 16px;
            cursor: pointer;
            margin-bottom: 20px;
        }

        /* スレッドカード（画像通りのスタイル） */
        .board-grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
            gap: 15px;
        }
        .board-card {
            background: white;
            border: 1px solid #ddd;
            border-radius: 8px;
            padding: 15px;
            cursor: pointer;
            box-shadow: 0 1px 3px rgba(0,0,0,0.05);
        }
        .board-title {
            font-size: 18px;
            color: var(--e-purple);
            font-weight: bold;
            text-decoration: underline;
            margin-bottom: 5px;
        }
        .post-count { font-size: 13px; color: #666; text-decoration: underline; }

        /* 投稿画面（IMG_0084を完全再現） */
        #bbs-view { display: none; background: white; padding: 20px; border-radius: 4px; border: 1px solid #ddd; }
        .form-label { font-size: 16px; margin-bottom: 5px; display: block; }
        .form-input { width: 100%; border: 1px solid #ccc; padding: 8px; margin-bottom: 15px; box-sizing: border-box; }
        .form-textarea { width: 100%; border: 1px solid #ccc; padding: 8px; margin-bottom: 15px; box-sizing: border-box; height: 100px; }
        
        .btn-group { display: flex; gap: 10px; margin-bottom: 25px; }
        .btn-green { background-color: var(--e-green); color: white; border: none; padding: 8px 20px; border-radius: 4px; font-size: 16px; cursor: pointer; }

        /* 投稿の見た目（IMG_0084再現） */
        .post-item { margin-bottom: 20px; font-size: 16px; line-height: 1.5; }
        .post-user { font-weight: bold; }
        .post-time { font-size: 13px; color: #333; margin-top: 2px; }
        .admin-del-btn { font-size: 12px; color: #ff0000; cursor: pointer; margin-left: 10px; opacity: 0.3; }
        .admin-del-btn:hover { opacity: 1; }
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
        <button class="btn-green" onclick="showTop()">戻る</button>
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
    const ADMIN_PASS = "YWRtaW4xMjM="; 
    let currentBoardKey = '';

    // 時刻フォーマット (YYYY-MM-DD HH:mm:ss)
    const getTime = (ts) => {
        const d = new Date(ts);
        return `${d.getFullYear()}-${(d.getMonth()+1).toString().padStart(2,'0')}-${d.getDate().toString().padStart(2,'0')} ${d.getHours().toString().padStart(2,'0')}:${d.getMinutes().toString().padStart(2,'0')}:${d.getSeconds().toString().padStart(2,'0')}`;
    };

    window.createNewBoard = () => {
        const t = prompt("掲示板名:");
        if (t) push(ref(db, 'boards'), { title: t });
    };

    onChildAdded(ref(db, 'boards'), (snap) => {
        const k = snap.key;
        const b = snap.val();
        const card = document.createElement('div');
        card.className = 'board-card';
        card.onclick = () => openBoard(k, b.title);
        onValue(ref(db, `messages/${k}`), (mSnap) => {
            const count = mSnap.exists() ? Object.keys(mSnap.val()).length : 0;
            card.innerHTML = `<div class="board-title">${b.title}</div><div class="post-count">投稿数: ${count}</div>`;
        });
        document.getElementById('board-list').appendChild(card);
    });

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
                <div class="post-time">${m.timestamp ? getTime(m.timestamp) : ''} <span class="admin-del-btn" onclick="delPost('${snap.key}')">[削除]</span></div>
            `;
            container.prepend(div);
        });
    };

    window.delPost = (mKey) => {
        const p = prompt("パスワード:");
        if (p && btoa(p) === ADMIN_PASS) {
            remove(ref(db, `messages/${currentBoardKey}/${mKey}`));
            document.getElementById(`msg-${mKey}`).remove();
        }
    };

    window.showTop = () => { location.reload(); };

    document.getElementById('send-btn').onclick = () => {
        const txt = document.getElementById('content').value;
        if (!txt) return;
        push(ref(db, `messages/${currentBoardKey}`), {
            username: document.getElementById('username').value || "名無しさん",
            text: txt,
            timestamp: Date.now()
        });
        document.getElementById('content').value = '';
    };
</script>
</body>
</html>
