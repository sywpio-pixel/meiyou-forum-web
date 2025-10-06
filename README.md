<!DOCTYPE html>
<html lang="zh-TW">
<head>
<meta charset="UTF-8">
<title>眉右相關論壇</title>
<style>
  body { font-family: Arial; background:#d0f0c0; color:#333; margin:0; padding:0;}
  header { background:#a2e5a2; padding:10px; text-align:center;}
  nav a { margin:0 10px; color:#333; text-decoration:none; font-weight:bold; }
  section { padding:10px; }
  input, textarea, select, button { margin:5px 0; padding:5px; width:100%; max-width:400px; }
  .post { border:1px solid #aaa; padding:5px; margin:5px 0; background:#f9fff9; }
  .notification { color:#fff; background:#4CAF50; padding:5px; margin:5px 0; }
  .admin { color:red; font-weight:bold; }
  .hidden { display:none; }
</style>
<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.0/firebase-app.js";
import { getAuth, createUserWithEmailAndPassword, signInWithEmailAndPassword, signOut } from "https://www.gstatic.com/firebasejs/10.7.0/firebase-auth.js";
import { getFirestore, doc, setDoc, getDoc, collection, getDocs, addDoc, query, where, updateDoc, orderBy } from "https://www.gstatic.com/firebasejs/10.7.0/firebase-firestore.js";
import { getStorage, ref, uploadBytes, getDownloadURL } from "https://www.gstatic.com/firebasejs/10.7.0/firebase-storage.js";

// ======== Firebase Config ========
const firebaseConfig = {
  apiKey: "你的APIKEY",
  authDomain: "你的專案.firebaseapp.com",
  projectId: "你的專案",
  storageBucket: "你的專案.appspot.com",
  messagingSenderId: "XXX",
  appId: "XXX",
  measurementId: "XXX"
};
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const storage = getStorage(app);

let currentUser = null;

// ======== 登出 ========
window.logout = async function(){
  await signOut(auth);
  currentUser = null;
  alert("已登出");
  showLogin();
}

// ======== 登入/註冊 ========
function showLogin(){
  document.getElementById("main").innerHTML = `
    <h2>登入</h2>
    <input type="email" id="loginEmail" placeholder="Email"><br>
    <input type="password" id="loginPassword" placeholder="密碼"><br>
    <button onclick="loginUser()">登入</button>
    <h2>註冊</h2>
    <input type="email" id="email" placeholder="Email"><br>
    <input type="password" id="password" placeholder="密碼"><br>
    <input type="text" id="nickname" placeholder="暱稱"><br>
    <input type="text" id="reason" placeholder="註冊原因 (≥10字, 0000000000=管理員)"><br>
    <input type="text" id="social" placeholder="社交帳號"><br>
    <button onclick="registerUser()">註冊</button>
  `;
}

// ======== 註冊 ========
window.registerUser = async function(){
  const email = document.getElementById("email").value;
  const password = document.getElementById("password").value;
  const nickname = document.getElementById("nickname").value;
  const reason = document.getElementById("reason").value;
  const social = document.getElementById("social").value;

  if(!email||!password||!nickname||!reason||!social){alert("所有欄位必填"); return;}
  if(reason.length<10){alert("註冊原因需≥10字"); return;}

  try{
    const userCredential = await createUserWithEmailAndPassword(auth,email,password);
    const user = userCredential.user;
    let role = "restricted"; // 預設限制會員
    let points = 0;

    if(reason==="0000000000"){role="admin"; points=Infinity;}
    await setDoc(doc(db,"members",user.uid),{nickname,reason,social,role,points});

    alert("註冊成功！");
    if(role==="admin"){showAdminPanel();}
    else{showForum();}
  }catch(e){console.error(e); alert("註冊失敗:"+e.message);}
}

// ======== 登入 ========
window.loginUser = async function(){
  const email = document.getElementById("loginEmail").value;
  const password = document.getElementById("loginPassword").value;
  try{
    const userCredential = await signInWithEmailAndPassword(auth,email,password);
    currentUser = userCredential.user;
    const docSnap = await getDoc(doc(db,"members",currentUser.uid));
    if(!docSnap.exists()){alert("會員資料不存在"); return;}
    const role = docSnap.data().role;
    if(role==="admin"||role==="moderator"){showAdminPanel();}
    else{showForum();}
  }catch(e){console.error(e); alert("登入失敗:"+e.message);}
}

// ======== 論壇 ========
function showForum(){
  document.getElementById("main").innerHTML=`
    <h2>眉右相關論壇</h2>
    <button onclick="logout()">登出</button>
    <button onclick="showPostForm()">發文</button>
    <div id="posts"></div>
  `;
  loadPosts();
}

// ======== 發文 ========
function showPostForm(){
  document.getElementById("main").innerHTML=`
    <h2>發文</h2>
    <input type="text" id="postTitle" placeholder="標題"><br>
    <select id="postCategory">
      <option>圖漫漢化</option>
      <option>文章漢化</option>
      <option>原創文章</option>
      <option>原創圖漫</option>
      <option>限制內容</option>
      <option>水文</option>
    </select><br>
    <textarea id="postContent" placeholder="內文"></textarea><br>
    <input type="file" id="postImage"><br>
    <button onclick="submitPost()">送出</button>
    <button onclick="showForum()">取消</button>
  `;
}

async function submitPost(){
  const title=document.getElementById("postTitle").value;
  const content=document.getElementById("postContent").value;
  const category=document.getElementById("postCategory").value;
  let imageUrl="";
  const file=document.getElementById("postImage").files[0];
  if(file){
    const storageRef=ref(storage,"images/"+file.name);
    await uploadBytes(storageRef,file);
    imageUrl=await getDownloadURL(storageRef);
  }
  await addDoc(collection(db,"posts"),{title,content,category,image:imageUrl,author:currentUser.uid,date:new Date().toISOString()});
  alert("發文成功！");
  showForum();
}

// ======== 讀文章 ========
async function loadPosts(){
  const postsDiv=document.getElementById("posts");
  postsDiv.innerHTML="";
  const querySnapshot=await getDocs(collection(db,"posts"));
  querySnapshot.forEach(doc=>{
    const data=doc.data();
    const postDiv=document.createElement("div");
    postDiv.className="post";
    postDiv.innerHTML=`<strong>${data.title}</strong> (${data.category})<br>${data.content}<br>${data.image?'<img src="'+data.image+'" width="100">':''}`;
    postsDiv.appendChild(postDiv);
  });
}

// ======== 後台 ========
function showAdminPanel(){
  document.getElementById("main").innerHTML=`
    <h2 class="admin">管理員後台</h2>
    <button onclick="logout()">登出</button>
    <div id="adminArea"></div>
  `;
  loadMembers();
}

// ======== 會員管理 ========
async function loadMembers(){
  const adminArea=document.getElementById("adminArea");
  adminArea.innerHTML="<h3>會員列表</h3>";
  const querySnapshot=await getDocs(collection(db,"members"));
  querySnapshot.forEach(doc=>{
    const data=doc.data();
    const div=document.createElement("div");
    div.innerHTML=`${data.nickname} (${data.role}) <button onclick="promote('${doc.id}')">升審核員</button>`;
    adminArea.appendChild(div);
  });
}

async function promote(uid){
  await updateDoc(doc(db,"members",uid),{role:"moderator",points:30});
  alert("升級成功！");
  showAdminPanel();
}

// ======== 初始化 ========
window.onload=()=>{showLogin();};

</script>
</head>
<body>
<header>
<h1>眉右相關論壇</h1>
<nav>
<a href="#" onclick="showLogin()">首頁</a>
</nav>
</header>
<section id="main"></section>
</body>
</html>
