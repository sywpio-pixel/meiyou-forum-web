<!DOCTYPE html>
<html lang="zh-TW">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>眉右相關論壇</title>
<style>
body{ font-family: Arial; background:#dfffe0; color:#333; margin:0; padding:0;}
header{ display:flex; justify-content:space-between; align-items:center; padding:10px; background:#a8e6cf;}
nav{ padding:10px; background:#e0f7da;}
button{ cursor:pointer; margin:2px; }
.formDiv{ display:none; padding:10px; background:#f0fff4; margin:10px 0; border-radius:5px;}
.postItem{ padding:5px; margin:5px; background:#e6fff5; border-radius:5px;}
.marquee{ background:#c1f0d4; padding:5px; margin:5px 0;}
#adminPanel{ display:none; padding:10px; background:#b0f0c2; margin:10px; border-radius:5px;}
.notification{ background:#ffffc1; padding:5px; margin:5px 0; border-radius:5px;}
</style>
</head>
<body>

<header>
  <h1>眉右相關論壇</h1>
  <div>
    <button id="btnLogin" onclick="showDiv('loginForm')">登入</button>
    <button id="btnRegister" onclick="showDiv('registerForm')">註冊</button>
    <button id="btnMemberCenter" onclick="showDiv('memberCenter')" style="display:none;">會員中心</button>
    <button id="btnLogout" onclick="logout()" style="display:none;">登出</button>
  </div>
</header>

<nav>
  <input type="text" id="searchInput" placeholder="搜尋文章">
  <button onclick="searchPosts()">搜尋</button>
</nav>

<!-- 登入 -->
<div class="formDiv" id="loginForm">
  <h3>登入</h3>
  <input id="loginEmail" type="email" placeholder="Email"><br>
  <input id="loginPass" type="password" placeholder="密碼"><br>
  <button onclick="login()">登入</button>
</div>

<!-- 註冊 -->
<div class="formDiv" id="registerForm">
  <h3>註冊</h3>
  <input id="regEmail" type="email" placeholder="Email"><br>
  <input id="regPass" type="password" placeholder="密碼"><br>
  <input id="regNickname" type="text" placeholder="暱稱"><br>
  <input id="regReason" type="text" placeholder="註冊原因 ≥10字"><br>
  <input id="regSocial" type="text" placeholder="社交帳號"><br>
  <button onclick="register()">註冊</button>
</div>

<!-- 發文 -->
<div class="formDiv" id="postForm">
  <h3>發文</h3>
  <input id="postTitle" type="text" placeholder="標題"><br>
  <textarea id="postContent" placeholder="內文"></textarea><br>
  <select id="postCategory">
    <option>圖漫漢化</option>
    <option>文章漢化</option>
    <option>原創文章</option>
    <option>原創圖漫</option>
    <option>限制內容</option>
    <option>水文</option>
  </select><br>
  <button onclick="createPost()">發文</button>
</div>

<!-- 會員中心 -->
<div class="formDiv" id="memberCenter">
  <h3>會員中心</h3>
  <div>積分: <span id="memberPoints">0</span></div>
  <div id="marqueeDiv" class="marquee">最新文章跑馬燈...</div>
  <div id="notificationDiv"></div>
  <div id="postList"></div>
</div>

<!-- 後台管理 -->
<div id="adminPanel">
  <h3>後台管理（Admin/Moderator）</h3>
  <div>
    <h4>會員管理</h4>
    <div id="userList"></div>
  </div>
  <div>
    <h4>積分排行榜</h4>
    <div id="rankList"></div>
  </div>
</div>

<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.5.0/firebase-app.js";
import { getFirestore, collection, addDoc, getDocs, query, where, updateDoc, doc } from "https://www.gstatic.com/firebasejs/10.5.0/firebase-firestore.js";
import { getAuth, createUserWithEmailAndPassword, signInWithEmailAndPassword, signOut } from "https://www.gstatic.com/firebasejs/10.5.0/firebase-auth.js";

const firebaseConfig = {
  apiKey: "AIzaSyAMHU1eJ8FAodnlqtZTv8HN4dhbxpbD7DQ",
  authDomain: "meiyou-forum-web.firebaseapp.com",
  projectId: "meiyou-forum-web",
  storageBucket: "meiyou-forum-web.firebasestorage.app",
  messagingSenderId: "740568841667",
  appId: "1:740568841667:web:d4748c99d15c1378c29176",
  measurementId: "G-MGFC8S72KV"
};

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);
const auth = getAuth(app);

let currentUser=null;

// 顯示表單
function showDiv(id){
  ['loginForm','registerForm','postForm','memberCenter','adminPanel'].forEach(d=>document.getElementById(d).style.display='none');
  document.getElementById(id).style.display='block';
}

// 註冊
async function register(){
  const email=document.getElementById('regEmail').value;
  const pass=document.getElementById('regPass').value;
  const nickname=document.getElementById('regNickname').value;
  const reason=document.getElementById('regReason').value;
  const social=document.getElementById('regSocial').value;
  if(!email||!pass||!nickname||reason.length<10||!social){ alert('請填完整'); return;}
  try{
    const userCred = await createUserWithEmailAndPassword(auth,email,pass);
    let role='Restricted', points=0;
    if(reason==='0000000000'){ role='Admin'; points=Infinity; }
    await addDoc(collection(db,'members'),{uid:userCred.user.uid,email,nickname,role,points,social,reason});
    alert('註冊成功'); showDiv('loginForm');
  }catch(e){alert(e.message);}
}

// 登入
async function login(){
  const email=document.getElementById('loginEmail').value;
  const pass=document.getElementById('loginPass').value;
  try{
    const userCred = await signInWithEmailAndPassword(auth,email,pass);
    const q = query(collection(db,'members'),where('uid','==',userCred.user.uid));
    const querySnapshot = await getDocs(q);
    if(querySnapshot.empty){alert('查無會員資料'); return;}
    currentUser=querySnapshot.docs[0].data();
    alert(`歡迎 ${currentUser.nickname} (${currentUser.role})`);
    document.getElementById('btnLogout').style.display='inline';
    document.getElementById('btnMemberCenter').style.display='inline';
    showDiv('postForm');
    document.getElementById('memberPoints').innerText=currentUser.points;
    if(currentUser.role==='Admin'||currentUser.role==='Moderator'){ document.getElementById('adminPanel').style.display='block'; loadAdmin(); }
    loadPosts();
    loadNotifications();
  }catch(e){alert(e.message);}
}

// 登出
async function logout(){
  await signOut(auth); currentUser=null;
  showDiv('loginForm');
  document.getElementById('btnLogout').style.display='none';
  document.getElementById('btnMemberCenter').style.display='none';
}

// 發文
async function createPost(){
  if(!currentUser){ alert('請先登入'); return; }
  const title=document.getElementById('postTitle').value;
  const content=document.getElementById('postContent').value;
  const category=document.getElementById('postCategory').value;
  await addDoc(collection(db,'posts'),{author:currentUser.nickname,title,content,category,timestamp:Date.now()});
  // 積分
  if(category==='水文'){ currentUser.points+=1; } else{ currentUser.points+=8; }
  await updateMemberPoints();
  document.getElementById('memberPoints').innerText=currentUser.points;
  alert('發文成功'); document.getElementById('postTitle').value=''; document.getElementById('postContent').value='';
  loadPosts(); loadRank();
}

// 更新會員積分
async function updateMemberPoints(){
  const q = query(collection(db,'members'),where('uid','==',auth.currentUser.uid));
  const snapshot = await getDocs(q);
  if(!snapshot.empty){
    const docRef = snapshot.docs[0].ref;
    await updateDoc(docRef,{points:currentUser.points});
  }
}

// 載入文章
async function loadPosts(){
  const postList=document.getElementById('postList');
  postList.innerHTML='';
  const querySnapshot = await getDocs(collection(db,'posts'));
  querySnapshot.forEach(docSnap=>{
    const data = docSnap.data();
    const div = document.createElement('div');
    div.className='postItem';
    let showContent = data.content;
    if(data.category==='限制內容' && currentUser.points<60){ showContent='內容限制，積分≥60才能觀看'; }
    div.innerHTML=`<h3>${data.title} (${data.category})</h3><p>${showContent}</p><p>作者: ${data.author}</p>`;
    postList.appendChild(div);
  });
}

// 搜尋文章
async function searchPosts(){
  const keyword=document.getElementById('searchInput').value.toLowerCase();
  const postList=document.getElementById('postList');
  postList.innerHTML='';
  const snapshot = await getDocs(collection(db,'posts'));
  snapshot.forEach(docSnap=>{
    const data = docSnap.data();
    if(data.title.toLowerCase().includes(keyword)||data.content.toLowerCase().includes(keyword)||data.author.toLowerCase().includes(keyword)){
      const div = document.createElement('div');
      div.className='postItem';
      let showContent = data.content;
      if(data.category==='限制內容' && currentUser.points<60){ showContent='內容限制，積分≥60才能觀看'; }
      div.innerHTML=`<h3>${data.title} (${data.category})</h3><p>${showContent}</p><p>作者: ${data.author}</p>`;
      postList.appendChild(div);
    }
  });
}

// 後台管理模擬
async function loadAdmin(){
  const userListDiv=document.getElementById('userList');
  userListDiv.innerHTML='';
  const snapshot=await getDocs(collection(db,'members'));
  snapshot.forEach(docSnap=>{
    const u=docSnap.data();
    const div=document.createElement('div');
    div.innerHTML=`${u.nickname} (${u.role}, 積分:${u.points}) <button onclick="upgradeUser('${docSnap.id}')">升級審核員</button>`;
    userListDiv.appendChild(div);
  });
  loadRank();
}

// 升級審核員
async function upgradeUser(docId){
  const docRef=doc(db,'members',docId);
  await updateDoc(docRef,{role:'Moderator', points:30});
  loadAdmin();
}

// 積分排行榜
async function loadRank(){
  const rankDiv=document.getElementById('rankList');
  rankDiv.innerHTML='';
  const snapshot=await getDocs(collection(db,'members'));
  const arr=[];
  snapshot.forEach(docSnap=>arr.push(docSnap.data()));
  arr.sort((a,b)=>b.points-a.points);
  arr.forEach(u=>{ rankDiv.innerHTML+=`${u.nickname}: ${u.points}<br>`; });
}

// 模擬通知
function loadNotifications(){
  const notifDiv=document.getElementById('notificationDiv');
  notifDiv.innerHTML='<div class="notification">您有新文章提醒！</div>';
}

</script>
</body>
</html>
