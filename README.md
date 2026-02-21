<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Soulkin Custom Board</title>
    <style>
        :root { --main-color: #007bff; --bg-color: #f0f2f5; }
        body { font-family: sans-serif; background: var(--bg-color); margin: 0; display: flex; flex-direction: row; height: 100vh; }
        
        /* サイドバー（スレッド一覧） */
        aside { width: 250px; background: #fff; border-right: 1px solid #ddd; display: flex; flex-direction: column; }
        .side-header { padding: 15px; background: var(--main-color); color: white; font-weight: bold; }
        #thread-list { flex: 1; overflow-y: auto; list-style: none; padding: 0; margin: 0; }
        #thread-list li { padding: 12px 15px; border-bottom: 1px solid #eee; cursor: pointer; transition: 0.2s; }
        #thread-list li:hover { background: #e9ecef; }
        #thread-list li.active { background: #cfe2ff; border-left: 5px solid var(--main-color); }
        .add-thread { padding: 10px; border-top: 1px solid #ddd; }

        /* メインコンテンツ */
        main { flex: 1; display: flex; flex-direction: column; background: var(--bg-color); }
        header { background: #fff; padding: 15px; border-bottom: 1px solid #ddd; font-weight: bold; font-size: 1.2rem; }
        
        #messages { flex: 1; overflow-y: auto; padding: 20px; }
        .msg { background: white; padding: 12px; border-radius: 8px; margin-bottom: 10px; position: relative; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
        .name { font-weight: bold; color: var(--main-color); display: block; margin-bottom: 4px; }
        .delete-btn { position: absolute; top: 10px; right: 10px; color: #ccc; cursor: pointer; border: none; background: none; font-size: 12px; }

        /* 入力エリア */
        .input-area { background: white; padding: 20px; border-top: 1px solid #ddd; display: flex; flex-direction: column; gap: 10px; }
        .input-area input, .input-area textarea { padding: 10px; border: 1px solid #ddd; border-radius: 4px; }
        .input-area button { padding: 12px; background: var(--main-color); color: white; border: none; border-radius: 4px; cursor: pointer; }

        @media (max-width: 600px) {
            body { flex-direction: column; }
            aside { width: 100%; height: 200px; }
        }
    </style>
</head>
<body>

<aside>
    <div class="side-header">スレッド一覧</div>
    <ul id="thread-list"></ul>
    <div class="add-thread">
        <input type="text" id="new-thread-name" placeholder="新しいスレッド名" style="width: 80%;">
        <button id="create-thread" style="width: 100%; margin-top: 5px;">作成</button>
    </div>
</aside>

<main>
    <header id="current-thread-title">スレッドを選択してください</header>
    <div id="messages"></div>
    <div class="input-area">
        <input type="text" id="username" placeholder="名前">
        <textarea id="content" placeholder="メッセージ内容" rows="3"></textarea>
        <button id="send">投稿する</button>
    </div>
</main>

<script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
    import { getDatabase, ref, push, onChildAdded, onChildRemoved, remove, set } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-database.js";

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
    const MASK = "YWRtaW4xMjM="; // admin123

    let currentThreadKey = ''; 
    let msgUnsubscribe = null;

    // --- スレッド管理 ---
    const threadListRef = ref(db, 'thread_list');

    // スレッドの作成
    document.getElementById('create-thread').addEventListener('click', () => {
        const name = document.getElementById('new-thread-name').value;
        if(!name) return;
        push(threadListRef, { name: name });
        document.getElementById('new-thread-name').value = '';
    });

    // スレッド一覧の読み込み
    onChildAdded(threadListRef, (data) => {
        const li = document.createElement('li');
        li.innerText = data.val().name;
        li.id = `thread-${data.key}`;
        li.onclick = () => selectThread(data.key, data.val().name);
        document.getElementById('thread-list').appendChild(li);
    });

    // スレッド選択
    function selectThread(key, name) {
        currentThreadKey = key;
        document.getElementById('current-thread-title').innerText = name;
        document.getElementById('messages').innerHTML = '';
        
        // アクティブ表示の切り替え
        document.querySelectorAll('#thread-list li').forEach(el => el.classList.remove('active'));
        document.getElementById(`thread-${key}`).classList.add('active');

        // メッセージ監視（既存の監視はリセットできない簡易版のため、新しく追加されたものだけ表示するように制御が必要ですが、今回はシンプルに都度取得）
        loadMessages(key);
    }

    const loadMessages = (threadKey) => {
        const dbRef = ref(db, `messages/${threadKey}`);
        onChildAdded(dbRef, (data) => {
            // 他のスレッドのメッセージが混ざらないようチェック
            if(currentThreadKey !== threadKey) return;

            const msg = data.val();
            const div = document.createElement("div");
            div.className = "msg";
            div.id = `msg-${data.key}`;
            div.innerHTML = `
                <span class="name">${msg.username}</span>
                <span class="text">${msg.text}</span>
                <button class="delete-btn" onclick="window.deleteMsg('${data.key}')">削除</button>
            `;
            document.getElementById("messages").appendChild(div);
            document.getElementById("messages").scrollTop = document.getElementById("messages").scrollHeight;
        });

        onChildRemoved(dbRef, (data) => {
            const el = document.getElementById(`msg-${data.key}`);
            if(el) el.remove();
        });
    };

    // --- 投稿・削除 ---
    document.getElementById("send").addEventListener("click", () => {
        if(!currentThreadKey) return alert("スレッドを選択してください");
        const username = document.getElementById("username").value || "名無し";
        const content = document.getElementById("content").value;
        if(!content) return;

        push(ref(db, `messages/${currentThreadKey}`), {
            username: username,
            text: content,
            timestamp: Date.now()
        });
        document.getElementById("content").value = "";
    });

    window.deleteMsg = (msgKey) => {
        const psw = prompt("管理者パスワードを入力してください");
        if(psw && btoa(psw) === MASK) {
            remove(ref(db, `messages/${currentThreadKey}/${msgKey}`));
        } else {
            alert("不一致");
        }
    };
</script>

</body>
</html>
