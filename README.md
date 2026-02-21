<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Soulkin Multi-Thread Board</title>
    <style>
        :root { --main-color: #007bff; --bg-color: #f0f2f5; }
        body { font-family: sans-serif; background: var(--bg-color); margin: 0; display: flex; flex-direction: column; height: 100vh; }
        header { background: var(--main-color); color: white; padding: 1rem; text-align: center; box-shadow: 0 2px 5px rgba(0,0,0,0.2); }
        
        /* スレッド切り替えエリア */
        #thread-selector { background: #fff; padding: 10px; border-bottom: 1px solid #ddd; display: flex; gap: 10px; justify-content: center; }
        .thread-btn { border: 1px solid var(--main-color); background: none; color: var(--main-color); padding: 5px 15px; border-radius: 20px; cursor: pointer; }
        .thread-btn.active { background: var(--main-color); color: white; }

        /* メッセージエリア */
        #messages { flex: 1; overflow-y: auto; padding: 20px; max-width: 800px; margin: 0 auto; width: 100%; box-sizing: border-box; }
        .msg { background: white; padding: 12px; border-radius: 8px; margin-bottom: 10px; position: relative; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
        .name { font-weight: bold; color: var(--main-color); display: block; margin-bottom: 4px; }
        .delete-btn { position: absolute; top: 10px; right: 10px; color: #ccc; cursor: pointer; border: none; background: none; font-size: 12px; }
        .delete-btn:hover { color: red; }

        /* 入力エリア */
        .input-area { background: white; padding: 20px; border-top: 1px solid #ddd; display: flex; flex-direction: column; gap: 10px; align-items: center; }
        .input-area input, .input-area textarea { width: 100%; max-width: 600px; padding: 10px; border: 1px solid #ddd; border-radius: 4px; }
        .input-area button { width: 100%; max-width: 600px; padding: 12px; background: var(--main-color); color: white; border: none; border-radius: 4px; font-weight: bold; cursor: pointer; }
    </style>
</head>
<body>

<header>
    <h2 id="current-thread-title">メイン掲示板</h2>
</header>

<div id="thread-selector">
    <button class="thread-btn active" onclick="switchThread('main')">メイン</button>
    <button class="thread-btn" onclick="switchThread('game')">ゲーム</button>
    <button class="thread-btn" onclick="switchThread('free')">雑談</button>
</div>

<div id="messages"></div>

<div class="input-area">
    <input type="text" id="username" placeholder="名前">
    <textarea id="content" placeholder="メッセージ内容 (Shift+Enterで送信)" rows="3"></textarea>
    <button id="send">投稿する</button>
</div>

<script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
    import { getDatabase, ref, push, onChildAdded, onChildRemoved, remove } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-database.js";

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
    
    let currentThread = 'main'; // 現在のスレッドID
    let unsubscribe = null; // 監視解除用

    // 管理者パスワードのヒント（"admin123" を btoa でエンコードしたもの）
    // F12で見られても一瞬「？」となる程度の気休めです
    const MASK = "YWRtaW4xMjM="; 

    // スレッド切り替え関数
    window.switchThread = (id) => {
        currentThread = id;
        document.getElementById('messages').innerHTML = ''; // 画面クリア
        document.getElementById('current-thread-title').innerText = id === 'main' ? 'メイン掲示板' : id === 'game' ? 'ゲーム実況' : '雑談スレ';
        
        // ボタンの見た目更新
        document.querySelectorAll('.thread-btn').forEach(btn => btn.classList.remove('active'));
        event.target.classList.add('active');

        loadMessages();
    };

    const loadMessages = () => {
        const dbRef = ref(db, `threads/${currentThread}`);
        
        // メッセージの受信
        onChildAdded(dbRef, (data) => {
            const msg = data.val();
            const key = data.key;
            const div = document.createElement("div");
            div.className = "msg";
            div.id = `msg-${key}`;
            div.innerHTML = `
                <span class="name">${msg.username}</span>
                <span class="text">${msg.text}</span>
                <button class="delete-btn" onclick="deleteMessage('${key}')">削除</button>
            `;
            document.getElementById("messages").appendChild(div);
            document.getElementById("messages").scrollTop = document.getElementById("messages").scrollHeight;
        });

        // 削除のリアルタイム反映
        onChildRemoved(dbRef, (data) => {
            const el = document.getElementById(`msg-${data.key}`);
            if(el) el.remove();
        });
    };

    // 送信処理
    document.getElementById("send").addEventListener("click", () => {
        const username = document.getElementById("username").value || "名無し";
        const content = document.getElementById("content").value;
        if(!content) return;

        push(ref(db, `threads/${currentThread}`), {
            username: username,
            text: content,
            timestamp: Date.now()
        });
        document.getElementById("content").value = "";
    });

    // 削除処理（グローバル関数として登録）
    window.deleteMessage = (key) => {
        const psw = prompt("管理者パスワードを入力してください");
        if(psw && btoa(psw) === MASK) {
            remove(ref(db, `threads/${currentThread}/${key}`));
        } else {
            alert("パスワードが違います");
        }
    };

    // 初期読み込み
    loadMessages();
</script>

</body>
</html>
