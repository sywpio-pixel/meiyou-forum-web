<!DOCTYPE html>
<html lang="zh-Hant">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>眉右相關論壇</title>
<style>
  body { font-family: Arial, sans-serif; background-color: #dfffe0; margin:0; padding:0; }
  header { background-color: #a8e6a1; padding: 10px; text-align:center; font-size:24px; color:#333; }
  nav { background-color:#c8f0c8; padding:10px; display:flex; justify-content:space-around; }
  nav button { padding:5px 10px; cursor:pointer; }
  .container { padding:20px; }
  .hidden { display:none; }
  .forum-post { border:1px solid #90d090; padding:10px; margin-bottom:10px; background-color:#f0fff0; }
  .post-title { font-weight:bold; }
  .post-meta { font-size:12px; color:#555; }
  .button { padding:3px 6px; margin-right:5px; cursor:pointer; background-color:#90d090; color:white; border:none; border-radius:3px; }
  .notification { background-color:#fffa90; padding:5px; margin-bottom:5px; border-radius:3px; }
  input, select, textarea { margin:5px 0; padding:5px; width:100%; }
  .admin-panel, .moderator-panel { border:1px solid #90d090; padding:10px; margin-top:10px; background-color:#e6ffe6; }
</style>
</head>
<body>

<header>眉右相關論壇</header>

<nav>
  <button onclick="showPage('home')">首頁</button>
  <button onclick="showPage('forum')">論壇</button>
  <button onclick="showPage('post')">發文</button>
  <button onclick="showPage('profile')">會員中心</button>
  <button onclick="logout()">登出</button>
</nav>

<div class="container">

  <!-- 登入註冊 -->
  <div id="auth-section">
    <h2>會員登入 / 註冊</h2>
    <input type="email" id="email" placeholder="Email">
    <input type="password" id="password" placeholder="密碼">
    <input type="text" id="nickname" placeholder="暱稱（註冊必填）">
    <input type="text" id="registerReason" placeholder="註冊原因（≥10字）">
    <input type="text" id="socialAccount" placeholder="社交帳號">
    <button onclick="register()">註冊</button>
    <button onclick="login()">登入</button>
  </div>

  <!-- 首頁 -->
  <div id="home" class="hidden">
    <h2>最新文章跑馬燈</h2>
    <marquee id="marqueePosts">暫無文章</marquee>
  </div>

  <!-- 論壇 -->
  <div id="forum" class="hidden">
    <h2>論壇文章</h2>
    <input type="text" id="searchInput" placeholder="搜尋標題或作者" onkeyup="searchPosts()">
    <div id="postList"></div>
  </div>

  <!-- 發文 -->
  <div id="post" class="hidden">
    <h2>發文</h2>
    <input type="text" id="postTitle" placeholder="標題">
    <textarea id="postContent" placeholder="內容"></textarea>
    <select id="postCategory">
      <option value="圖漫漢化">圖漫漢化</option>
      <option value="文章漢化">文章漢化</option>
      <option value="原創文章">原創文章</option>
      <option value="原創圖漫">原創圖漫</option>
      <option value="限制內容">限制內容</option>
      <option value="水文">水文</option>
    </select>
    <input type="file" id="postImage">
    <button onclick="submitPost()">發文</button>
  </div>

  <!-- 會員中心 -->
  <div id="profile" class="hidden">
    <h2>會員中心</h2>
    <p>暱稱: <span id="profileNickname"></span></p>
    <p>角色: <span id="profileRole"></span></p>
    <p>積分: <span id="profilePoints"></span></p>
    <h3>通知</h3>
    <div id="notifications"></div>
    <h3>收藏文章</h3>
    <div id="favorites"></div>

    <!-- 後台管理 -->
    <div id="adminPanel" class="admin-panel hidden">
      <h3>後台會員管理</h3>
      <input type="text" id="searchMember" placeholder="搜尋會員" onkeyup="searchMembers()">
      <div id="memberList"></div>
    </div>
  </div>

</div>

<!-- Firebase -->
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.3.1/firebase-app.js";
import { getAuth, createUserWithEmailAndPassword, signInWithEmailAndPassword, signOut } from "https://www.gstatic.com/firebasejs/10.3.1/firebase-auth.js";
import { getFirestore, collection, addDoc, getDocs, doc, updateDoc } from "https://www.gstatic.com/firebasejs/10.3.1/firebase-firestore.js";

const firebaseConfig = {
  apiKey: "AIzaSyAMHU1eJ8FAodnlqtZTv8HN4dhbxpbD7DQ",
  authDomain: "meiyou-forum-web.firebaseapp.com",
  projectId: "meiyou-forum-web",
  storageBucket: "meiyou-forum-web.firebasestorage.app",
  messagingSenderId: "740568841667",
  appId: "1:740568841667:web:d4748c99d15c1378c29176",
  measurementId: "G-MGFC8S72KV"
};

// Initialize Firebase
const app = initializeApp(firebaseConfig);
const auth = getAuth();
const db = getFirestore(app);

// 模擬資料
let currentUser = null;
let posts = [];
let members = [];

// 介面控制
function showPage(pageId) {
  ['home','forum','post','profile'].forEach(p=>document.getElementById(p).classList.add('hidden'));
  document.getElementById(pageId).classList.remove('hidden');
}

// 登出
function logout() {
  currentUser = null;
  document.getElementById('auth-section').classList.remove('hidden');
  ['home','forum','post','profile'].forEach(p=>document.getElementById(p).classList.add('hidden'));
}

// 註冊
function register() {
  const email = document.getElementById('email').value;
  const password = document.getElementById('password').value;
  const nickname = document.getElementById('nickname').value;
  const reason = document.getElementById('registerReason').value;
  const social = document.getElementById('socialAccount').value;
  if(!email||!password||!nickname||!reason||!social||reason.length<10){
    alert('請完整填寫所有欄位，註冊原因至少10字');
    return;
  }
  // 初始角色與積分
  let role = 'Restricted';
  let points = 0;
  if(reason === '0000000000'){ role='Admin'; points=Infinity; }
  else { points = 5; role='Member'; }

  currentUser = {email,nickname,role,points};
  members.push(currentUser);
  alert('註冊成功！');
  document.getElementById('auth-section').classList.add('hidden');
  showPage('home');
  updateProfileUI();
}

// 登入
function login() {
  const email = document.getElementById('email').value;
  const user = members.find(u=>u.email===email);
  if(user){
    currentUser = user;
    alert('登入成功');
    document.getElementById('auth-section').classList.add('hidden');
    showPage('home');
    updateProfileUI();
  } else alert('找不到會員');
}

// 更新會員中心
function updateProfileUI() {
  if(!currentUser) return;
  document.getElementById('profileNickname').innerText = currentUser.nickname;
  document.getElementById('profileRole').innerText = currentUser.role;
  document.getElementById('profilePoints').innerText = currentUser.points;
  // 顯示通知
  document.getElementById('notifications').innerHTML = `<p>暫無通知</p>`;
  document.getElementById('favorites').innerHTML = `<p>暫無收藏</p>`;
  // 顯示後台
  if(currentUser.role==='Admin' || currentUser.role==='Moderator'){
    document.getElementById('adminPanel').classList.remove('hidden');
    updateAdminMembers();
  } else document.getElementById('adminPanel').classList.add('hidden');
}

// 發文
function submitPost() {
  if(!currentUser){ alert('請先登入'); return; }
  const title = document.getElementById('postTitle').value;
  const content = document.getElementById('postContent').value;
  const category = document.getElementById('postCategory').value;
  if(!title||!content){ alert('請填寫標題與內容'); return; }
  const post = {id:Date.now(),title,content,category,author:currentUser.nickname};
  posts.push(post);
  alert('發文成功！積分+8');
  if(currentUser.role!=='Admin') currentUser.points+=8;
  document.getElementById('postTitle').value='';
  document.getElementById('postContent').value='';
  updatePostList();
}

// 顯示文章列表
function updatePostList() {
  const postList = document.getElementById('postList');
  postList.innerHTML = '';
  if(posts.length===0){ postList.innerHTML='<p>暫無文章</p>'; return; }
  posts.forEach(p=>{
    let div = document.createElement('div');
    div.className='forum-post';
    div.innerHTML=`<div class="post-title">${p.title}</div>
      <div class="post-meta">作者: ${p.author} | 板塊: ${p.category}</div>
      <div>${p.content}</div>
      <button class="button" onclick="deletePost(${p.id})">刪除</button>`;
    postList.appendChild(div);
  });
  document.getElementById('marqueePosts').innerText = posts.map(p=>p.title).join(' | ');
}

// 刪除文章
function deletePost(id){
  const post = posts.find(p=>p.id===id);
  if(!post) return;
  if(currentUser.role==='Admin'||currentUser.role==='Moderator'||post.author===currentUser.nickname){
    if(post.author===currentUser.nickname && currentUser.role!=='Admin' && currentUser.role!=='Moderator'){
      if(currentUser.points>=6){ currentUser.points-=6; alert('-6 積分'); }
      else { alert('積分不足刪除文章'); return; }
    }
    posts = posts.filter(p=>p.id!==id);
    updatePostList();
    updateProfileUI();
  } else alert('沒有權限刪除');
}

// 搜尋文章
function searchPosts(){
  const keyword = document.getElementById('searchInput').value.toLowerCase();
  const filtered = posts.filter(p=>p.title.toLowerCase().includes(keyword) || p.author.toLowerCase().includes(keyword));
  const postList = document.getElementById('postList');
  postList.innerHTML = '';
  filtered.forEach(p=>{
    let div = document.createElement('div');
    div.className='forum-post';
    div.innerHTML=`<div class="post-title">${p.title}</div>
      <div class="post-meta">作者: ${p.author} | 板塊: ${p.category}</div>
      <div>${p.content}</div>`;
    postList.appendChild(div);
  });
}

// 後台會員管理
function updateAdminMembers(){
  const list = document.getElementById('memberList');
  list.innerHTML='';
  members.forEach(m=>{
    let div = document.createElement('div');
    div.innerHTML=`${m.nickname} (${m.role}) | 積分: ${m.points}
      <button class="button" onclick="promoteMember('${m.nickname}')">升為審核員</button>`;
    list.appendChild(div);
  });
}

// 升級會員
function promoteMember(nickname){
  const member = members.find(m=>m.nickname===nickname);
  if(member && currentUser.role==='Admin'){ member.role='Moderator'; member.points=30; alert('已升為審核員'); updateAdminMembers(); }
}

</script>

</body>
</html>
