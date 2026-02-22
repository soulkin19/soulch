<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>e ちゃんねる</title>
    <style>
        :root { --e-green: #5cb85c; --e-purple: #4b0082; --bg-light-green: #f4f9f4; }
        html, body { margin: 0; padding: 0; background-color: var(--bg-light-green); min-height: 100vh; font-family: sans-serif; }
        header { background-color: white; padding: 30px 20px; text-align: center; border-bottom: 1px solid #ddd; }
        .logo-text { font-size: 40px; color: var(--e-green); font-weight: bold; margin: 0; }
        .sub-title { font-size: 20px; color: var(--e-green); margin: 10px 0; font-weight: bold; }
        .container { max-width: 1000px; margin: 0 auto; padding: 20px; }
        .btn-create { background-color: var(--e-green); color: white; border: none; padding: 10px 15px; border-radius: 4px; font-size: 16px; cursor: pointer; margin-bottom: 20px; }
        .board-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(300px, 1fr)); gap: 15px; }
        .board-card { background: white; border: 1px solid #ddd; border-radius: 8px; padding: 15px; cursor: pointer; position: relative; }
        .board-title { font-size: 18px; color: var(--e-purple); font-weight: bold; text-decoration: underline; margin-bottom: 5px; padding-right: 20px; }
        .post-info { font-size: 13px; color: #666; text-decoration: underline; margin-top: 2px; }
        .board-del { position: absolute; top: 10px; right: 10px; font-size: 14px; color: #ccc; cursor: pointer; }
        #bbs-view { display: none; background: white; padding: 20px; border-radius: 4px; border: 1px solid #ddd; }
        .form-label { font-size: 16px; margin-bottom: 5px; display: block; }
        .form-input { width: 150px; border: 1px solid #ccc; padding: 8px; margin-bottom: 15px; box-sizing: border-box; }
        .form-textarea { width: 100%; border: 1px solid #ccc; padding: 8px; margin-bottom: 10px; box-sizing: border-box; height: 100px; }
        .btn-group { display: flex; gap: 10px; margin-bottom: 25px; align-items: center; }
        .btn-green { background-color: var(--e-green); color: white; border: none; padding: 8px 20px; border-radius: 4px; font-size: 16px; cursor: pointer; }
        .post-item { margin-bottom: 20px; font-size: 16px; border-bottom: 1px solid #eee; padding-bottom: 10px; word-break: break-all; }
        .post-user { font-weight: bold; }
        .post-content { margin-top: 5px; white-space: pre-wrap; } /* 改行を有効にしつつタグを無効化 */
        .post-time { font-size: 12px; color: #888; margin-top: 5px; }
        .admin-del { font-size: 11px; color: #ccc; cursor: pointer; margin-left: 10px; }
        @media (max-width: 600px) {
            header { padding: 15px 10px; }
            .logo-text { font-size: 30px; }
            .container { padding: 10px; }
            .form-textarea { height: 60px; }
        }
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
        <button class="btn-green" onclick="location.reload()">戻る</button>
    </div>
    <div id="message-container"></div>
</div>

<script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
    import { getFirestore, collection, addDoc, onSnapshot, query, orderBy, doc, deleteDoc, getDoc, setDoc } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";

    const firebaseConfig = {
        apiKey: "AIzaSyCwhHspaG94goiCIjVj3h-Un5pBK3JTjMU",
        authDomain: "soulkin-aa3b7.firebaseapp.com",
        projectId: "soulkin-aa3b7",
        storageBucket: "soulkin-aa3b7.firebasestorage.app",
        messagingSenderId: "358331064206",
        appId: "1:358331064206:web:d7760ea0919259418a4edf"
    };

    const app = initializeApp(firebaseConfig);
    const db = getFirestore(app);
    const ADMIN_PASS = "ZGVsdGE0Mzc="; 

    const getFp = () => {
        let id = localStorage.getItem('fp');
        if (!id) { id = 'fp_' + Math.random().toString(36).substr(2, 9); localStorage.setItem('fp', id); }
        return id;
    };
    const myId = getFp();
    let currentBoardId = '';

    onSnapshot(query(collection(db, 'boards'), orderBy('lastUpdated', 'desc')), (snap) => {
        const list = document.getElementById('board-list');
        list.innerHTML = '';
        snap.forEach(d => {
            const b = d.data();
            const card = document.createElement('div');
            card.className = 'board-card';
            card.onclick = (e) => { if(e.target.className !== 'board-del') openBoard(d.id, b.title); };
            onSnapshot(collection(db, `boards/${d.id}/messages`), (mSnap) => {
                const count = mSnap.size;
                const lastTime = b.lastUpdated ? new Date(b.lastUpdated).toLocaleString() : 'なし';
                card.innerHTML = `<div class="board-del" onclick="deleteBoard('${d.id}', '${b.ownerId}')">×</div>
                                  <div class="board-title"></div>
                                  <div class="post-info">投稿数: ${count}</div>
                                  <div class="post-info">最終投稿: ${lastTime}</div>`;
                card.querySelector('.board-title').innerText = b.title; // タイトルのタグ無効化
            });
            list.appendChild(card);
        });
    });

    window.deleteBoard = async (id, ownerId) => {
        const p = prompt("削除パスワード:");
        if (p && btoa(p) === ADMIN_PASS) {
            const a = prompt("1:削除のみ 2:削除+作成者を永久BAN 3:削除+作成者を24時間BAN");
            if (a === "1" || a === "2" || a === "3") {
                if(confirm("実行しますか？")) {
                    await deleteDoc(doc(db, 'boards', id));
                    if (a === "2" && ownerId) await setDoc(doc(db, 'blacklist', ownerId), { type: 'perm' });
                    if (a === "3" && ownerId) await setDoc(doc(db, 'blacklist', ownerId), { type: 'temp', until: Date.now() + (24 * 60 * 60 * 1000) });
                }
            }
        }
    };

    window.createNewBoard = async () => {
        const t = prompt("掲示板名:");
        if (t) await addDoc(collection(db, 'boards'), { title: t, lastUpdated: Date.now(), ownerId: myId });
    };

    window.openBoard = (id, title) => {
        currentBoardId = id;
        document.getElementById('top-view').style.display = 'none';
        document.getElementById('header-part').style.display = 'none';
        document.getElementById('bbs-view').style.display = 'block';
        const container = document.getElementById('message-container');
        onSnapshot(query(collection(db, `boards/${id}/messages`), orderBy('timestamp', 'desc')), (snap) => {
            container.innerHTML = '';
            snap.forEach(d => {
                const m = d.data();
                const div = document.createElement('div');
                div.className = 'post-item';
                
                // 構造を作成
                div.innerHTML = `<div><span class="post-user"></span>: <span class="post-content"></span></div>
                    <div class="post-time">${new Date(m.timestamp).toLocaleString()} 
                    <span class="admin-del" onclick="admin('${d.id}', '${m.uid}')">[管理]</span></div>`;
                
                // innerTextを使ってテキストのみを注入（HTMLタグを無効化）
                div.querySelector('.post-user').innerText = m.username;
                div.querySelector('.post-content').innerText = m.text;
                
                container.appendChild(div);
            });
        });
    };

    document.getElementById('send-btn').onclick = async () => {
        const txt = document.getElementById('content').value;
        if (!txt) return;
        
        const blackSnap = await getDoc(doc(db, 'blacklist', myId));
        if (blackSnap.exists()) {
            const data = blackSnap.data();
            if (data.type === 'perm') {
                location.href = "https://soulkin19.github.io/soulch_ura";
                return;
            } else if (data.type === 'temp' && Date.now() < data.until) {
                const rest = Math.ceil((data.until - Date.now()) / (1000 * 60 * 60));
                alert(`24時間BANされています。あと約${rest}時間投稿できません。`);
                return;
            }
        }

        const now = Date.now();
        await addDoc(collection(db, `boards/${currentBoardId}/messages`), {
            username: document.getElementById('username').value,
            text: txt,
            timestamp: now,
            uid: myId
        });
        await setDoc(doc(db, 'boards', currentBoardId), { lastUpdated: now }, { merge: true });

        document.getElementById('content').value = '';
    };

    window.admin = async (mId, uId) => {
        const p = prompt("パスワード:");
        if (p && btoa(p) === ADMIN_PASS) {
            const a = prompt("1:削除 2:永久BAN 3:24時間BAN");
            if (a === "1") await deleteDoc(doc(db, `boards/${currentBoardId}/messages`, mId));
            else if (a === "2") await setDoc(doc(db, 'blacklist', uId), { type: 'perm' });
            else if (a === "3") await setDoc(doc(db, 'blacklist', uId), { type: 'temp', until: Date.now() + (24 * 60 * 60 * 1000) });
        }
    };
</script>
</body>
</html>
