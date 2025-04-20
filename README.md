<?php
// index.php - GoldenTok Mobile-First TikTok-Like Interface

// Ensure uploads directory and posts storage exist
$uploadDir = __DIR__ . '/uploads';
$postsFile = $uploadDir . '/posts.json';
if (!is_dir($uploadDir)) {
    mkdir($uploadDir, 0755, true);
}
if (!file_exists($postsFile)) {
    file_put_contents($postsFile, json_encode([]));
}

// Handle AJAX actions
if (isset($_GET['action'])) {
    header('Content-Type: application/json');
    $posts = json_decode(file_get_contents($postsFile), true);
    switch ($_GET['action']) {
        case 'get_posts':
            // Return posts in reverse chronological order
            echo json_encode(array_reverse($posts));
            exit;
        case 'like_post':
            // Toggle like
            foreach ($posts as &$p) {
                if ($p['id'] === $_POST['post_id']) {
                    $user = htmlspecialchars($_POST['user']);
                    if (!in_array($user, $p['likes'])) {
                        $p['likes'][] = $user;
                    } else {
                        $p['likes'] = array_filter($p['likes'], fn($u) => $u !== $user);
                    }
                    break;
                }
            }
            file_put_contents($postsFile, json_encode($posts));
            echo json_encode(['success' => true]);
            exit;
        case 'add_comment':
            // Add comment to specific post
            foreach ($posts as &$p) {
                if ($p['id'] === $_POST['post_id']) {
                    $p['comments'][] = [
                        'autor' => htmlspecialchars($_POST['user']),
                        'texto' => nl2br(htmlspecialchars($_POST['texto']))
                    ];
                    break;
                }
            }
            file_put_contents($postsFile, json_encode($posts));
            echo json_encode(['success' => true]);
            exit;
    }
}

