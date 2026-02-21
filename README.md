<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Soulkin 掲示板</title>
    <style>
        body { font-family: 'Helvetica Neue', Arial, sans-serif; max-width: 600px; margin: 20px auto; padding: 20px; background-color: #f4f7f6; }
        h2 { color: #333; text-align: center; }
        #messages { background: white; border: 1px solid #ddd; height: 400px; overflow-y: scroll; padding: 15px; border-radius: 8px; margin-bottom: 20px; box-shadow: 0 2px 5px rgba(0,0,0,0.1); }
        .msg { border-bottom: 1px solid #eee; padding: 10px 0; }
        .name { font-weight: bold; color: #007bff; margin-right: 10px; }
        .text { color: #555; }
        .input-area { display: flex; flex-direction: column; gap: 10px; }
        input, textarea { padding: 10px; border: 1px solid #ccc; border-radius: 4px; font-size: 16px; }
        button { padding: 10px; background-color: #007bff; color: white; border: none; border-radius: 4px; cursor: pointer; font-size: 16px; }
        button:hover { background-color: #0056b3; }
    </style>
</head>
<body>

    <h2>Soulkin 掲示板</h2>
    <div id="messages"></div>

    <div class="input-area">
        <input type="text" id="username" placeholder="名前">
        <textarea id="content" placeholder="メッセージ内容" rows="3"></textarea>
        <button id="send">送信する</button>
    </div>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
        import { getDatabase, ref, push, onChildAdded } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-database.js";

        // あなたのプロジェクト設定
        const firebaseConfig = {
            apiKey: "AIzaSyCwhHspaG94goiCIjVj3h-Un5pBK3JTjMU",
            authDomain: "soulkin-aa3b7.firebaseapp.com",
            databaseURL: "https://soulkin-aa3b7-default-rtdb.firebaseio.com",
            projectId: "soulkin-aa3b7",
            storageBucket: "soulkin-aa3b7.firebasestorage.app",
            messagingSenderId: "358331064206",
            appId: "1:358331064206:web:d7760ea0919259418a4edf"
        };

        // Firebase初期化
        const app = initializeApp(firebaseConfig);
        const db = getDatabase(app);
        const dbRef = ref(db, "chat");

        // 送信ボタンの処理
        document.getElementById("send").addEventListener("click", () => {
            const username = document.getElementById("username").value;
            const content = document.getElementById("content").value;
            
            if(!username || !content) {
                alert("名前とメッセージを入力してください");
                return;
            }

            push(dbRef, {
                username: username,
                text: content,
                timestamp: Date.now()
            });

            document.getElementById("content").value = "";
        });

        // データの受信表示
        onChildAdded(dbRef, (data) => {
            const msg = data.val();
            const div = document.createElement("div");
            div.className = "msg";
            div.innerHTML = `<span class="name">${msg.username}</span><span class="text">${msg.text}</span>`;
            document.getElementById("messages").appendChild(div);
            
            // 自動スクロール
            const msgBox = document.getElementById("messages");
            msgBox.scrollTop = msgBox.scrollHeight;
        });
    </script>
</body>
</html>
