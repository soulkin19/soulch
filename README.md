<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>e ちゃんねる</title>
    <style>
        :root { --e-green: #5cb85c; --e-purple: #4b0082; --bg-light-green: #f4f9f4; }
        html, body { margin: 0; padding: 0; background-color: white; min-height: 100vh; font-family: sans-serif; }
        
        /* ヘッダー・ロゴ部分を画像に合わせる */
        header { padding: 40px 20px 20px; text-align: center; }
        .logo-text { font-size: 42px; color: var(--e-green); font-weight: bold; margin: 0; }
        .sub-title { font-size: 20px; color: var(--e-green); margin: 20px 0; font-weight: bold; }
        .logo-img { width: 80px; height: 80px; margin: 10px auto; background-color: #e3f2fd; display: flex; align-items: center; justify-content: center; border-radius: 4px; }
        .logo-img img { width: 60px; height: auto; }

        .container { max-width: 1000px; margin: 0 auto; padding: 20px; }
        
        /* ボタンのデザイン */
        .btn-create { background-color: var(--e-green); color: white; border: none; padding: 12px 24px; border-radius: 6px; font-size: 16px; cursor: pointer; margin-bottom: 25px; font-weight: bold; }
        
        /* 掲示板一覧のグリッドレイアウト */
        .board-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 15px; }
        .board-card { background: white; border: 1px solid #eee; border-radius: 8px; padding: 15px; cursor: pointer; position: relative; box-shadow: 0 2px 5px rgba(0,0,0,0.05); text-align: left; }
        .board-title { font-size: 16px; color: var(--e-purple); font-weight: bold; margin-bottom: 8px; display: block; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
        .post-count { font-size: 12px; color: #888; text-decoration: underline; }
        .board-del { position: absolute; top: 8px; right: 8px; color: #ddd; font-size: 18px; cursor: pointer; border: none; background: none; }

        /* 掲示板内部（bbs-view） */
        #bbs-view { display: none; background: white; padding: 20px; }
        .form-input { width: 100%; max-width: 200px; border: 1px solid #ccc; padding: 10px; margin-bottom: 15px; border-radius: 4px; }
        .form-textarea { width: 100%; border: 1px solid #ccc; padding: 10px; margin-bottom: 10px; border-radius: 4px; height: 120px; box-sizing: border-box; }
        .img-preview { max-width: 100px; margin-bottom: 10px; border: 1px solid #ddd; }
        .btn-group { display: flex; gap: 10px; margin-bottom: 30px; }
        .btn-green { background-color: var(--e-green); color: white; border: none; padding: 10px 25px; border-radius: 4px; cursor: pointer; font-weight: bold; }
        
        /* 投稿アイテム */
        .post-item { border-bottom: 1px solid #f0f0f0; padding: 15px 0; }
        .post-user { font-weight: bold; color: #444; }
        .post-text { margin-top: 5px; white-space: pre-wrap; word-wrap: break-word; }
        .post-img { max-width: 150px; border-radius: 4px; margin-top: 10px; border: 1px solid #eee; }
        .post-time { font-size: 11px; color: #aaa; margin-top: 5px; }
    </style>
</head>
<body>

<header id="header-part">
    <h1 class="logo-text">e ちゃんねる</h1>
    <div class="sub-title">掲示板一覧</div>
    <div class="logo-img">
        <img src="https://firebasestorage.googleapis.com/v0/b/soulkin-aa3b7.appspot.com/o/system%2Flogo_tv.png?alt=media" alt="logo" onerror="this.src='https://cdn-icons-png.flaticon.com/512/716/716429.png'">
    </div>
</header>

<div class="container" id="top-view">
    <button class="btn-create" onclick="createNewBoard()">掲示板を作成する</button>
    <div class="board-grid" id="board-list"></div>
</div>

<div class="container" id="bbs-view">
    <input type="text" id="username" class="form-input" value="名無しさん">
    <textarea id="content" class="form-textarea" placeholder="内容を入力してください"></textarea>
    <input type="file" id="image-input" accept="image/*" style="display:none;" onchange="previewImage(this)">
    <img id="preview-area" class="img-preview" style="display:none;">
    <div class="btn-group">
        <button class="btn-green" id="send-btn">投稿</button>
        <button class="btn-green" style="background-color: #666;" onclick="document.getElementById('image-input').click()">画像選択</button>
        <button class="btn-green" style="background-color: #999;" onclick="location.reload()">戻る</button>
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

    // HTMLタグを無効化するサニタイズ関数
    const sanitize = (str) => {
        return str.replace(/&/g, '&amp;')
                  .replace(/</g, '&lt;')
                  .replace(/>/g, '&gt;')
                  .replace(/"/g, '&quot;')
                  .replace(/'/g, '&#39;');
    };

    const getFp = () => {
        let id = localStorage.getItem('fp');
        if (!id) { id = 'fp_' + Math.random().toString(36).substr(2, 9); localStorage.setItem('fp', id); }
        return id;
    };
    const myId = getFp();
    let currentBoardId = '';
    let compressedImageData = '';

    async function checkBan() {
        const blackSnap = await getDoc(doc(db, 'blacklist', myId));
        if (blackSnap.exists()) {
            const data = blackSnap.data();
            if (data.type === 'perm') { location.href = "https://soulkin19.github.io/soulch_ura"; return true; }
            else if (data.type === 'temp' && Date.now() < data.until) { alert("現在制限されています"); return true; }
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

    // 掲示板一覧
    onSnapshot(query(collection(db, 'boards'), orderBy('lastUpdated', 'desc')), (snap) => {
        const list = document.getElementById('board-list');
        list.innerHTML = '';
        snap.forEach(d => {
            const b = d.data();
            const card = document.createElement('div');
            card.className = 'board-card';
            card.onclick = (e) => { if(e.target.className !== 'board-del') openBoard(d.id, b.title); };
            
            // 投稿件数のリアルタイム取得
            onSnapshot(collection(db, `boards/${d.id}/messages`), (mSnap) => {
                const count = mSnap.size;
                card.innerHTML = `
                    <button class="board-del" onclick="deleteBoard('${d.id}')">×</button>
                    <div class="board-title">${sanitize(b.title)}</div>
                    <div class="post-count">投稿数: ${count}</div>
                `;
            });
            list.appendChild(card);
        });
    });

    window.deleteBoard = async (id) => {
        event.stopPropagation();
        const p = prompt("削除パスワード:");
        if (p && btoa(p) === ADMIN_PASS) await deleteDoc(doc(db, 'boards', id));
    };

    window.createNewBoard = async () => {
        if (await checkBan()) return;
        const t = prompt("掲示板名を入力してください:");
        if (t) {
            await addDoc(collection(db, 'boards'), { 
                title: t, // title自体は入力時にサニタイズせず保存、表示時にサニタイズ
                lastUpdated: Date.now(), 
                ownerId: myId 
            });
        }
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
                div.innerHTML = `
                    <div><span class="post-user">${sanitize(m.username)}</span></div>
                    <div class="post-text">${sanitize(m.text)}</div>
                    ${m.imageUrl ? `<img src="${m.imageUrl}" class="post-img" onclick="window.open('${m.imageUrl}')">` : ''}
                    <div class="post-time">${new Date(m.timestamp).toLocaleString()} 
                    <span class="admin-del" onclick="admin('${d.id}', '${m.uid}')">[管理]</span></div>
                `;
                container.appendChild(div);
            });
        });
    };

    document.getElementById('send-btn').onclick = async () => {
        const rawTxt = document.getElementById('content').value;
        const rawName = document.getElementById('username').value;
        if (!rawTxt && !compressedImageData) return;
        if (await checkBan()) return;

        const now = Date.now();
        // 保存時にもサニタイズを考慮（表示時にも行う二重ガード）
        await addDoc(collection(db, `boards/${currentBoardId}/messages`), {
            username: rawName, 
            text: rawTxt, 
            timestamp: now, 
            uid: myId, 
            imageUrl: compressedImageData
        });
        await setDoc(doc(db, 'boards', currentBoardId), { lastUpdated: now }, { merge: true });
        
        document.getElementById('content').value = '';
        document.getElementById('preview-area').style.display = 'none';
        compressedImageData = '';
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
