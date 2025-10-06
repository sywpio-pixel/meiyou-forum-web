<!DOCTYPE html>
<html lang="zh-Hant">
<head>
<meta charset="utf-8" />
<title>眉右相關論壇</title>
<meta name="viewport" content="width=device-width,initial-scale=1" />
<link href="https://fonts.googleapis.com/css2?family=Nunito:wght@300;400;700&display=swap" rel="stylesheet">
<style>
  :root{
    --bg:#e6f6ee;
    --panel:#ffffff;
    --accent:#a7e3bf;
    --accent-2:#b7e4c7;
    --text:#1f3d34;
    --muted:#6b7f75;
  }
  *{box-sizing:border-box}
  body{font-family:'Nunito',sans-serif;background:var(--bg);color:var(--text);margin:0;padding:0;}
  header{background:linear-gradient(90deg,var(--accent-2),#ccecdb);padding:14px 18px;display:flex;align-items:center;justify-content:space-between;box-shadow:0 2px 8px rgba(0,0,0,0.06)}
  header h1{margin:0;font-size:1.3rem}
  nav button{background:transparent;border:1px solid rgba(0,0,0,0.05);padding:8px 12px;border-radius:8px;margin-left:8px;cursor:pointer}
  main{max-width:1100px;margin:18px auto;padding:0 12px;}
  .row{display:flex;gap:16px;}
  .col-left{width:260px;}
  .col-main{flex:1;}
  .card{background:var(--panel);padding:12px;border-radius:10px;box-shadow:0 2px 6px rgba(0,0,0,0.04);margin-bottom:14px}
  input,select,textarea,button{font-family:inherit}
  input,select,textarea{width:100%;padding:8px;border-radius:8px;border:1px solid #d9e7df;margin:6px 0}
  button.primary{background:var(--accent);border:none;padding:8px 12px;border-radius:8px;cursor:pointer}
  .post{padding:12px;margin-bottom:12px;border-radius:8px;background:#fff;border:1px solid #eef7ef}
  .muted{color:var(--muted);font-size:0.9rem}
  .marquee{background:linear-gradient(90deg,#dff6e9,#eafaf0);padding:8px;border-radius:8px;margin-bottom:14px;overflow:hidden;white-space:nowrap}
  .marquee span{display:inline-block;padding-left:100%;animation:marq 18s linear infinite}
  @keyframes marq{0%{transform:translateX(0)}100%{transform:translateX(-100%)}}
  ul{padding-left:18px}
  .small{font-size:0.9rem;color:var(--muted)}
  .tag{display:inline-block;padding:4px 8px;background:#f0fcf6;border-radius:12px;margin-right:6px;font-size:0.85rem}
  .img-preview{max-width:320px;border-radius:8px;margin-top:8px;border:1px solid #e6f6ee}
  table{width:100%;border-collapse:collapse}
  table th, table td{padding:8px;border:1px solid #eef7ef;text-align:left}
  .danger{background:#ffdede}
  footer{padding:20px;text-align:center;color:var(--muted)}
  .flex{display:flex;gap:8px;align-items:center}
  .badge{background:#fff;padding:4px 8px;border-radius:999px;border:1px solid #e6f6ee}
</style>
</head>
<body>

<header>
  <div style="display:flex;align-items:center;gap:12px">
    <h1>眉右相關論壇</h1>
    <div class="small muted">前端模擬</div>
  </div>
  <nav>
    <button onclick="showView('home')">首頁</button>
    <button onclick="showView('forum')">論壇</button>
    <button onclick="showView('postForm')">發文</button>
    <button onclick="showView('profile')">個人</button>
    <button onclick="showView('admin')">後台</button>
    <button onclick="showView('notifications')">通知 <span id="notifBadge" class="badge" style="margin-left:6px"></span></button>
  </nav>
</header>

<main>
  <div id="app"></div>
</main>

<footer>示範前端版 — 資料存在 localStorage（非正式後端）</footer>

<script>
// --------------- 持久化狀態及初始化 ---------------
const DB = {
  members: JSON.parse(localStorage.getItem('members')||'[]'),
  posts: JSON.parse(localStorage.getItem('posts')||'[]'),
  comments: JSON.parse(localStorage.getItem('comments')||'[]'), // array of arrays
  notifications: JSON.parse(localStorage.getItem('notifications')||'[]'),
  reports: JSON.parse(localStorage.getItem('reports')||'[]'),
  logs: JSON.parse(localStorage.getItem('logs')||'[]'),
  blacklist: JSON.parse(localStorage.getItem('blacklist')||'[]'),
  lastPostTime: JSON.parse(localStorage.getItem('lastPostTime')||'0'),
  currentUser: JSON.parse(localStorage.getItem('currentUser')||'null')
};

function saveDB(){ // persist
  localStorage.setItem('members', JSON.stringify(DB.members));
  localStorage.setItem('posts', JSON.stringify(DB.posts));
  localStorage.setItem('comments', JSON.stringify(DB.comments));
  localStorage.setItem('notifications', JSON.stringify(DB.notifications));
  localStorage.setItem('reports', JSON.stringify(DB.reports));
  localStorage.setItem('logs', JSON.stringify(DB.logs));
  localStorage.setItem('blacklist', JSON.stringify(DB.blacklist));
  localStorage.setItem('lastPostTime', JSON.stringify(DB.lastPostTime));
  localStorage.setItem('currentUser', JSON.stringify(DB.currentUser));
}

// helper: find member by email or nickname
function findMemberByEmail(email){ return DB.members.find(m=>m.email===email) }
function findMemberByNick(nick){ return DB.members.find(m=>m.nickname===nick) }

// ensure structure
if(!Array.isArray(DB.posts)) DB.posts=[];
if(!Array.isArray(DB.comments)) DB.comments=[];
if(!Array.isArray(DB.notifications)) DB.notifications=[];
if(!Array.isArray(DB.members)) DB.members=[];

// Utility logs
function pushLog(text){
  DB.logs.unshift({time:new Date().toLocaleString(), text});
  if(DB.logs.length>500) DB.logs.pop();
  saveDB();
}

// Notification helper
function pushNotification(to, content){
  DB.notifications.unshift({to, content, time:new Date().toLocaleString(), read:false});
  saveDB();
  renderNotifBadge();
}

// render notification badge
function renderNotifBadge(){
  if(!DB.currentUser){ document.getElementById('notifBadge').textContent=''; return; }
  const count = DB.notifications.filter(n=>n.to===DB.currentUser.nickname && !n.read).length;
  document.getElementById('notifBadge').textContent = count?count:'';
}
renderNotifBadge();

// --------------- Views (router-like) ---------------
const app = document.getElementById('app');

function showView(v){
  switch(v){
    case 'home': return renderHome();
    case 'forum': return renderForum();
    case 'postForm': return renderPostForm();
    case 'profile': return renderProfile();
    case 'admin': return renderAdmin();
    case 'notifications': return renderNotifications();
    default: return renderHome();
  }
}

/* ---------- HOME (含跑馬燈 & 搜尋 & 積分排行) ---------- */
function renderHome(){
  // marquee latest 5 titles
  const latest = DB.posts.slice(-5).map(p=>escapeHtml(p.title)).join('　✦　') || '目前尚無文章';
  app.innerHTML = `
    <div class="marquee card"><span>${latest}</span></div>
    <div class="row">
      <div class="col-left">
        <div class="card">
          <h3>快速功能</h3>
          <div><button class="primary" onclick="showView('forum')">進入論壇</button></div>
          <div><button onclick="showView('postForm')">發文</button></div>
          <div style="margin-top:8px" class="small">若要登入測試已存在帳號可用 demo@example.com / demo123</div>
        </div>
        <div class="card">
          <h4>板塊</h4>
          <ul>
            <li><a href="#" onclick="filterByCategory('圖漫漢化')">圖漫漢化</a></li>
            <li><a href="#" onclick="filterByCategory('文章漢化')">文章漢化</a></li>
            <li><a href="#" onclick="filterByCategory('原創文章')">原創文章</a></li>
            <li><a href="#" onclick="filterByCategory('原創圖漫')">原創圖漫</a></li>
            <li><a href="#" onclick="filterByCategory('限制內容')">限制內容</a></li>
            <li><a href="#" onclick="filterByCategory('水文')">水文</a></li>
          </ul>
        </div>
      </div>
      <div class="col-main">
        <div class="card">
          <h3>積分排行榜</h3>
          <ol id="rankList"></ol>
        </div>
        <div class="card">
          <h3>最新文章</h3>
          <div id="homePosts"></div>
        </div>
      </div>
    </div>
  `;
  renderRank();
  renderHomePosts();
}

/* ---------- FORUM (文章列表、搜尋) ---------- */
function renderForum(){
  if(!DB.currentUser){ alert('請先登入'); renderLoginBox(); return;}
  app.innerHTML = `
    <div class="card">
      <div style="display:flex;justify-content:space-between;align-items:center">
        <div><h3>論壇列表</h3><div class="small muted">角色：${DB.currentUser.role}　｜　積分：${DB.currentUser.points}</div></div>
        <div style="display:flex;gap:8px">
          <input id="globalSearch" placeholder="搜尋標題/作者/標籤" style="width:320px">
          <button onclick="doSearch()">搜尋</button>
        </div>
      </div>
    </div>
    <div id="forumList"></div>
  `;
  renderForumList(DB.posts);
  renderMarqueeTop();
}

function renderForumList(list){
  const cont = document.getElementById('forumList');
  if(!cont) return;
  cont.innerHTML = list.length? list.map((p,i)=>renderPostCard(p,i)).join('') : '<div class="card">目前尚無文章</div>';
}

function renderPostCard(p,i){
  const canView = !(p.category==='限制內容' && (DB.currentUser.points < 60) && DB.currentUser.role!=='管理員');
  const sticky = p.sticky?'<span class="tag">置頂</span>':'';
  const featured = p.featured?'<span class="tag">精華</span>':'';
  const imgHtml = p.image?`<img src="${p.image}" class="img-preview">`:'';
  return `<div class="post card">
    <div style="display:flex;justify-content:space-between">
      <div><strong>${escapeHtml(p.title)}</strong> ${sticky}${featured}</div>
      <div class="small muted">${escapeHtml(p.author)} ・ ${p.category}</div>
    </div>
    <div class="small muted" style="margin-top:6px">${escapeHtml(p.summary || p.content.slice(0,200))}</div>
    ${imgHtml}
    <div style="margin-top:8px">
      <button onclick="viewPost(${i})">檢視</button>
      <button onclick="toggleLike(${i})">👍 ${p.likes||0}</button>
      <button onclick="toggleFav(${i})">⭐ ${p.favs||0}</button>
      ${p.author===DB.currentUser.nickname || DB.currentUser.role==='管理員' ? `<button onclick="deletePost(${i})" style="background:#ffdede">刪除</button>` : ''}
    </div>
    <div class="small muted" style="margin-top:6px">瀏覽: ${p.views||0}</div>
  </div>`;
}

/* ---------- POST VIEW & CREATE ---------- */
function renderPostForm(){
  if(!DB.currentUser){ alert('請先登入'); renderLoginBox(); return; }
  app.innerHTML = `
    <div class="card">
      <h3>發表文章</h3>
      <input id="postTitle" placeholder="標題">
      <textarea id="postContent" placeholder="內文" rows="6"></textarea>
      <select id="postCategory">
        <option>圖漫漢化</option>
        <option>文章漢化</option>
        <option>原創文章</option>
        <option>原創圖漫</option>
        <option>限制內容</option>
        <option>水文</option>
      </select>
      <input id="postTags" placeholder="標籤，逗號分隔（例：18禁,CP名）">
      <input id="postImage" type="file" accept="image/*">
      <div style="display:flex;gap:8px;align-items:center">
        <label><input type="checkbox" id="needCaptcha"> 需要驗證碼發文</label>
        <button onclick="submitPost()">送出</button>
      </div>
      <div class="small muted">發文 +8（若為水文 +1），留言 +3，刪除自己文章 -6；限制內容需 ≥60 積分觀看</div>
    </div>
  `;
}

// Anti-spam + captcha + image handling
function submitPost(){
  if(!DB.currentUser){ alert('請先登入'); return; }
  const now = Date.now();
  if(now - DB.lastPostTime < 30000){ alert('發文太頻繁，請稍後再發（每30秒）'); return; }
  const title = document.getElementById('postTitle').value.trim();
  const content = document.getElementById('postContent').value.trim();
  const category = document.getElementById('postCategory').value;
  const tags = document.getElementById('postTags').value.trim();
  if(!title || !content){ alert('標題與內文不可為空'); return; }
  if(document.getElementById('needCaptcha').checked){
    const code = prompt('請輸入驗證碼（輸入 1234 測試）');
    if(code !== '1234'){ alert('驗證碼錯誤'); return; }
  }
  const file = document.getElementById('postImage').files[0];
  const pointsGain = (category==='水文'?1:8);
  if(file){
    const reader = new FileReader();
    reader.onload = e => {
      DB.posts.push({title,content,category,tags:tags.split(',').map(s=>s.trim()).filter(Boolean),author:DB.currentUser.nickname,image:e.target.result,views:0,likes:0,favs:0,sticky:false,featured:false,points:pointsGain, created: new Date().toLocaleString()});
      finalizePost(pointsGain);
    };
    reader.readAsDataURL(file);
  } else {
    DB.posts.push({title,content,category,tags:tags.split(',').map(s=>s.trim()).filter(Boolean),author:DB.currentUser.nickname,views:0,likes:0,favs:0,sticky:false,featured:false,points:pointsGain, created: new Date().toLocaleString()});
    finalizePost(pointsGain);
  }
}
function finalizePost(pointsGain){
  DB.lastPostTime = Date.now();
  DB.currentUser.points = (DB.currentUser.points||0) + pointsGain;
  // update member record
  const mIdx = DB.members.findIndex(m=>m.email===DB.currentUser.email);
  if(mIdx>=0) DB.members[mIdx].points = DB.currentUser.points;
  pushLog(`${DB.currentUser.nickname} 發文 "${DB.posts[DB.posts.length-1].title}" (+${pointsGain})`);
  saveDB();
  alert(`發文成功，積分 +${pointsGain}`);
  showView('forum');
}

/* ---------- VIEW single post ---------- */
function viewPost(i){
  const p = DB.posts[i];
  p.views = (p.views||0) + 1;
  saveDB();
  let canView = !(p.category==='限制內容' && DB.currentUser.points < 60 && DB.currentUser.role!=='管理員');
  app.innerHTML = `
    <div class="card">
      <h2>${escapeHtml(p.title)}</h2>
      <div class="small muted">作者：<a href="#" onclick="viewProfileByNick('${p.author}')">${escapeHtml(p.author)}</a>｜板塊：${p.category}｜發表：${p.created}</div>
      ${p.image?`<img src="${p.image}" class="img-preview">`:''}
      <div style="margin-top:10px">${canView?escapeHtml(p.content):'<i>限制內容，積分不足（需 ≥60）</i>'}</div>
      <div style="margin-top:10px">
        <button onclick="replyToPost(${i})">回覆</button>
        <button onclick="quotePost(${i})">引用</button>
        <button onclick="mentionUserPrompt(${i})">@標記</button>
        <button onclick="toggleLike(${i})">👍 ${p.likes||0}</button>
        <button onclick="toggleFav(${i})">⭐ ${p.favs||0}</button>
        ${DB.currentUser.role==='管理員'?`<button onclick="adminToggleSticky(${i})">置頂</button><button onclick="adminToggleFeatured(${i})">精華</button>`:''}
      </div>
      <hr>
      <div id="commentsArea">${renderComments(i)}</div>
    </div>
    <div style="margin-top:12px"><button onclick="showView('forum')">回列表</button></div>
  `;
}

/* ---------- COMMENTS (reply/quote/mention) ---------- */
function renderComments(postIndex){
  const list = DB.comments[postIndex] || [];
  const html = list.map((c,ci)=>`<div class="comment"><b>${escapeHtml(c.author)}</b>：${escapeHtml(c.content)} <span class="small muted">(${c.time})</span></div>`).join('');
  return `<h4>留言（${list.length}）</h4>${html}<div style="margin-top:8px"><button onclick="promptAddComment(${postIndex})">新增留言</button></div>`;
}
function promptAddComment(postIndex){
  const txt = prompt('留言內容（留空取消）');
  if(!txt) return;
  addComment(postIndex, txt);
}
function addComment(postIndex, text){
  if(!DB.currentUser){ alert('請先登入'); return; }
  if(!DB.comments[postIndex]) DB.comments[postIndex]=[];
  DB.comments[postIndex].push({author:DB.currentUser.nickname,content:text,time:new Date().toLocaleString()});
  DB.currentUser.points = (DB.currentUser.points||0)+3;
  const idx = DB.members.findIndex(m=>m.email===DB.currentUser.email);
  if(idx>=0) DB.members[idx].points = DB.currentUser.points;
  pushLog(`${DB.currentUser.nickname} 在文章[${DB.posts[postIndex].title}] 留言 (+3)`);
  // notify author
  pushNotification(DB.posts[postIndex].author, `${DB.currentUser.nickname} 回覆了你的文章 "${DB.posts[postIndex].title}"`);
  saveDB();
  viewPost(postIndex);
}
function replyToPost(i){ promptAddComment(i) }
function quotePost(i){
  const quote = `引用 ${DB.posts[i].author}：\n> ${DB.posts[i].content.slice(0,120)}\n\n`; 
  const txt = prompt('編輯引用後的留言', quote);
  if(txt) addComment(i, txt);
}
function mentionUserPrompt(i){
  const who = prompt('輸入要標記的使用者暱稱（nickname）');
  if(!who) return;
  pushNotification(who, `${DB.currentUser.nickname} 在文章 "${DB.posts[i].title}" 中標記了你`);
  alert('已標記並通知該用戶（模擬）');
}

/* ---------- LIKES / FAVS ---------- */
function toggleLike(i){
  DB.posts[i].likes = (DB.posts[i].likes||0) + 1;
  pushLog(`${DB.currentUser.nickname} 按讚 "${DB.posts[i].title}"`);
  saveDB();
  renderForumList(DB.posts);
}
function toggleFav(i){
  DB.posts[i].favs = (DB.posts[i].favs||0) + 1;
  pushNotification(DB.posts[i].author, `${DB.currentUser.nickname} 收藏了你的文章 "${DB.posts[i].title}"`);
  saveDB();
  renderForumList(DB.posts);
}

/* ---------- DELETE ---------- */
function deletePost(i){
  if(!DB.currentUser) return;
  if(DB.posts[i].author !== DB.currentUser.nickname && DB.currentUser.role !== '管理員'){ alert('無權刪除'); return; }
  if(DB.posts[i].author === DB.currentUser.nickname){
    DB.currentUser.points = Math.max(0, (DB.currentUser.points||0)-6);
    const mi = DB.members.findIndex(m=>m.email===DB.currentUser.email);
    if(mi>=0) DB.members[mi].points = DB.currentUser.points;
  }
  pushLog(`${DB.currentUser.nickname} 刪除文章 "${DB.posts[i].title}"`);
  DB.posts.splice(i,1);
  DB.comments.splice(i,1);
  saveDB();
  renderForumList(DB.posts);
}

/* ---------- SEARCH / FILTER ---------- */
function doSearch(){
  const key = document.getElementById('globalSearch')?.value?.toLowerCase() || document.getElementById('searchInput')?.value?.toLowerCase() || '';
  const filtered = DB.posts.filter(p => (p.title||'').toLowerCase().includes(key) || (p.author||'').toLowerCase().includes(key) || (p.tags||[]).join(',').toLowerCase().includes(key) || (p.content||'').toLowerCase().includes(key));
  renderForumList(filtered);
}
function filterByCategory(cat){ showView('forum'); setTimeout(()=>renderForumList(DB.posts.filter(p=>p.category===cat)), 100) }

/* ---------- PROFILE ---------- */
function renderProfile(){
  if(!DB.currentUser){ alert('請先登入'); renderLoginBox(); return; }
  app.innerHTML = `
    <div class="card">
      <h3>個人主頁 — ${DB.currentUser.nickname}</h3>
      <div>角色：${DB.currentUser.role}　｜　積分：${DB.currentUser.points}</div>
      <div style="margin-top:8px"><textarea id="bio" rows="4" placeholder="個人簡介">${DB.currentUser.bio||''}</textarea><button onclick="saveBio()">儲存簡介</button></div>
    </div>
    <div class="card">
      <h4>我的文章</h4>
      ${DB.posts.filter(p=>p.author===DB.currentUser.nickname).map((p,i)=>`<div class="small card"><b>${escapeHtml(p.title)}</b> <div class="small muted">${p.category}</div></div>`).join('') || '<div class="small muted">尚無</div>'}
    </div>
    <div class="card">
      <h4>收藏清單（模擬）</h4>
      <div class="small muted">此處可顯示已收藏文章並支援通知</div>
    </div>
  `;
}
function saveBio(){
  const bio = document.getElementById('bio').value.trim();
  DB.currentUser.bio = bio;
  const idx = DB.members.findIndex(m=>m.email===DB.currentUser.email);
  if(idx>=0) DB.members[idx].bio = bio;
  pushLog(`${DB.currentUser.nickname} 更新個人簡介`);
  saveDB();
  alert('已儲存');
}
function viewProfileByNick(nick){
  const user = DB.members.find(m=>m.nickname===nick);
  if(!user){ alert('找不到用戶'); return; }
  app.innerHTML = `
    <div class="card"><h3>${user.nickname}</h3><div>角色：${user.role}｜積分：${user.points}</div><div class="small">簡介：${escapeHtml(user.bio||'尚未填寫')}</div></div>
    <div><button onclick="showView('forum')">回論壇</button></div>
  `;
}

/* ---------- ADMIN (會員審核、升級、檢舉、黑名單、統計、日誌) ---------- */
function renderAdminPanel(){
  if(!DB.currentUser || DB.currentUser.role!=='管理員'){ alert('您無權限進入後台'); showView('home'); return; }
  app.innerHTML = `
    <div class="card"><h3>後台管理（管理員）</h3></div>
    <div class="row">
      <div class="col-left">
        <div class="card"><h4>待審核會員</h4><div id="pendingUsers"></div></div>
        <div class="card"><h4>黑名單</h4><div id="blacklist"></div></div>
      </div>
      <div class="col-main">
        <div class="card"><h4>文章管理</h4><div id="adminPosts"></div></div>
        <div class="card"><h4>檢舉列表</h4><div id="reportList"></div></div>
        <div class="card"><h4>系統統計 & 日誌</h4><div id="stats"></div><div style="margin-top:8px"><button onclick="viewLogs()">查看日誌</button></div></div>
      </div>
    </div>
  `;
  renderPending();
  renderBlacklistAdmin();
  renderAdminPosts();
  renderReports();
  renderStats();
}

function renderAdmin(){ renderAdminPanel(); }
function renderPending(){
  const pending = DB.members.filter(m=>m.role==='限制會員');
  document.getElementById('pendingUsers').innerHTML = pending.length? pending.map((u,idx)=>`<div style="margin-bottom:8px">${u.nickname} (${u.email}) 
    <div class="flex" style="margin-top:6px">
      <button onclick="approveMember('${u.email}')">批准（成為會員+5）</button>
      <button onclick="promoteToModerator('${u.email}')">升為審核員(+30)</button>
      <button onclick="rejectMember('${u.email}')" style="background:#ffdede">拒絕</button>
    </div></div>`).join('') : '<div class="small muted">無待審核會員</div>';
}

function approveMember(email){
  const idx = DB.members.findIndex(m=>m.email===email);
  if(idx<0) return alert('找不到使用者');
  DB.members[idx].role = '會員';
  DB.members[idx].points = 5;
  pushLog(`管理員 ${DB.currentUser.nickname} 批准 ${DB.members[idx].nickname} 為 會員 (+5)`);
  pushNotification(DB.members[idx].nickname, '您的帳號已由管理員批准為 一般會員（+5 積分）');
  saveDB();
  renderPending();
  renderAdminPanel();
}

function promoteToModerator(email){
  const idx = DB.members.findIndex(m=>m.email===email);
  if(idx<0) return alert('找不到使用者');
  DB.members[idx].role = '審核員';
  DB.members[idx].points = 30;
  pushLog(`管理員 ${DB.currentUser.nickname} 升級 ${DB.members[idx].nickname} 為 審核員 (+30)`);
  pushNotification(DB.members[idx].nickname, '您已被管理員升級為 審核員（+30 積分）');
  saveDB();
  renderPending();
  renderAdminPanel();
}

function rejectMember(email){
  const idx = DB.members.findIndex(m=>m.email===email);
  if(idx<0) return alert('找不到使用者');
  const name = DB.members[idx].nickname;
  DB.members.splice(idx,1);
  pushLog(`${DB.currentUser.nickname} 拒絕並刪除註冊 ${name}`);
  saveDB();
  renderPending();
}

function renderBlacklistAdmin(){
  document.getElementById('blacklist').innerHTML = DB.blacklist.length? DB.blacklist.map(b=>`<div>${b} <button onclick="unblock('${b}')">解除封鎖</button></div>`).join('') : '<div class="small muted">無黑名單</div>';
}
function blockUser(nick){
  if(!nick) return;
  if(!DB.blacklist.includes(nick)) DB.blacklist.push(nick);
  // mark member role to blocked
  const m = DB.members.find(x=>x.nickname===nick);
  if(m) m.role = '封鎖';
  pushLog(`${DB.currentUser.nickname} 封鎖 ${nick}`);
  saveDB();
  renderBlacklistAdmin();
}
function unblock(nick){
  DB.blacklist = DB.blacklist.filter(x=>x!==nick);
  const m = DB.members.find(x=>x.nickname===nick);
  if(m && m.role==='封鎖') m.role='限制會員';
  pushLog(`${DB.currentUser.nickname} 解除封鎖 ${nick}`);
  saveDB();
  renderBlacklistAdmin();
}

function renderAdminPosts(){
  const html = DB.posts.map((p,i)=>`<div style="margin-bottom:8px"><b>${escapeHtml(p.title)}</b> 作者:${p.author} | 板塊:${p.category}
    <div class="flex" style="margin-top:6px">
      <button onclick="adminDeletePost(${i})" class="danger">刪除</button>
      <button onclick="adminToggleSticky(${i})">${p.sticky?'取消置頂':'置頂'}</button>
      <button onclick="adminToggleFeatured(${i})">${p.featured?'取消精華':'設為精華'}</button>
    </div></div>`).join('') || '<div class="small muted">無文章</div>';
  document.getElementById('adminPosts').innerHTML = html;
}

function adminDeletePost(i){ DB.posts.splice(i,1); DB.comments.splice(i,1); pushLog(`${DB.currentUser.nickname} 管理員刪除文章`); saveDB(); renderAdminPosts(); }
function adminToggleSticky(i){ DB.posts[i].sticky = !DB.posts[i].sticky; pushLog(`${DB.currentUser.nickname} 置頂切換 ${DB.posts[i].title}`); saveDB(); renderAdminPosts(); }
function adminToggleFeatured(i){ DB.posts[i].featured = !DB.posts[i].featured; pushLog(`${DB.currentUser.nickname} 精華切換 ${DB.posts[i].title}`); saveDB(); renderAdminPosts(); }

function renderReports(){
  const html = DB.reports.length? DB.reports.map((r,idx)=>`<div style="margin-bottom:8px">文章:${escapeHtml(r.title)} (index:${r.postIndex}) 被 ${r.reporter} 檢舉
    <div class="flex" style="margin-top:6px"><button onclick="adminReviewReport(${idx}, true)">移除文章</button><button onclick="adminReviewReport(${idx}, false)">無異常</button></div></div>`).join('') : '<div class="small muted">無檢舉</div>';
  document.getElementById('reportList').innerHTML = html;
}
function adminReviewReport(idx, remove){
  const rep = DB.reports[idx];
  if(!rep) return;
  if(remove){
    DB.posts.splice(rep.postIndex,1);
    DB.comments.splice(rep.postIndex,1);
    pushLog(`${DB.currentUser.nickname} 刪除被檢舉文章: ${rep.title}`);
  } else {
    pushLog(`${DB.currentUser.nickname} 標記檢舉為無異常: ${rep.title}`);
  }
  DB.reports.splice(idx,1);
  saveDB();
  renderReports();
  renderAdminPosts();
}
function renderStats(){
  const active = DB.members.filter(m=>m.role!=='封鎖').length;
  const views = DB.posts.reduce((s,p)=>s+(p.views||0),0);
  const totalPosts = DB.posts.length;
  document.getElementById('stats').innerHTML=`活躍會員：${active}<br>總瀏覽量：${views}<br>發文數：${totalPosts}`;
}
function viewLogs(){ alert('日誌：\n' + DB.logs.slice(0,50).map(l=>`${l.time} - ${l.text}`).join('\n')) }

/* ---------- REPORT (user side) ---------- */
function reportPost(postIndex){
  const title = DB.posts[postIndex].title;
  const reporter = DB.currentUser?DB.currentUser.nickname:'匿名';
  DB.reports.push({postIndex, title, reporter, time:new Date().toLocaleString()});
  pushLog(`${reporter} 檢舉文章：${title}`);
  // notify admin(s)
  DB.members.filter(m=>m.role==='管理員').forEach(a=>pushNotification(a.nickname, `有檢舉待審核：${title}`));
  saveDB();
  alert('已檢舉，管理員會審核（模擬）');
}

/* ---------- UTILS & INIT ---------- */
function renderRank(){
  const list = DB.members.slice().sort((a,b)=> (b.points||0) - (a.points||0));
  document.getElementById('rankList').innerHTML = list.slice(0,10).map(m => `<li>${escapeHtml(m.nickname)} (${m.points||0})</li>`).join('');
}
function renderHomePosts(){
  const el = document.getElementById('homePosts');
  if(!el) return;
  const last5 = DB.posts.slice(-10).reverse();
  el.innerHTML = last5.map((p,i)=>`<div class="card"><b>${escapeHtml(p.title)}</b><div class="small muted">作者：${p.author}｜${p.category}</div></div>`).join('') || '<div class="small muted">無</div>';
}
function renderMarqueeTop(){ const m = DB.posts.slice(-5).map(p=>p.title).join('　✦　'); document.querySelector('.marquee span').textContent = m || '目前尚無文章'; }

function renderNotifBadge(){ renderNotifBadge; renderNotifBadgeInner(); }
function renderNotifBadgeInner(){ /* kept for compatibility */ }
renderNotifBadge();

// small helpers
function escapeHtml(s){ if(!s) return ''; return s.replace(/[&<>"']/g, (c)=>({ '&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;' })[c]); }

// quick login box for tests if not logged
function renderLoginBox(){ app.innerHTML = `<div class="card"><h3>請登入或註冊</h3><div class="flex"><input id="quickEmail" placeholder="Email"><input id="quickPwd" placeholder="密碼"><button onclick="quickLogin()">快速登入</button></div></div>`; }
function quickLogin(){
  const e=document.getElementById('quickEmail').value.trim();
  const p=document.getElementById('quickPwd').value.trim();
  const u = DB.members.find(m=>m.email===e && m.password===p);
  if(u){ DB.currentUser = u; saveDB(); alert('登入成功'); showView('forum'); } else alert('找不到帳號');
}

// initial demo account (if empty)
if(DB.members.length===0){
  DB.members.push({nickname:'admin',email:'admin@local',password:'admin',role:'管理員',points:Infinity, bio:'我是系統管理員'});
  DB.members.push({nickname:'demo',email:'demo@example.com',password:'demo123',role:'會員',points:5, bio:'示範帳號'});
  saveDB();
  pushLog('系統初始化 demo 帳號');
}

// initial render
showView('home');
renderNotifBadge();

</script>
</body>
</html>
