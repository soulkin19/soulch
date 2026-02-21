<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Soulkin e-Board</title>
    <style>
        /* eちゃんねる風デザイン */
        body { background-color: #efefef; color: #000; font-family: "MS PGothic", "IPAMonaPGothic", sans-serif; margin: 0; display: flex; height: 100vh; }
        
        /* サイドバー */
        aside { width: 220px; background: #e0e0e0; border-right: 2px solid #ccc; display: flex; flex-direction: column; font-size: 14px; }
        .side-header { background: #666; color: white; padding: 10px; font-weight: bold; text-align: center; }
        #thread-list { flex: 1; overflow-y: auto; list-style: none; padding: 5px; margin: 0; }
        #thread-list li { padding: 8px; border-bottom: 1px solid #ccc; cursor: pointer; color: blue; text-decoration: underline; }
        #thread-list li:hover { background: #d0d0d0; }
        #thread-list li.active { background: #fff; color: #ff0000; font-weight: bold; text-decoration: none; }
        
        /* メイン */
        main { flex: 1; display: flex; flex-direction: column; background: #fff; }
        header { background: #f5f5f5; padding: 10px; border-bottom: 1px solid #ccc; font-size: 1.5rem; color: #ff0000; font-weight: bold; }
        
        #messages { flex: 1; overflow-y: auto; padding: 20px; }
        .msg { margin-bottom: 1.5rem; border-bottom: 1px dashed #ccc; padding-bottom: 10px; line-height: 1.4; }
        .msg-info { font-size: 0.9rem; color: #222; margin-bottom: 5px; }
        .res-num { color: #ff0000; font-weight: bold; margin-right: 5px; }
        .user-name { color: #228b22; font-weight: bold; }
        .user-id { color: #666; font-size: 0.8rem; }
        .msg-text { margin-top: 5px; white-space: pre-wrap; word-break: break-all; }
        .delete-btn { font-size: 10px; color: #aaa; cursor: pointer; border: none; background: none; margin-left: 10px; }

        /* 書き込みエリア */
        .form-area { background: #f5f5f5; border-top: 2px solid #ccc; padding: 15px; }
        .form-area table { width: 100%; max-width: 600px; }
        .form-area input, .form-area textarea { border: 1px solid #aaa; padding: 5px; font-family: inherit; }
        .form-area button { padding: 10px 20px; background: #eee; cursor: pointer; border: 1px solid #aaa; font-weight: bold; }
    </style>
</head>
<body>

<aside>
    <div class="side-header">スレッド一覧</div>
    <ul id="thread-list"></ul>
    <div style="padding: 10px; border-top: 1px solid #ccc;">
        <input type="text" id="new-thread-name" placeholder="新スレ名" style="width: 100px;">
        <button id="create-thread">作成</button>
    </div>
</aside>

<main>
    <header id="current-thread-title">eちゃんねる風 掲示板</header>
    <div id="messages"></div>
    
    <div class="form-area">
        <table>
            <tr><td>名前:</td><td><input type="text" id="username" value="名無しさん" style="width: 200px;"></td></tr>
            <tr><td>内容:</td><td><textarea id="content" style="width: 90%; height: 80px;"></textarea></td></tr>
            <tr><td></td><td><button id="send">書き込む</button></td></tr>
        </table>
    </div>
</main>

<script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
    import { getDatabase, ref, push, onChildAdded, onChildRemoved, remove, serverTimestamp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-database.js";

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
    let resCount = 0;

    // --- ID生成 (簡易版: 日替わり) ---
    const generateID = () => {
        const date = new Date().toLocaleDateString();
        return btoa(date).substring(0, 8);
    };

    // --- スレッド管理 ---
    const threadListRef = ref(db, 'thread_list');
    onChildAdded(threadListRef, (data) => {
        const li = document.createElement('li');
        li.innerText = data.val().name;
        li.id = `thread-${data.key}`;
        li.onclick = () => selectThread(data.key, data.val().name);
        document.getElementById('thread-list').appendChild(li);
    });

    document.getElementById('create-thread').addEventListener('click', () => {
        const name = document.getElementById('new-thread-name').value;
        if(!name) return;
        push(threadListRef, { name: name });
        document.getElementById('new-thread-name').value = '';
    });

    function selectThread(key, name) {
        currentThreadKey = key;
        resCount = 0;
        document.getElementById('current-thread-title').innerText = name;
        document.getElementById('messages').innerHTML = '';
        document.querySelectorAll('#thread-list li').forEach(el => el.classList.remove('active'));
        document.getElementById(`thread-${key}`).classList.add('active');
        loadMessages(key);
    }

    const loadMessages = (threadKey) => {
        const dbRef = ref(db, `messages/${threadKey}`);
        onChildAdded(dbRef, (data) => {
            if(currentThreadKey !== threadKey) return;
            resCount++;
            const msg = data.val();
            const div = document.createElement("div");
            div.className = "msg";
            div.id = `msg-${data.key}`;
            div.innerHTML = `
                <div class="msg-info">
                    <span class="res-num">${resCount}</span>
                    名前：<span class="user-name">${msg.username}</span> 
                    ID:<span class="user-id">${msg.id}</span>
                    <button class="delete-btn" onclick="window.deleteMsg('${data.key}')">[削除]</button>
                </div>
                <div class="msg-text">${msg.text}</div>
            `;
            document.getElementById("messages").appendChild(div);
            document.getElementById("messages").scrollTop = document.getElementById("messages").scrollHeight;
        });
        onChildRemoved(dbRef, (data) => {
            const el = document.getElementById(`msg-${data.key}`);
            if(el) el.remove();
        });
    };

    // --- 投稿 ---
    document.getElementById("send").addEventListener("click", () => {
        if(!currentThreadKey) return alert("スレを選択してください");
        const content = document.getElementById("content").value;
        if(!content) return;

        push(ref(db, `messages/${currentThreadKey}`), {
            username: document.getElementById("username").value || "名無しさん",
            text: content,
            id: generateID(),
            timestamp: serverTimestamp()
        });
        document.getElementById("content").value = "";
    });

    window.deleteMsg = (msgKey) => {
        const psw = prompt("管理者パス");
        if(psw && btoa(psw) === MASK) {
            remove(ref(db, `messages/${currentThreadKey}/${msgKey}`));
        }
    };
</script>
</body>
</html>