// Handle new post submission
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_POST['texto'])) {
    $posts = json_decode(file_get_contents($postsFile), true);
    $id = uniqid();
    $media = null;
    if (!empty($_FILES['arquivo']['name']) && $_FILES['arquivo']['error'] === UPLOAD_ERR_OK) {
        $ext = pathinfo($_FILES['arquivo']['name'], PATHINFO_EXTENSION);
        $fileName = "$id.$ext";
        move_uploaded_file($_FILES['arquivo']['tmp_name'], "$uploadDir/$fileName");
        $media = ['file' => $fileName, 'type' => strtolower($ext)];
    }
    $posts[] = [
        'id' => $id,
        'autor' => $_POST['user'],
        'texto' => nl2br(htmlspecialchars($_POST['texto'])),
        'media' => $media,
        'likes' => [],
        'comments' => [],
        'timestamp' => time()
    ];
    file_put_contents($postsFile, json_encode($posts));
    header('Location: index.php');
    exit;
}
?><!DOCTYPE html><html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>GoldenTok</title>
    <style>
        /* Mobile-first, full-screen TikTok-like styling */
        html, body { height:100%; margin:0; background:#000; color:#fff; font-family:sans-serif; }
        .topbar {
            position:fixed; top:0; left:0; right:0; height:50px;
            background:rgba(0,0,0,0.8); display:flex; align-items:center;
            padding:0 16px; z-index:10;
        }
        .logo { font-size:1.2rem; font-weight:bold; color:#ff0050; flex:1; }
        .btn-user { background:none; border:none; font-size:1.6rem; color:#fff; cursor:pointer; }
        #feed {
            position:absolute; top:50px; bottom:0; width:100%; overflow-y:scroll;
            scroll-snap-type:y mandatory; -webkit-overflow-scrolling:touch;
        }
        .post {
            position:relative; height:100%; /* fit between topbar and bottom */
            scroll-snap-align:start; display:flex; align-items:center; justify-content:center;
        }
        .post-media {
            width:100%; height:100%; object-fit:cover;
        }
        .overlay {
            position:absolute; bottom:80px; left:12px; right:12px;
            text-shadow:0 0 5px rgba(0,0,0,0.7); max-width:75%;
        }
        .overlay .author { font-weight:bold; margin-bottom:4px; }
        .overlay .text { line-height:1.2; }
        .actions {
            position:absolute; right:12px; bottom:100px; display:flex; flex-direction:column; align-items:center; gap:8px;
        }
        .actions button {
            background:transparent; border:none; font-size:2.2rem; width:48px; height:48px; cursor:pointer;
        }
        .actions .like-count {
            font-size:1rem; margin-top:-8px;
        }
        #btn-create {
            position:fixed; bottom:24px; left:50%; transform:translateX(-50%); /* center bottom */
            width:56px; height:56px; border-radius:50%; border:none; background:#ff0050; color:#fff; font-size:2rem; z-index:10; box-shadow:0 2px 10px rgba(0,0,0,0.3); cursor:pointer;
        }
        /* Post Modal */
        #postModal {
            display:none; position:fixed; inset:0; background:rgba(0,0,0,0.8); justify-content:center; align-items:center; z-index:20;
        }
        #postModal .box {
            background:#222; padding:20px; border-radius:12px; width:90%; max-width:400px;
        }
        #postModal textarea { width:100%; height:100px; margin-bottom:12px; background:#000; color:#fff; border:1px solid #555; padding:8px; border-radius:8px; resize:none; }
        #postModal input[type=file] { width:100%; margin-bottom:12px; }
        #postModal .btn-submit { width:100%; padding:12px; border:none; border-radius:8px; background:#ff0050; color:#fff; font-size:1rem; cursor:pointer; }
        /* Comment Modal */
        #commentModal { display:none; position:fixed; inset:0; background:rgba(0,0,0,0.8); justify-content:center; align-items:center; z-index:20; }
        #commentModal .box { background:#222; border-radius:12px; width:90%; max-width:400px; max-height:80%; display:flex; flex-direction:column; overflow:hidden; }
        #commentModal .header { display:flex; justify-content:space-between; align-items:center; padding:12px; border-bottom:1px solid #444; }
        #commentModal .header .title { font-size:1.1rem; }
        #commentModal .header .btn-close { background:none; border:none; font-size:1.5rem; color="#fff"; cursor:pointer; }
        #commentModal .comments-list { flex:1; overflow-y:auto; padding:12px; }
        #commentModal textarea { width:100%; height:60px; border:none; padding:8px; background:#000; color:#fff; resize:none; box-sizing:border-box; }
        #commentModal .btn-send { border:none; background:#ff0050; color:#fff; padding:12px; font-size:1rem; cursor:pointer; }
    </style>
</head>
<body>
    <div class="topbar">
        <div class="logo">GoldenTok</div>
        <button class="btn-user" id="btn-user">👤</button>
    </div>
    <div id="feed"></div>
    <button id="btn-create">+</button><!-- Post Modal -->
<div id="postModal">
    <div class="box">
        <form method="post" enctype="multipart/form-data">
            <textarea name="texto" placeholder="O que está pensando?" required></textarea>
            <input type="file" name="arquivo" accept="image/*,video/*"><br>
            <input type="hidden" name="user" id="input-user">
            <button type="submit" class="btn-submit">Postar</button>
        </form>
    </div>
</div>

<!-- Comment Modal -->
<div id="commentModal">
    <div class="box">
        <div class="header">
            <div class="title">Comentários</div>
            <button class="btn-close" id="btn-close-comments">✕</button>
        </div>
        <div class="comments-list" id="commentsList"></div>
        <textarea id="commentText" placeholder="Escreva um comentário..."></textarea>
        <button class="btn-send" id="btn-send-comment">Enviar</button>
    </div>
</div>

