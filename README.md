# meiyou-forum-web
<!DOCTYPE html>
<html lang="zh-TW">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>眉右相關論壇</title>
<link rel="stylesheet" href="css/style.css">
<script type="module" src="js/main.js" defer></script>
</head>
<body>
<header>
  <h1>眉右相關論壇</h1>
  <nav>
    <button onclick="showSection('home')">首頁</button>
    <button onclick="showSection('forum')">論壇</button>
    <button onclick="showSection('profile')">會員中心</button>
    <button onclick="showSection('admin')">後台管理</button>
    <button onclick="logout()">登出</button>
  </nav>
</header>

<div id="login-section">
  <h2>登入 / 註冊</h2>
  <input type="email" id="email" placeholder="Email"><br>
  <input type="password" id="password" placeholder="密碼"><br>
  <input type="text" id="regReason" placeholder="註冊原因 (≥10字)"><br>
  <input type="text" id="social" placeholder="社交帳號"><br>
  <button onclick="login()">登入</button>
  <button onclick="register()">註冊</button>
</div>

<div id="home" class="hidden">
  <h2>首頁</h2>
  <div id="leaderboard"></div>
</div>

<div id="forum" class="hidden">
  <h2>論壇</h2>
  <select id="category">
    <option value="圖漫漢化">圖漫漢化</option>
    <option value="文章漢化">文章漢化</option>
    <option value="原創文章">原創文章</option>
    <option value="原創圖漫">原創圖漫</option>
    <option value="限制內容">限制內容</option>
    <option value="水文">水文</option>
  </select>
  <textarea id="postContent" placeholder="輸入文章內容"></textarea><br>
  <button onclick="createPost()">發文</button>
  <div id="posts"></div>
</div>

<div id="profile" class="hidden">
  <h2>會員中心</h2>
  <p>暱稱：<span id="profileName"></span></p>
  <p>積分：<span id="profilePoints"></span></p>
</div>

<div id="admin" class="hidden">
  <h2>後台管理</h2>
  <p>管理員/審核員專用</p>
  <div id="adminData"></div>
</div>
</body>
</html>
body { font-family: Arial; background-color:#e0f2e9; color:#333; margin:0; }
.hidden { display:none; }
header { background:#b8e0d2; padding:10px; text-align:center; }
nav button { margin:0 5px; }
.forum-post { border:1px solid #cce6d5; padding:10px; margin:5px 0; border-radius:5px; background:#f4fff8; }
import { initializeApp } from "https://www.gstatic.com/firebasejs/9.23.0/firebase-app.js";
import { getAnalytics } from "https://www.gstatic.com/firebasejs/9.23.0/firebase-analytics.js";
import { getAuth, createUserWithEmailAndPassword, signInWithEmailAndPassword, signOut, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/9.23.0/firebase-auth.js";
import { getFirestore, collection, addDoc, getDocs, doc, updateDoc, query, orderBy, limit } from "https://www.gstatic.com/firebasejs/9.23.0/firebase-firestore.js";

const firebaseConfig = {
  apiKey: "你的apikey",
  authDomain: "你的authDomain",
  projectId: "你的projectId",
  storageBucket: "你的storageBucket",
  messagingSenderId: "你的messagingSenderId",
  appId: "你的appId",
  measurementId: "你的measurementId"
};

const app = initializeApp(firebaseConfig);
const analytics = getAnalytics(app);
const auth = getAuth();
const db = getFirestore();

let currentUser = null;
let currentUserData = null;

function showSection(section) {
  ["login-section", "home", "forum", "profile", "admin"].forEach(id => {
    document.getElementById(id).classList.add("hidden");
  });
  document.getElementById(section).classList.remove("hidden");
}

function logout() { signOut(auth).then(()=>location.reload()); }

async function register() {
  const email = document.getElementById("email").value;
  const password = document.getElementById("password").value;
  const reason = document.getElementById("regReason").value;
  const social = document.getElementById("social").value;
  if (reason.length<10) { alert("註冊原因≥10字"); return; }
  if (!social) { alert("請填社交帳號"); return; }
  try {
    const userCredential = await createUserWithEmailAndPassword(auth,email,password);
    const user = userCredential.user;
    let role="Restricted", points=0;
    if(reason==="0000000000"){ role="Admin"; points=Infinity; }
    await addDoc(collection(db,"users"),{
      uid:user.uid,email,role,points,nickname:"",avatar:"",social,createdAt:new Date()
    });
    alert("註冊成功");
  } catch(err){ alert(err.message);}
}

async function login(){
  const email=document.getElementById("email").value;
  const password=document.getElementById("password").value;
  try{
    const userCredential=await signInWithEmailAndPassword(auth,email,password);
    currentUser=userCredential.user;
    await loadCurrentUser();
  }catch(err){ alert(err.message);}
}

async function loadCurrentUser(){
  const usersSnap=await getDocs(collection(db,"users"));
  usersSnap.forEach(docSnap=>{
    if(docSnap.data().uid===currentUser.uid){
      currentUserData={...docSnap.data(),docId:docSnap.id};
    }
  });
  if(!currentUserData){ alert("找不到使用者資料"); return; }
  if(currentUserData.role==="Admin"||currentUserData.role==="Moderator"){ showSection("admin"); }
  else { showSection("home"); }
  updateProfileDisplay();
  loadPosts();
  loadLeaderboard();
}

function updateProfileDisplay(){
  if(!currentUserData) return;
  document.getElementById("profileName").innerText=currentUserData.nickname||currentUserData.email;
  document.getElementById("profilePoints").innerText=currentUserData.points;
}

async function createPost(){
  if(!currentUserData){ alert("請先登入"); return; }
  const category=document.getElementById("category").value;
  const content=document.getElementById("postContent").value;
  if(!content){ alert("請輸入內容"); return; }
  try{
    await addDoc(collection(db,"posts"),{
      authorUid:currentUserData.uid,
      authorName:currentUserData.nickname||currentUserData.email,
      category,content,createdAt:new Date()
    });
    let addPoints=category==="水文"?1:8;
    const userRef=doc(db,"users",currentUserData.docId);
    await updateDoc(userRef,{points:currentUserData.points+addPoints});
    currentUserData.points+=addPoints;
    document.getElementById("postContent").value="";
    loadPosts();
    updateProfileDisplay();
  }catch(err){ alert(err.message);}
}

async function loadPosts(){
  const postsDiv=document.getElementById("posts");
  postsDiv.innerHTML="";
  const postsSnap=await getDocs(query(collection(db,"posts"),orderBy("createdAt","desc")));
  postsSnap.forEach(docSnap=>{
    const post=docSnap.data();
    const postDiv=document.createElement("div");
    postDiv.classList.add("forum-post");
    postDiv.innerHTML=`<strong>${post.authorName}</strong> [${post.category}]<br>${post.content}`;
    postsDiv.appendChild(postDiv);
  });
}

async function loadLeaderboard(){
  const lbDiv=document.getElementById("leaderboard");
  lbDiv.innerHTML="";
  const usersSnap=await getDocs(query(collection(db,"users"),orderBy("points","desc"),limit(10)));
  usersSnap.forEach(docSnap=>{
    const user=docSnap.data();
    const div=document.createElement("div");
    div.innerText=`${user.nickname||user.email} - ${user.points} 分`;
    lbDiv.appendChild(div);
  });
}

onAuthStateChanged(auth,user=>{
  if(user){ currentUser=user; loadCurrentUser(); }
  else{ showSection("login-section"); }
});

window.showSection=showSection;
window.logout=logout;
window.register=register;
window.login=login;
window.createPost=createPost;
