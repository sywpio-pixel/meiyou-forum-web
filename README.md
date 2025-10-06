<!DOCTYPE html>
<html lang="zh-TW">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>眉右相關論壇</title>
<style>
body { font-family: Arial; background-color:#e0f2e9; color:#333; margin:0; padding:0;}
.hidden { display:none; }
header { background:#b8e0d2; padding:10px; text-align:center; }
nav button { margin:0 5px; }
.forum-post { border:1px solid #cce6d5; padding:10px; margin:5px 0; border-radius:5px; background:#f4fff8; }
input, select, textarea, button { margin:5px 0; padding:5px; width:200px; }
textarea { width:400px; height:100px; }
</style>
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
  <input type="password" id="password" placeholder="密碼"><br