<script>
    // Ask new users for name
    let user = localStorage.getItem('gs_user');
    if (!user) {
        user = prompt('Qual seu nome?');
        localStorage.setItem('gs_user', user);
    }
    document.getElementById('input-user').value = user;
    document.getElementById('btn-user').onclick = () => {
        let n = prompt('Novo nome:', user);
        if (n) { user = n; localStorage.setItem('gs_user', user); document.getElementById('input-user').value = user; }
    };
    // Modals
    const postModal = document.getElementById('postModal');
    const commentModal = document.getElementById('commentModal');
    document.getElementById('btn-create').onclick = () => postModal.style.display = 'flex';
    postModal.onclick = e => { if (e.target === postModal) postModal.style.display = 'none'; };
    document.getElementById('btn-close-comments').onclick = () => commentModal.style.display = 'none';
    // IntersectionObserver to auto-play/pause videos
    const observer = new IntersectionObserver((entries) => {
        entries.forEach(entry => {
            const v = entry.target;
            if (entry.isIntersecting) { v.play(); } else { v.pause(); }
        });
    }, { threshold: 0.75 });
    // Load posts
    async function loadPosts() {
        const res = await fetch('?action=get_posts');
        const posts = await res.json();
        const feed = document.getElementById('feed');
        feed.innerHTML = '';
        posts.forEach(p => {
            const div = document.createElement('div'); div.className = 'post';
            // Media element creation
            if (p.media) {
                if (['jpg','jpeg','png','gif'].includes(p.media.type)) {
                    const img = document.createElement('img');
                    img.className = 'post-media';
                    img.src = 'uploads/' + p.media.file;
                    div.appendChild(img);
                } else {
                    const vid = document.createElement('video');
                    vid.className = 'post-media';
                    vid.src = 'uploads/' + p.media.file;
                    vid.playsInline = true;
                    vid.loop = true;
                    vid.controls = true;
                    vid.autoplay = true; vid.muted = false;
                    // Toggle play/pause on click
                    vid.addEventListener('click', () => {
                        if (vid.paused) vid.play();
                        else vid.pause();
                    });
                    observer.observe(vid);
                    div.appendChild(vid);
                }
            }
            // Overlay and actions
            const overlay = document.createElement('div'); overlay.className = 'overlay';
            overlay.innerHTML = `<div class="author">${p.autor}</div><div class="text">${p.texto}</div>`;
            const actions = document.createElement('div'); actions.className = 'actions';
            // Updated to include like count next to heart
            actions.innerHTML = `<button class="btn-like">${p.likes.includes(user)? '❤️':'🤍'}</button><span class="like-count">${p.likes.length}</span><button class="btn-comment">💬</button>`;
            div.appendChild(overlay);
            div.appendChild(actions);
            // Like button handler
            actions.querySelector('.btn-like').onclick = async () => {
                await fetch('?action=like_post', {
                    method: 'POST',
                    headers: {'Content-Type':'application/x-www-form-urlencoded'},
                    body: `post_id=${p.id}&user=${encodeURIComponent(user)}`
                });
                loadPosts();
            };
            // Comment button handler
            actions.querySelector('.btn-comment').onclick = () => {
                document.getElementById('commentsList').innerHTML =
                    p.comments.map(c => `<div><b>${c.autor}:</b> ${c.texto}</div>`).join('');
                commentModal.style.display = 'flex';
                document.getElementById('btn-send-comment').onclick = async () => {
                    const txt = document.getElementById('commentText').value.trim();
                    if (!txt) return;
                    await fetch('?action=add_comment', {
                        method:'POST',
                        headers:{'Content-Type':'application/x-www-form-urlencoded'},
                        body: `post_id=${p.id}&user=${encodeURIComponent(user)}&texto=${encodeURIComponent(txt)}`
                    });
                    commentModal.style.display = 'none';
                    loadPosts();
                };
            };
            feed.appendChild(div);
        });
    }
    loadPosts();
</script>

</body>
</html>
