<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>e ちゃんねる - 投稿数表示版</title>
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
        .board-title { font-size: 18px; color: var(--e-purple); font-weight: bold; text-decoration: underline; margin-bottom: 5px; padding-right: 25px; }
        .post-info { font-size: 13px; color: #666; margin-top: 2px; }
        .board-del { position: absolute; top: 10px; right: 10px; font-size: 18px; color: #ccc; cursor: pointer; border: none; background: none; }
        #bbs-view { display: none; background: white; padding: 20px; border-radius: 8px; border: 1px solid #ddd; }
        .form-label { font-size: 16px; margin-bottom: 5px; display: block; font-weight: bold; }
        .form-input { width: 150px; border: 1px solid #ccc; padding: 8px; margin-bottom: 15px; border-radius: 4px; }
        .form-textarea { width: 100%; border: 1px solid #ccc; padding: 8px; margin-bottom: 10px; border-radius: 4px; height: 100px; }
        .img-preview { max-width: 100px; max-height: 100px; display: block; margin-bottom: 10px; border: 2px dashed #ccc; }
        .btn-group { display: flex; gap: 10px; margin-bottom: 25px; }
        .btn-green { background-color: var(--e-green); color: white; border: none; padding: 10px 20px; border-radius: 4px; font-size: 16px; cursor: pointer; }
        .post-item { margin-bottom: 20px; border-bottom: 1px solid #eee; padding-bottom: 15px; }
        .post-user { font-weight: bold; }
        /* 写真がデカすぎないようにサイズ制限（最大150px） */
        .post-img { max-width: 150px; max-height: 150px; width: auto; height: auto; display: block; margin-top: 10px; border-radius: 4px; border: 1px solid #ddd; cursor: zoom-in; }
        .post-time { font-size: 12px; color: #888; margin-top: 5px; }
        .admin-del { font-size: 12px; color: #bbb; cursor: pointer; margin-left: 10px; text-decoration: underline; }
    </style>
</head>
<body>

<header id="header-part"><h1 class="logo-text">e ちゃんねる</h1></header>

<div class="container" id="top-view">
    <button class="btn-create" onclick="createNewBoard()">＋ 掲示板を作成する</button>
    <div class="board-grid" id="board-list"></div>
</div>

<div class="container" id="bbs-view">
    <input type="text" id="username" class="form-input" value="名無しさん">
    <textarea id="content" class="form-textarea" placeholder="書き込み内容..."></textarea>
    <input type="file" id="image-input" accept="image/*" style="display:none;" onchange="previewImage(this)">
    <img id="preview-area" class="img-preview" style="display:none;">
    <div class="btn-group">
        <button class="btn-green" id="send-btn">投稿</button>
        <button class="btn-green" style="background-color: #555;" onclick="document.getElementById('image-input').click()">📷 画像</button>
        <button class="btn-green" style="background-color: #888;" onclick="location.reload()">戻る</button>
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
    let compressedImageData = '';

    async function getIp() {
        try { const res = await fetch('https://api.ipify.org?format=json'); const data = await res.json(); return data.ip; }
        catch { return 'unknown'; }
    }

    async function checkBan() {
        const blackSnap = await getDoc(doc(db, 'blacklist', myId));
        if (blackSnap.exists()) {
            const data = blackSnap.data();
            if (data.type === 'perm') { location.href = "https://soulkin19.github.io/soulch_ura"; return true; }
            else if (data.type === 'temp' && Date.now() < data.until) { alert("24時間BAN中です"); return true; }
        }
        return false;
    }

    window.previewImage = (input) => {
        const file = input.files[0];
        if (!file) return;
        const reader = new FileReader();
        reader.onload = (e) => {
            const img = new Image();
            img.onload = () => {
                const canvas = document.createElement('canvas');
                let width = img.width, height = img.height, max = 300;
                if (width > height) { if (width > max) { height *= max / width; width = max; } }
                else { if (height > max) { width *= max / height; height = max; } }
                canvas.width = width; canvas.height = height;
                const ctx = canvas.getContext('2d');
                ctx.drawImage(img, 0, 0, width, height);
                compressedImageData = canvas.toDataURL('image/jpeg', 0.5);
                document.getElementById('preview-area').src = compressedImageData;
                document.getElementById('preview-area').style.display = 'block';
            };
            img.src = e.target.result;
        };
        reader.readAsDataURL(file);
    };

    // 掲示板一覧（投稿件数表示を復活）
    onSnapshot(query(collection(db, 'boards'), orderBy('lastUpdated', 'desc')), (snap) => {
        const list = document.getElementById('board-list');
        list.innerHTML = '';
        snap.forEach(d => {
            const b = d.data();
            const card = document.createElement('div');
            card.className = 'board-card';
            card.onclick = (e) => { if(e.target.className !== 'board-del') openBoard(d.id, b.title); };
            
            // 投稿件数を取得
            onSnapshot(collection(db, `boards/${d.id}/messages`), (mSnap) => {
                const count = mSnap.size;
                card.innerHTML = `
                    <button class="board-del" onclick="deleteBoard('${d.id}')">×</button>
                    <div class="board-title">${b.title}</div>
                    <div class="post-info">投稿数: ${count}</div>
                    <div class="post-info">更新: ${new Date(b.lastUpdated).toLocaleString()}</div>
                `;
            });
            list.appendChild(card);
        });
    });

    window.deleteBoard = async (id) => {
        event.stopPropagation();
        const p = prompt("パスワード:");
        if (p && btoa(p) === ADMIN_PASS) await deleteDoc(doc(db, 'boards', id));
    };

    window.createNewBoard = async () => {
        if (await checkBan()) return;
        const t = prompt("掲示板名:");
        if (t) {
            const ip = await getIp();
            await addDoc(collection(db, 'boards'), { title: t, lastUpdated: Date.now(), ownerId: myId, ip: ip });
        }
    };

    window.openBoard = (id, title) => {
        currentBoardId = id;
        document.getElementById('top-view').style.display = 'none';
        document.getElementById('bbs-view').style.display = 'block';
        const container = document.getElementById('message-container');
        onSnapshot(query(collection(db, `boards/${id}/messages`), orderBy('timestamp', 'desc')), (snap) => {
            container.innerHTML = '';
            snap.forEach(d => {
                const m = d.data();
                const div = document.createElement('div');
                div.className = 'post-item';
                div.innerHTML = `
                    <div><span class="post-user">${m.username}</span>: ${m.text}</div>
                    ${m.imageUrl ? `<img src="${m.imageUrl}" class="post-img" onclick="window.open('${m.imageUrl}')">` : ''}
                    <div class="post-time">${new Date(m.timestamp).toLocaleString()} 
                    <span class="admin-del" onclick="admin('${d.id}', '${m.uid}')">[管理]</span></div>
                `;
                container.appendChild(div);
            });
        });
    };

    document.getElementById('send-btn').onclick = async () => {
        const txt = document.getElementById('content').value;
        if (!txt && !compressedImageData) return;
        if (await checkBan()) return;
        const now = Date.now();
        const ip = await getIp();
        await addDoc(collection(db, `boards/${currentBoardId}/messages`), {
            username: document.getElementById('username').value,
            text: txt, timestamp: now, uid: myId, imageUrl: compressedImageData, ip: ip
        });
        await setDoc(doc(db, 'boards', currentBoardId), { lastUpdated: now }, { merge: true });
        document.getElementById('content').value = '';
        document.getElementById('preview-area').style.display = 'none';
        compressedImageData = '';
    };

    window.admin = async (mId, uId) => {
        const p = prompt("パス:");
        if (p && btoa(p) === ADMIN_PASS) {
            const a = prompt("1:削除 2:永久 3:24h");
            if (a === "1") await deleteDoc(doc(db, `boards/${currentBoardId}/messages`, mId));
            else if (a === "2") await setDoc(doc(db, 'blacklist', uId), { type: 'perm' });
            else if (a === "3") await setDoc(doc(db, 'blacklist', uId), { type: 'temp', until: Date.now() + (24 * 60 * 60 * 1000) });
        }
    };
</script>
</body>
</html>
