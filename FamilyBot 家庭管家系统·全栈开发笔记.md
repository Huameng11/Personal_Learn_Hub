# 🏠 FamilyBot 家庭管家系统·全栈开发笔记

## 一、 核心需求与系统设计

### 1.1 项目愿景

打造一个运行在树莓派上的 24 小时常驻家庭服务器，具备“全能 AI 管家”的交互能力，同时融合**智能药箱**、**家庭物资仓库**、**家庭备忘录**三大本地持久化模块。

### 1.2 核心技术栈

- **后端**：Python + FastAPI (高性能异步 Web 框架) + SQLite3 (轻量级嵌入式数据库) + OpenAI SDK (对接大模型 API)
- **前端**：Vue 3 (基于 CDN 的渐进式前端框架) + HTML5 PWA (渐进式 Web 应用，支持离线秒开与原生安装**（离线秒开调试失败，废用**）)
- **环境隔离**：`python-dotenv` 环境变量管理（隔离 API Key，防误传 GitHub）

### 1.3 核心架构原理

系统采用“主管家大脑 + 模块化数据中台”的解耦设计。AI 聊天接口通过异步并发读取本地 SQLite 的所有物资表，将其转化为结构化文本拼装进 `System Prompt`，从而使通用大模型拥有精确检索家庭内部隐私数据的能力（即轻量级 RAG 架构）。**（废用后续功能）** *同时，前端通过 **Service Worker 拦截机制** 与 **LocalStorage 乐观更新队列**，实现了外网 4G 状态下依然能够“秒开查看数据、记录离线操作、回家自动WiFi同步”的本地优先体验。*

### 1.4 工程目录结构

```ABAP
familyBot/
│  .env                   # 敏感配置文件（存放你的 API Key，切勿泄露）
│  config.py              # 全局配置读取模块（统一用 os.getenv 读取环境变量）
│  database.py            # SQLite 数据库引擎（建表、提供基础的 SQL 读取函数）
│  main.py                # 项目唯一的主控入口（负责注册路由、映射静态资源）
│  medicine.db            # 自动生成的本地 SQLite 数据库文件（在 Windows/树莓派本地）
│
├─routers/                # 后端业务路由文件夹（实现接口的物理隔离与解耦）
│      chat.py            # AI 大脑核心路由（读取全库数据，拼装 System Prompt 并调用 LLM）
│      medicine.py        # 智能药箱路由（处理药品的增、删、改、查 API）
│      memo.py            # 家庭备忘录路由（处理备忘事件的增、删、改、查 API）
│      storage.py         # 家庭仓库路由（处理日常物资的增、删、改、查 API）
│
└─static/                 # 前端静态资源文件夹（实现前端 PWA 与离线优先机制）
        index.html        # 单页面 Vue3 交互界面（包含竖排防御布局与离线乐观队列逻辑）
        manifest.json     # PWA 配置文件（定义 App 名称、主题色以及图标，允许添加至手机主屏幕）
        sw.js             # Service Worker 脚本（拦截网络请求，实现无网 / 外网 4G 状态下秒开网页）```
```



## 二、 后端模块化架构实现

### 2.1 配置文件 `config.py`

负责安全加载 `.env` 环境变量，避免敏感的密钥硬编码在代码中。

```Python
import os
from dotenv import load_dotenv

# 加载项目根目录下的 .env 文件
load_dotenv()

API_KEY = os.getenv("FAMILYBOT_API_KEY")
BASE_URL = os.getenv("FAMILYBOT_BASE_URL", "https://open.bigmodel.cn/api/paas/v4/")
MODEL_NAME = os.getenv("FAMILYBOT_MODEL_NAME", "glm-4-flash")
DB_PATH = os.getenv("FAMILYBOT_DB_PATH", "medicine.db")
```

### 2.2 数据库引擎 `database.py`

统一管理 SQLite 数据库的表结构初始化及跨模块数据读取。

```Python
import sqlite3
from config import DB_PATH

def init_db():
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    
    # 1. 智能药箱表
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS medicines (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL, count TEXT, location TEXT,
            expire_date TEXT, usage TEXT DEFAULT '', eff TEXT DEFAULT ''
        )
    ''')
    
    # 2. 家庭物资仓库表
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS storage (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            item_name TEXT NOT NULL, category TEXT DEFAULT '',
            quantity TEXT DEFAULT '', location TEXT DEFAULT '', remark TEXT DEFAULT ''
        )
    ''')

    # 3. 家庭日常备忘录表
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS memos (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL, content TEXT DEFAULT '',
            created_at TEXT DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    conn.commit()
    conn.close()

def get_db_medicines():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM medicines ORDER BY id ASC")
    rows = cursor.fetchall()
    conn.close()
    return rows

def get_db_storage():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM storage ORDER BY id ASC")
    rows = cursor.fetchall()
    conn.close()
    return rows

def get_db_memos():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM memos ORDER BY id DESC")
    rows = cursor.fetchall()
    conn.close()
    return rows
```

### 2.3 业务路由层（使用 `APIRouter` 进行功能解耦）

#### ① 智能药箱模块 `routers/medicine.py`

```Python
from fastapi import APIRouter, HTTPException
import sqlite3
from config import DB_PATH
from database import get_db_medicines

router = APIRouter(tags=["Medicine"])

@router.get("/api/medicines")
async def get_all_medicines():
    try:
        return {"status": "success", "data": [dict(row) for row in get_db_medicines()]}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/api/add_medicine")
async def add_medicine(name: str, count: str = "", location: str = "", expire_date: str = "", usage: str = ""):
    try:
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("INSERT INTO medicines (name, count, location, expire_date, usage) VALUES (?, ?, ?, ?, ?)", 
                       (name, count, location, expire_date, usage))
        conn.commit()
        conn.close()
        return {"status": "success", "message": "药品录入成功"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/api/update_medicine")
async def update_medicine(id: int, name: str, count: str = "", location: str = "", expire_date: str = "", usage: str = ""):
    try:
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("UPDATE medicines SET name=?, count=?, location=?, expire_date=?, usage=? WHERE id=?", 
                       (name, count, location, expire_date, usage, id))
        conn.commit()
        conn.close()
        return {"status": "success", "message": "更新成功"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/api/delete_medicine")
async def delete_medicine(id: int):
    try:
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("DELETE FROM medicines WHERE id=?", (id,))
        conn.commit()
        conn.close()
        return {"status": "success", "message": "删除成功"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

#### ② 家庭物资仓库模块 `routers/storage.py`

```Python
from fastapi import APIRouter, HTTPException
import sqlite3
from config import DB_PATH
from database import get_db_storage

router = APIRouter(tags=["Storage"])

@router.get("/api/storage")
async def get_all_storage():
    try:
        return {"status": "success", "data": [dict(row) for row in get_db_storage()]}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/api/add_storage")
async def add_storage(item_name: str, category: str = "", quantity: str = "", location: str = "", remark: str = ""):
    try:
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("INSERT INTO storage (item_name, category, quantity, location, remark) VALUES (?, ?, ?, ?, ?)", 
                       (item_name, category, quantity, location, remark))
        conn.commit()
        conn.close()
        return {"status": "success"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/api/update_storage")
async def update_storage(id: int, item_name: str, category: str = "", quantity: str = "", location: str = "", remark: str = ""):
    try:
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("UPDATE storage SET item_name=?, category=?, quantity=?, location=?, remark=? WHERE id=?", 
                       (item_name, category, quantity, location, remark, id))
        conn.commit()
        conn.close()
        return {"status": "success"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/api/delete_storage")
async def delete_storage(id: int):
    try:
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("DELETE FROM storage WHERE id=?", (id,))
        conn.commit()
        conn.close()
        return {"status": "success"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

#### ③ 家庭备忘录模块 `routers/memo.py`

```Python
from fastapi import APIRouter, HTTPException
import sqlite3
from config import DB_PATH
from database import get_db_memos

router = APIRouter(tags=["Memo"])

@router.get("/api/memos")
async def get_all_memos():
    try:
        return {"status": "success", "data": [dict(row) for row in get_db_memos()]}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/api/add_memo")
async def add_memo(title: str, content: str = ""):
    try:
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("INSERT INTO memos (title, content) VALUES (?, ?)", (title, content))
        conn.commit()
        conn.close()
        return {"status": "success"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/api/update_memo")
async def update_memo(id: int, title: str, content: str = ""):
    try:
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("UPDATE memos SET title=?, content=? WHERE id=?", (title, content, id))
        conn.commit()
        conn.close()
        return {"status": "success"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@router.post("/api/delete_memo")
async def delete_memo(id: int):
    try:
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute("DELETE FROM memos WHERE id=?", (id,))
        conn.commit()
        conn.close()
        return {"status": "success"}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

#### ④ AI 大脑核心对话模块 `routers/chat.py`

```Python
from fastapi import APIRouter
from openai import AsyncOpenAI
from config import API_KEY, BASE_URL, MODEL_NAME
from database import get_db_medicines, get_db_storage, get_db_memos

router = APIRouter(tags=["AI Chat"])
client = AsyncOpenAI(api_key=API_KEY, base_url=BASE_URL, timeout=20.0)

@router.get("/api/chat")
async def chat_with_bot(query: str):
    db_medicines = get_db_medicines()
    med_str = "\n".join([f"- {m['name']} | 数量:{m['count']} | 位置:{m['location']} | 有效期:{m['expire_date']} | 用法:{m['usage']}" for m in db_medicines])
    
    db_storage = get_db_storage()
    sto_str = "\n".join([f"- {s['item_name']}[{s['category']}] | 数量:{s['quantity']} | 位置:{s['location']} | 备注:{s['remark']}" for s in db_storage])

    db_memos = get_db_memos()
    memo_str = "\n".join([f"- 主题: {s['title']} | 内容: {s['content']}" for s in db_memos])

    system_prompt = f"""你是一个全能的家庭超级大管家智能体（FamilyBot）。
你目前已经成功接入了主人的【家庭药品数据库】、【家庭物资仓库数据库】和【家庭日常备忘录】。

【家庭药品清单】：
{med_str}

【家庭物资仓库清单】：
{sto_str}

【家庭备忘录信息】：
{memo_str}

要求：涉及寻找药品或日常物资时，必须明确告知主人物品的存放精确位置；日常备忘涉及账号密码或事件时准确回答；语言要简洁、亲切、专业。"""

    try:
        response = await client.chat.completions.create(
            model=MODEL_NAME,
            messages=[{"role": "system", "content": system_prompt}, {"role": "user", "content": query}],
            temperature=0.7
        )
        return {"reply": response.choices[0].message.content}
    except Exception as e:
        return {"reply": f"大管家思考失败，请检查密钥或网络: {str(e)}"}
```

### 2.4 主入口程序 `main.py`

```Python
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles
from fastapi.responses import RedirectResponse
from database import init_db
from routers import chat, medicine, storage, memo

app = FastAPI(title="FamilyBot - 家庭管家核心引擎")

# 初始化数据库结构
init_db()

# 一行代码优雅组装注册各个解耦后的路由模块
app.include_router(chat.router)
app.include_router(medicine.router)
app.include_router(storage.router)
app.include_router(memo.router)

@app.get("/")
async def read_root():
    return RedirectResponse(url="/static/index.html")

@app.get("/index.html")
async def read_index_html():
    return RedirectResponse(url="/static/index.html")

# 挂载静态文件目录
app.mount("/static", StaticFiles(directory="static"), name="static")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run("main:app", host="0.0.0.0", port=8000, reload=True)
```

## 三、 前端组件化与离线优先（Local-First）实现

离线优先废用

### 3.1 离线服务工作线程 `static/sw.js`（调试失败，废用）

相对路径设计，支持无网络环境下通过手机浏览器拦截缓存并秒开网页。

```JavaScript
const CACHE_NAME = 'familybot-cache-v3';
const urlsToCache = [
  './index.html',
  './manifest.json',
  'https://unpkg.com/vue@3/dist/vue.global.prod.js' 
];

self.addEventListener('install', event => {
  self.skipWaiting(); 
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(urlsToCache))
  );
});

self.addEventListener('activate', event => {
  event.waitUntil(
    caches.keys().then(cacheNames => {
      return Promise.all(
        cacheNames.map(cacheName => {
          if (cacheName !== CACHE_NAME) return caches.delete(cacheName);
        })
      );
    })
  );
  self.clients.claim();
});

self.addEventListener('fetch', event => {
  if (event.request.url.includes('/api/')) return; // 放行 API 请求，由 Vue 接管超时判定
  event.respondWith(
    fetch(event.request).catch(() => caches.match(event.request))
  );
});
```

### 3.2 应用配置清单 `static/manifest.json`（调试失败，废用）

```json
{
  "name": "FamilyBot 家庭管家",
  "short_name": "家庭管家",
  "start_url": "./index.html",
  "display": "standalone",
  "background_color": "#f5f6f9",
  "theme_color": "#3F51B5",
  "icons": [
    {
      "src": "data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'><text y='80' font-size='80'>🏠</text></svg>",
      "sizes": "192x192",
      "type": "image/svg+xml"
    }
  ]
}
```

### 3.3 前端统一交互界面 `static/index.html`

引入 Vue 3 的单页面数据驱动模型，重构了**防溢出竖排防御布局**、**Fetch 毫秒级断网检测**与**离线操作追补同步队列**。

```HTML
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <meta name="theme-color" content="#3F51B5">
  <title>FamilyBot 家庭管家</title>
  <script src="https://unpkg.com/vue@3/dist/vue.global.prod.js"></script>
  <style>
    * { box-sizing: border-box; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
    html, body { height: 100%; margin: 0; overflow: hidden; background: #f5f6f9; }
    #app { display: flex; flex-direction: column; height: 100%; }
    
    header { background: #3F51B5; color: white; padding: 14px; text-align: center; font-size: 18px; font-weight: bold; flex-shrink: 0; z-index: 10; }
    .content-section { flex: 1; overflow-y: auto; -webkit-overflow-scrolling: touch; }
    
    /* 聊天样式 */
    #chat-box { padding: 15px; display: flex; flex-direction: column; gap: 12px; padding-bottom: 80px; }
    .msg { max-width: 80%; padding: 12px 16px; border-radius: 12px; font-size: 15px; line-height: 1.5; word-wrap: break-word; }
    .bot { background: white; align-self: flex-start; color: #333; box-shadow: 0 1px 3px rgba(0,0,0,0.1); border-bottom-left-radius: 2px; }
    .user { background: #3F51B5; color: white; align-self: flex-end; border-bottom-right-radius: 2px; }
    
    .input-area { background: white; padding: 10px; display: flex; gap: 8px; border-top: 1px solid #eee; position: fixed; bottom: 56px; left:0; right:0; z-index: 10; }
    input[type="text"] { border: 1px solid #ddd; border-radius: 8px; padding: 10px; font-size: 14px; outline: none; width: 100%; background: #fafafa; }
    input[type="text"]:focus { border-color: #3F51B5; background: #fff; }
    .chat-input { border-radius: 20px !important; }
    
    .btn { background: #3F51B5; color: white; border: none; border-radius: 8px; padding: 8px 14px; font-size: 14px; cursor: pointer; font-weight: bold; }
    .voice-btn { background: #ff9800; border-radius: 50%; width: 42px; height: 42px; padding: 0; display: flex; align-items: center; justify-content: center; border:none; color:white; flex-shrink: 0; }
    .loading { align-self: flex-start; color: #888; font-size: 13px; margin-left: 5px; }

    /* 表单与卡片 */
    .manager-padding { padding: 15px 15px 80px 15px; } 
    .card { background: white; padding: 16px; border-radius: 12px; box-shadow: 0 2px 6px rgba(0,0,0,0.05); margin-bottom: 15px; }
    .card h3 { margin-top: 0; margin-bottom: 15px; font-size: 16px; color: #333; border-bottom: 1px solid #eee; padding-bottom: 8px; }
    .form-group { margin-bottom: 12px; display: flex; flex-direction: column; gap: 6px; }
    .form-group label { font-size: 13px; color: #666; font-weight: bold; }
    .form-control { border: 1px solid #ddd; border-radius: 8px; padding: 10px 12px; font-size: 14px; outline: none; width: 100%; background: #fafafa; }
    .form-control:focus { border-color: #3F51B5; background: #fff; }
    
    .item-card { background: white; border-radius: 12px; padding: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.06); display: flex; flex-direction: column; gap: 10px; margin-bottom: 15px; }
    .med-border { border-left: 5px solid #4CAF50; }
    .storage-border { border-left: 5px solid #FF9800; }
    .memo-border { border-left: 5px solid #9C27B0; }
    .item-card-header { display: flex; justify-content: space-between; align-items: center; gap: 10px; border-bottom: 1px dashed #eee; padding-bottom: 8px; }
    .item-title-input { font-weight: bold; font-size: 16px; border: none; background: none; flex: 1; outline: none; padding: 4px 0; color: #222; }
    .action-btns { display: flex; gap: 6px; }
    
    textarea { border: 1px solid #ddd; border-radius: 8px; padding: 10px; font-size: 14px; outline: none; resize: vertical; background: #fafafa; width: 100%; }
    textarea:focus { border-color: #9C27B0; background: #fff; }

    /* 底部 Tab */
    .nav-bar { position: fixed; bottom: 0; left: 0; right: 0; height: 56px; background: white; border-top: 1px solid #e0e0e0; display: flex; justify-content: space-around; align-items: center; z-index: 100; }
    .nav-item { background: none; border: none; flex: 1; height: 100%; display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 2px; color: #757575; font-size: 11px; cursor: pointer; }
    .nav-item.active { color: #3F51B5; font-weight: bold; }
    .nav-icon { font-size: 18px; }
  </style>
</head>
<body>
<div id="app">
  <header>{{ headerTitle }}</header>

  <div v-show="currentTab === 'home'" class="content-section">
    <div id="chat-box" ref="chatBox">
      <div v-for="(msg, index) in chatHistory" :key="index" :class="['msg', msg.role]">
        {{ msg.text }}
      </div>
      <div v-if="isThinking" class="loading">大管家正在检索数据库...</div>
    </div>
    <div class="input-area">
      <button class="voice-btn" @click="startVoice">{{ isRecording ? '🎙️' : '🎤' }}</button>
      <input type="text" class="chat-input" v-model="userInput" @keypress.enter="sendMessage" placeholder="有事请吩咐大管家...">
      <button class="btn" style="border-radius:20px;" @click="sendMessage">发送</button>
    </div>
  </div>

  <div v-show="currentTab === 'medicine'" class="content-section manager-padding">
    <div class="card">
      <h3>➕ 录入新药品</h3>
      <div class="form-group"><label>药品名称 *</label><input type="text" v-model="newMed.name" class="form-control" placeholder="如：布洛芬"></div>
      <div class="form-group"><label>数量 / 规格</label><input type="text" v-model="newMed.count" class="form-control"></div>
      <div class="form-group"><label>存放位置</label><input type="text" v-model="newMed.location" class="form-control"></div>
      <div class="form-group"><label>有效期</label><input type="text" v-model="newMed.expire_date" class="form-control"></div>
      <div class="form-group"><label>用法用量</label><input type="text" v-model="newMed.usage" class="form-control"></div>
      <button class="btn" style="width:100%; background:#4CAF50;" @click="addMedicine">保存到药箱</button>
    </div>
    <div class="card">
      <h3>📋 药箱库存</h3>
      <div v-if="medList.length === 0" style="text-align:center;color:#999;font-size:13px;">暂无数据</div>
      <div v-for="med in medList" :key="med.id" class="item-card med-border">
        <div class="item-card-header">
          <input type="text" v-model="med.name" class="item-title-input">
          <div class="action-btns">
            <button class="btn" style="background:#f44336;padding:6px 12px;font-size:13px;" @click="deleteMedicine(med.id)">删除</button>
            <button class="btn" style="background:#2196F3;padding:6px 12px;font-size:13px;" @click="updateMedicine(med)">保存</button>
          </div>
        </div>
        <div class="form-group"><label>数量/规格</label><input type="text" v-model="med.count" class="form-control"></div>
        <div class="form-group"><label>存放位置</label><input type="text" v-model="med.location" class="form-control"></div>
        <div class="form-group"><label>有效期</label><input type="text" v-model="med.expire_date" class="form-control"></div>
        <div class="form-group"><label>用法用量</label><input type="text" v-model="med.usage" class="form-control"></div>
      </div>
    </div>
  </div>

  <div v-show="currentTab === 'storage'" class="content-section manager-padding">
    <div class="card">
      <h3>➕ 录入新物资</h3>
      <div class="form-group"><label>物品名称 *</label><input type="text" v-model="newStorage.item_name" class="form-control"></div>
      <div class="form-group"><label>分类</label><input type="text" v-model="newStorage.category" class="form-control"></div>
      <div class="form-group"><label>数量 / 规格</label><input type="text" v-model="newStorage.quantity" class="form-control"></div>
      <div class="form-group"><label>存放位置</label><input type="text" v-model="newStorage.location" class="form-control"></div>
      <div class="form-group"><label>备注说明</label><input type="text" v-model="newStorage.remark" class="form-control"></div>
      <button class="btn" style="width:100%; background:#FF9800;" @click="addStorage">保存到仓库</button>
    </div>
    <div class="card">
      <h3>📋 仓库库存</h3>
      <div v-if="storageList.length === 0" style="text-align:center;color:#999;font-size:13px;">暂无数据</div>
      <div v-for="item in storageList" :key="item.id" class="item-card storage-border">
        <div class="item-card-header">
          <input type="text" v-model="item.item_name" class="item-title-input">
          <div class="action-btns">
            <button class="btn" style="background:#f44336;padding:6px 12px;font-size:13px;" @click="deleteStorage(item.id)">删除</button>
            <button class="btn" style="background:#FF9800;padding:6px 12px;font-size:13px;" @click="updateStorage(item)">保存</button>
          </div>
        </div>
        <div class="form-group"><label>分类</label><input type="text" v-model="item.category" class="form-control"></div>
        <div class="form-group"><label>数量/规格</label><input type="text" v-model="item.quantity" class="form-control"></div>
        <div class="form-group"><label>存放位置</label><input type="text" v-model="item.location" class="form-control"></div>
        <div class="form-group"><label>备注</label><input type="text" v-model="item.remark" class="form-control"></div>
      </div>
    </div>
  </div>

  <div v-show="currentTab === 'memo'" class="content-section manager-padding">
    <div class="card">
      <h3>➕ 新增备忘条目</h3>
      <div class="form-group"><label>备忘主题 / 问题 *</label><input type="text" v-model="newMemo.title" class="form-control"></div>
      <div class="form-group"><label>详细记录内容</label><textarea v-model="newMemo.content" rows="3"></textarea></div>
      <button class="btn" style="width:100%; background:#9C27B0;" @click="addMemo">保存到备忘录</button>
    </div>
    <div class="card">
      <h3>📋 备忘历史记录</h3>
      <div v-if="memoList.length === 0" style="text-align:center;color:#999;font-size:13px;">暂无备忘数据</div>
      <div v-for="memo in memoList" :key="memo.id" class="item-card memo-border">
        <div class="item-card-header">
          <input type="text" v-model="memo.title" class="item-title-input">
          <div class="action-btns">
            <button class="btn" style="background:#f44336;padding:6px 12px;font-size:13px;" @click="deleteMemo(memo.id)">删除</button>
            <button class="btn" style="background:#9C27B0;padding:6px 12px;font-size:13px;" @click="updateMemo(memo)">保存</button>
          </div>
        </div>
        <div class="form-group"><label>详细内容</label><textarea v-model="memo.content" rows="3"></textarea></div>
      </div>
    </div>
  </div>

  <nav class="nav-bar">
    <button :class="['nav-item', {active: currentTab==='home'}]" @click="switchTab('home', '🏠 FamilyBot 家庭管家')">
      <span class="nav-icon">🏠</span><span>管家问答</span>
    </button>
    <button :class="['nav-item', {active: currentTab==='medicine'}]" @click="switchTab('medicine', '💊 智能药箱')">
      <span class="nav-icon">💊</span><span>智能药箱</span>
    </button>
    <button :class="['nav-item', {active: currentTab==='storage'}]" @click="switchTab('storage', '📦 家庭仓库')">
      <span class="nav-icon">📦</span><span>家庭仓库</span>
    </button>
    <button :class="['nav-item', {active: currentTab==='memo'}]" @click="switchTab('memo', '📝 家庭备忘录')">
      <span class="nav-icon">📝</span><span>家庭备忘</span>
    </button>
  </nav>
</div>

<script>
  const { createApp } = Vue;
  createApp({
    data() {
      return {
        headerTitle: '🏠 FamilyBot 家庭管家',
        currentTab: 'home',
        chatHistory: [{ role: 'bot', text: '欢迎回来！我是你的家庭超级管家 FamilyBot。' }],
        userInput: '',
        isThinking: false,
        isRecording: false,
        medList: [],
        storageList: [],
        memoList: [],
        newMed: { name: '', count: '', location: '', expire_date: '', usage: '' },
        newStorage: { item_name: '', category: '', quantity: '', location: '', remark: '' },
        newMemo: { title: '', content: '' }
      }
    },
    methods: {
      switchTab(tab, title) {
        this.currentTab = tab;
        this.headerTitle = title;
        if(tab === 'medicine') this.fetchData('/api/medicines', 'medList');
        if(tab === 'storage') this.fetchData('/api/storage', 'storageList');
        if(tab === 'memo') this.fetchData('/api/memos', 'memoList');
      },
      async sendMessage() {
        if (!this.userInput.trim()) return;
        const query = this.userInput;
        this.chatHistory.push({ role: 'user', text: query });
        this.userInput = '';
        this.isThinking = true;
        this.$nextTick(() => { this.scrollToBottom(); });

        try {
          const res = await fetch(`/api/chat?query=${encodeURIComponent(query)}`);
          const data = await res.json();
          this.chatHistory.push({ role: 'bot', text: data.reply });
        } catch (e) {
          this.chatHistory.push({ role: 'bot', text: '网络连接失败，请确认后端服务是否正常运行。' });
        } finally {
          this.isThinking = false;
          this.$nextTick(() => { this.scrollToBottom(); });
        }
      },
      scrollToBottom() {
        const box = this.$refs.chatBox;
        if(box) box.scrollTop = box.scrollHeight;
      },
      async fetchData(url, targetVar) {
        try {
          const res = await fetch(url);
          const data = await res.json();
          if (data.status === 'success') this[targetVar] = data.data;
        } catch (e) { console.error("加载数据失败", e); }
      },
      
      // --- 药箱方法 ---
      async addMedicine() {
        if (!this.newMed.name) return alert('请输入名称');
        const params = new URLSearchParams(this.newMed).toString();
        await fetch(`/api/add_medicine?${params}`, { method: 'POST' });
        alert('录入成功');
        this.newMed = { name: '', count: '', location: '', expire_date: '', usage: '' };
        this.fetchData('/api/medicines', 'medList');
      },
      async updateMedicine(med) {
        const params = new URLSearchParams(med).toString();
        await fetch(`/api/update_medicine?${params}`, { method: 'POST' });
        alert('修改成功');
        this.fetchData('/api/medicines', 'medList');
      },
      async deleteMedicine(id) {
        if (confirm('确定要删除这条药品记录吗？')) {
          await fetch(`/api/delete_medicine?id=${id}`, { method: 'POST' });
          this.fetchData('/api/medicines', 'medList');
        }
      },

      // --- 仓库方法 ---
      async addStorage() {
        if (!this.newStorage.item_name) return alert('请输入名称');
        const params = new URLSearchParams(this.newStorage).toString();
        await fetch(`/api/add_storage?${params}`, { method: 'POST' });
        alert('录入成功');
        this.newStorage = { item_name: '', category: '', quantity: '', location: '', remark: '' };
        this.fetchData('/api/storage', 'storageList');
      },
      async updateStorage(item) {
        const params = new URLSearchParams(item).toString();
        await fetch(`/api/update_storage?${params}`, { method: 'POST' });
        alert('修改成功');
        this.fetchData('/api/storage', 'storageList');
      },
      async deleteStorage(id) {
        if (confirm('确定要删除这条物资记录吗？')) {
          await fetch(`/api/delete_storage?id=${id}`, { method: 'POST' });
          this.fetchData('/api/storage', 'storageList');
        }
      },

      // --- 备忘录方法 ---
      async addMemo() {
        if (!this.newMemo.title) return alert('请输入主题描述');
        const params = new URLSearchParams(this.newMemo).toString();
        await fetch(`/api/add_memo?${params}`, { method: 'POST' });
        alert('录入成功');
        this.newMemo = { title: '', content: '' };
        this.fetchData('/api/memos', 'memoList');
      },
      async updateMemo(memo) {
        const params = new URLSearchParams({ id: memo.id, title: memo.title, content: memo.content }).toString();
        await fetch(`/api/update_memo?${params}`, { method: 'POST' });
        alert('修改成功');
        this.fetchData('/api/memos', 'memoList');
      },
      async deleteMemo(id) {
        if (confirm('确定要删除这条备忘信息吗？')) {
          await fetch(`/api/delete_memo?id=${id}`, { method: 'POST' });
          this.fetchData('/api/memos', 'memoList');
        }
      },

      // --- 语音识别 ---
      startVoice() {
        const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
        if (!SpeechRecognition) return alert('您的浏览器不支持语音识别');
        const recognition = new SpeechRecognition();
        recognition.lang = 'zh-CN';
        recognition.start();
        this.isRecording = true;
        recognition.onresult = (e) => {
          this.userInput = e.results[0][0].transcript;
          this.isRecording = false;
          this.sendMessage();
        };
        recognition.onerror = () => { this.isRecording = false; };
        recognition.onend = () => { this.isRecording = false; };
      }
    },
    mounted() {
      // 首次加载初始化拉取
      this.fetchData('/api/medicines', 'medList');
      this.fetchData('/api/storage', 'storageList');
      this.fetchData('/api/memos', 'memoList');
    }
  }).mount('#app');
</script>
</body>
</html>
```

## 四、 树莓派生产环境部署步骤

当项目在本地 Windows 电脑测试无误后，执行以下步骤将其持久化部署到树莓派 4B。

### 步骤 1：传输项目代码

将你本地电脑上的整个 `familyBot` 文件夹（包含 `static/`、`routers/` 目录以及 `main.py`、`config.py`、`database.py`、`.env` 文件）通过 SCP 命令行或 VS Code SSH 拖拽方式，传输至树莓派的 `/home/pi/familyBot/` 目录下。

### 步骤 2：安装运行依赖库(虚拟环境)

使用 SSH 连接上树莓派，进入对应项目文件夹，执行以下命令安装运行环境：

创建虚拟环境，并安装所需包

```Bash
python3 -m venv .venv
source .venv/bin/activate #激活虚拟环境
pip install python-multipart fastapi uvicorn openai python-dotenv #安装包
```

```Bash
#下面这种办法强行将所需包安装至系统内部
cd /home/pi/familyBot/
pip3 install fastapi uvicorn python-dotenv openai --break-system-packages 2>/dev/null || pip3 install fastapi uvicorn python-dotenv openai
```

### 步骤 3：编写 Linux 服务配置文件

创建一个 Linux 系统服务管理文件，以实现服务在后台 24 小时稳定运行：



```Bash
sudo nano /etc/systemd/system/familybot.service
```

将以下配置内容写入其中：

*使用虚拟环境.venv中python运行脚本*

```Plaintext
[Unit]
Description=FamilyBot Smart Home Server
After=network.target

[Service]
User=pi
WorkingDirectory=/home/pi/familyBot

ExecStart=/home/pi/familyBot/.venv/bin/uvicorn main:app --host 0.0.0.0 --port 8000
Restart=always
RestartSec=5
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
```

保存并退出（`Ctrl + O` 保存，`Ctrl + X` 退出）。

### 步骤 4：激活与启动服务

在终端依次执行以下命令，刷新系统配置并激活开机自启：

Bash

```
# 1. 刷新系统服务列表
sudo systemctl daemon-reload

# 2. 设置服务开机自动启动
sudo systemctl enable familybot

# 3. 立刻启动当前管家服务
sudo systemctl start familybot
```

### 步骤 5：服务状态与自启核验

- **检查服务是否成功开机自启**：输入 `systemctl is-enabled familybot`，若输出 `enabled` 则代表成功。
- **查看服务当前运行日志与状态**：输入 `sudo systemctl status familybot`，若看到绿色的 `active (running)` 字样，说明大管家已经在树莓派后台完美起飞！

## 五、前端代码重构

采用**原生 ES Modules** 的方式把 `index.html` 拆解开。

### 📂 拆分后的前端目录结构

拆分后，`static/` 文件夹会变得非常规整清晰：



```Plaintext
static/
│  index.html            # 极简的主页面壳子（只剩头部、底部 Tab 导航和组件挂载点）
│
├─css/
│      style.css         # 抽离出来的全局与卡片 CSS 样式
│
└─components/            # 拆分出来的子组件（每个模块对应一个文件）
        ChatTab.js       # 1. 管家问答组件
        MedicineTab.js   # 2. 智能药箱组件（含搜索过滤逻辑）
        StorageTab.js    # 3. 家庭仓库组件（含搜索过滤逻辑）
        MemoTab.js       # 4. 家庭备忘录组件（含搜索过滤逻辑）
```

### 🛠️ 拆分步骤与源码

#### 1. 新建 `static/css/style.css`

把所有 CSS 样式抽离到独立文件：



```CSS
* { box-sizing: border-box; font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
html, body { height: 100%; margin: 0; overflow: hidden; background: #f5f6f9; }
#app { display: flex; flex-direction: column; height: 100%; }

header { background: #3F51B5; color: white; padding: 14px; text-align: center; font-size: 18px; font-weight: bold; flex-shrink: 0; z-index: 10; }
.content-section { flex: 1; overflow-y: auto; -webkit-overflow-scrolling: touch; }

/* 聊天 */
#chat-box { padding: 15px; display: flex; flex-direction: column; gap: 12px; padding-bottom: 80px; }
.msg { max-width: 80%; padding: 12px 16px; border-radius: 12px; font-size: 15px; line-height: 1.5; word-wrap: break-word; }
.bot { background: white; align-self: flex-start; color: #333; box-shadow: 0 1px 3px rgba(0,0,0,0.1); border-bottom-left-radius: 2px; }
.user { background: #3F51B5; color: white; align-self: flex-end; border-bottom-right-radius: 2px; }

.input-area { background: white; padding: 10px; display: flex; gap: 8px; border-top: 1px solid #eee; position: fixed; bottom: 56px; left:0; right:0; z-index: 10; }
input[type="text"] { border: 1px solid #ddd; border-radius: 8px; padding: 10px; font-size: 14px; outline: none; width: 100%; background: #fafafa; }
input[type="text"]:focus { border-color: #3F51B5; background: #fff; }
.chat-input { border-radius: 20px !important; }

.btn { background: #3F51B5; color: white; border: none; border-radius: 8px; padding: 8px 14px; font-size: 14px; cursor: pointer; font-weight: bold; }
.voice-btn { background: #ff9800; border-radius: 50%; width: 42px; height: 42px; padding: 0; display: flex; align-items: center; justify-content: center; border:none; color:white; flex-shrink: 0; }
.loading { align-self: flex-start; color: #888; font-size: 13px; margin-left: 5px; }

/* 表单卡片 */
.manager-padding { padding: 15px 15px 80px 15px; } 
.card { background: white; padding: 16px; border-radius: 12px; box-shadow: 0 2px 6px rgba(0,0,0,0.05); margin-bottom: 15px; }
.card h3 { margin-top: 0; margin-bottom: 15px; font-size: 16px; color: #333; border-bottom: 1px solid #eee; padding-bottom: 8px; }
.form-group { margin-bottom: 12px; display: flex; flex-direction: column; gap: 6px; }
.form-group label { font-size: 13px; color: #666; font-weight: bold; }
.form-control { border: 1px solid #ddd; border-radius: 8px; padding: 10px 12px; font-size: 14px; outline: none; width: 100%; background: #fafafa; }
.form-control:focus { border-color: #3F51B5; background: #fff; }

.search-input { border: 2px solid #3F51B5; background: #fff; font-size: 14px; font-weight: 500; }
.item-card { background: white; border-radius: 12px; padding: 16px; box-shadow: 0 2px 8px rgba(0,0,0,0.06); display: flex; flex-direction: column; gap: 10px; margin-bottom: 15px; }
.med-border { border-left: 5px solid #4CAF50; }
.storage-border { border-left: 5px solid #FF9800; }
.memo-border { border-left: 5px solid #9C27B0; }
.item-card-header { display: flex; justify-content: space-between; align-items: center; gap: 10px; border-bottom: 1px dashed #eee; padding-bottom: 8px; }
.item-title-input { font-weight: bold; font-size: 16px; border: none; background: none; flex: 1; outline: none; padding: 4px 0; color: #222; }
.action-btns { display: flex; gap: 6px; }

textarea { border: 1px solid #ddd; border-radius: 8px; padding: 10px; font-size: 14px; outline: none; resize: vertical; background: #fafafa; width: 100%; }

/* 底部 Tab */
.nav-bar { position: fixed; bottom: 0; left: 0; right: 0; height: 56px; background: white; border-top: 1px solid #e0e0e0; display: flex; justify-content: space-around; align-items: center; z-index: 100; }
.nav-item { background: none; border: none; flex: 1; height: 100%; display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 2px; color: #757575; font-size: 11px; cursor: pointer; }
.nav-item.active { color: #3F51B5; font-weight: bold; }
.nav-icon { font-size: 18px; }
```

#### 2. 新建子组件（以 `MedicineTab.js` 为例，其他组件同理）

在 `static/components/` 文件夹下新建 `MedicineTab.js`：



```JavaScript
export default {
  name: 'MedicineTab',
  template: `
    <div class="content-section manager-padding">
      <div class="card">
        <h3>➕ 录入新药品</h3>
        <div class="form-group"><label>药品名称 *</label><input type="text" v-model="newMed.name" class="form-control" placeholder="如：布洛芬"></div>
        <div class="form-group"><label>数量 / 规格</label><input type="text" v-model="newMed.count" class="form-control"></div>
        <div class="form-group"><label>存放位置</label><input type="text" v-model="newMed.location" class="form-control"></div>
        <div class="form-group"><label>有效期</label><input type="text" v-model="newMed.expire_date" class="form-control"></div>
        <div class="form-group"><label>用法用量</label><input type="text" v-model="newMed.usage" class="form-control"></div>
        <button class="btn" style="width:100%; background:#4CAF50;" @click="addMedicine">保存到药箱</button>
      </div>
      <div class="card">
        <h3>📋 药箱库存 (显示: {{ filteredMedList.length }} / 共: {{ medList.length }})</h3>
        <div style="margin-bottom: 15px;">
          <input type="text" v-model="searchKey" class="form-control search-input" placeholder="🔍 快速搜索药品名称、位置、用途...">
        </div>
        <div v-if="filteredMedList.length === 0" style="text-align:center;color:#999;font-size:13px;padding:15px 0;">
          {{ searchKey ? '未找到符合条件的药品' : '暂无数据' }}
        </div>
        <div v-for="med in filteredMedList" :key="med.id" class="item-card med-border">
          <div class="item-card-header">
            <input type="text" v-model="med.name" class="item-title-input">
            <div class="action-btns">
              <button class="btn" style="background:#f44336;padding:6px 12px;font-size:13px;" @click="deleteMedicine(med.id)">删除</button>
              <button class="btn" style="background:#2196F3;padding:6px 12px;font-size:13px;" @click="updateMedicine(med)">保存</button>
            </div>
          </div>
          <div class="form-group"><label>数量/规格</label><input type="text" v-model="med.count" class="form-control"></div>
          <div class="form-group"><label>存放位置</label><input type="text" v-model="med.location" class="form-control"></div>
          <div class="form-group"><label>有效期</label><input type="text" v-model="med.expire_date" class="form-control"></div>
          <div class="form-group"><label>用法用量</label><input type="text" v-model="med.usage" class="form-control"></div>
        </div>
      </div>
    </div>
  `,
  data() {
    return {
      searchKey: '',
      medList: [],
      newMed: { name: '', count: '', location: '', expire_date: '', usage: '' }
    }
  },
  computed: {
    filteredMedList() {
      if (!this.searchKey.trim()) return this.medList;
      const key = this.searchKey.toLowerCase();
      return this.medList.filter(item => 
        (item.name && item.name.toLowerCase().includes(key)) ||
        (item.location && item.location.toLowerCase().includes(key)) ||
        (item.usage && item.usage.toLowerCase().includes(key)) ||
        (item.expire_date && item.expire_date.toLowerCase().includes(key))
      );
    }
  },
  methods: {
    async fetchList() {
      const res = await fetch('/api/medicines');
      const data = await res.json();
      if (data.status === 'success') this.medList = data.data;
    },
    async addMedicine() {
      if (!this.newMed.name) return alert('请输入名称');
      const params = new URLSearchParams(this.newMed).toString();
      await fetch(`/api/add_medicine?${params}`, { method: 'POST' });
      alert('录入成功');
      this.newMed = { name: '', count: '', location: '', expire_date: '', usage: '' };
      this.fetchList();
    },
    async updateMedicine(med) {
      const params = new URLSearchParams(med).toString();
      await fetch(`/api/update_medicine?${params}`, { method: 'POST' });
      alert('修改成功');
      this.fetchList();
    },
    async deleteMedicine(id) {
      if (confirm('确定要删除这条记录吗？')) {
        await fetch(`/api/delete_medicine?id=${id}`, { method: 'POST' });
        this.fetchList();
      }
    }
  },
  mounted() {
    this.fetchList();
  }
}
```

#### 3. 极简的入口 `static/index.html`（只剩约 60 行！）

拆分后，主文件引入 ES 模块脚本 (`type="module"`)，代码变得极其干净清爽：



```HTML
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <title>FamilyBot 家庭管家</title>
  <link rel="stylesheet" href="./css/style.css">
  <script src="https://unpkg.com/vue@3/dist/vue.global.prod.js"></script>
</head>
<body>
<div id="app">
  <header>{{ headerTitle }}</header>

  <chat-tab v-show="currentTab === 'home'"></chat-tab>
  <medicine-tab v-show="currentTab === 'medicine'"></medicine-tab>
  <storage-tab v-show="currentTab === 'storage'"></storage-tab>
  <memo-tab v-show="currentTab === 'memo'"></memo-tab>

  <nav class="nav-bar">
    <button :class="['nav-item', {active: currentTab==='home'}]" @click="switchTab('home', '🏠 FamilyBot 家庭管家')">
      <span class="nav-icon">🏠</span><span>管家问答</span>
    </button>
    <button :class="['nav-item', {active: currentTab==='medicine'}]" @click="switchTab('medicine', '💊 智能药箱')">
      <span class="nav-icon">💊</span><span>智能药箱</span>
    </button>
    <button :class="['nav-item', {active: currentTab==='storage'}]" @click="switchTab('storage', '📦 家庭仓库')">
      <span class="nav-icon">📦</span><span>家庭仓库</span>
    </button>
    <button :class="['nav-item', {active: currentTab==='memo'}]" @click="switchTab('memo', '📝 家庭备忘录')">
      <span class="nav-icon">📝</span><span>家庭备忘</span>
    </button>
  </nav>
</div>

<script type="module">
  import ChatTab from './components/ChatTab.js';
  import MedicineTab from './components/MedicineTab.js';
  import StorageTab from './components/StorageTab.js';
  import MemoTab from './components/MemoTab.js';

  const { createApp } = Vue;
  createApp({
    components: { ChatTab, MedicineTab, StorageTab, MemoTab },
    data() {
      return {
        headerTitle: '🏠 FamilyBot 家庭管家',
        currentTab: 'home'
      }
    },
    methods: {
      switchTab(tab, title) {
        this.currentTab = tab;
        this.headerTitle = title;
      }
    }
  }).mount('#app');
</script>
</body>
</html>
```

## 六、后续升级提问AI模板

**提问模板：** “AI，我想为我的本地项目 **FamilyBot** 增加一个新模块：**[在这里写你的新模块名字，例如：本地密码管理器]**。

以下是该项目的**核心架构蓝图与目录规范**：

```
分析这个项目，并在上下文中记录该项目的整体结构，目录结构以及所有源码，同时需要知道# FamilyBot 项目上下文蓝图

## 1. 系统愿景与定位
FamilyBot 是一个部署在树莓派/本地的 24 小时全能家庭管家大中台。
主入口是一个全能 AI 问答聊天框，它通过并发读取本地 SQLite 的所有数据表（药箱、仓库、备忘录等），拼装进 System Prompt，实现轻量级内网 RAG（检索增强生成）架构。

## 2. 技术栈与设计原则
- 后端：FastAPI + SQLite3 + AsyncOpenAI (GLM-4-flash)。
- 前端：单文件 Vue 3 (CDN 引入) + HTML5 PWA (Service Worker 相对路径拦截)。
- 架构原则：
  1. 后端必须使用 `APIRouter` 实现物理隔离与解耦，每个模块在 `routers/` 下独立为一个文件。
  2. 前端采用 Local-First（本地优先）架构，使用 LocalStorage 作为数据快照和离线写操作队列，具备毫秒级超时断网检测，支持回家 WiFi 自动心跳同步。
  3. 前端 UI 采用完全竖排的移动端防御布局，防止多列输入撑爆卡片。

## 3. 规范文件目录树
familyBot/
├── .env                # 存放 FAMILYBOT_API_KEY 等
├── config.py           # 配置读取
├── database.py         # SQLite 统一建表与公共查询 (包含 init_db)
├── main.py             # 极简主入口，负责 app.include_router
├── routers/            # 路由层
│   ├── chat.py         # AI 核心问答（读取全库拼装 Prompt）
│   ├── medicine.py     # 智能药箱 CRUD (带 delete_medicine)
│   ├── storage.py      # 家庭仓库 CRUD (带 delete_storage)
│   └── memo.py         # 家庭备忘录 CRUD (带 delete_memo)
└── static/             # 静态前端
    ├── index.html      # Vue3 单页面交互与离线乐观队列
    ├── manifest.json   # PWA 配置文件
    └── sw.js           # SW 相对路径离线秒开缓存。 把这些内容都记录到上下文，接下来我会继续开发这个项目，新增或修改功能，你要能够整体判断。另外项目内是有.env环境文件和.db的，只不过我没有上传
```

**你**：“Gemini，帮我把**家庭记账**模块也改成通用折叠卡片模式。

1. 列表常显：`账目名称`。
2. 徽章挂：主徽章 `金额`，副徽章 `消费分类`（如：餐饮、娱乐）。
3. 展开后编辑：`账目名称`、`金额`、`分类`、`消费日期`、`备注`。
4. 底部按钮：保留【删除】和【保存修改】。
5. 主题色：想要个**淡粉色/红酒色**系。”



>  **“帮我把 [某某模块] 里的 [某某字段] 升级为『Markdown 流式面板』。要求：默认展示富文本，带 '👁️预览 / ✍️编辑' 切换开关，并且保存后自动切回预览模式。”**

### 📝 为什么这句暗号管用？它会触发我做什么？

当你发出这个指令时，我会在脑海中自动执行以下 4 步标准作业流程（SOP），一步到位给你完美代码：

1. **状态注入**：我会在该组件的 `fetchList()` 遍历中，精准为你添加对应的布尔值开关（如 `isEditingRemark: false`）。
2. **防崩渲染器**：我会在 `methods` 里自动为你补齐那个带有降级保护的 `renderMarkdown(text)` 函数。
3. **UI 替换**：我会把旧的 `<textarea>` 替换成 `v-if` / `v-else` 的双态结构，套用你最喜欢的 `.accordion-markdown-panel` 样式。
4. **闭环体验**：我会在提交 `update` 的方法里，加上保存成功后自动把开关拨回 `false` 的逻辑。

以下是目前 **`database.py` 中已有的表结构**：



```Python
import sqlite3
from config import DB_PATH

def init_db():
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    
    # 1. 创建药品表
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS medicines (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL, 
            count TEXT, 
            location TEXT,
            expire_date TEXT, 
            usage TEXT DEFAULT '', 
            eff TEXT DEFAULT ''
        )
    ''')
    
    # 2. 创建仓库表
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS storage (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            item_name TEXT NOT NULL, 
            category TEXT DEFAULT '',
            quantity TEXT DEFAULT '', 
            location TEXT DEFAULT '', 
            remark TEXT DEFAULT ''
        )
    ''')
    # 3. 新增：家庭备忘录表
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS memos (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            title TEXT NOT NULL,
            content TEXT DEFAULT '',
            created_at TEXT DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    conn.commit()
    conn.close()

def get_db_medicines():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM medicines ORDER BY id ASC")
    rows = cursor.fetchall()
    conn.close()
    return rows

def get_db_storage():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM storage ORDER BY id ASC")
    rows = cursor.fetchall()
    conn.close()
    return rows
# 新增：获取所有备忘录记录
def get_db_memos():
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM memos ORDER BY id DESC") # 按时间倒序，最新的在上面
    rows = cursor.fetchall()
    conn.close()
    return rows
```

请严格按照上述模块化解耦规范，帮我规划新模块的实现。请输出：

1. `database.py` 里需要新增的建表语句与读取函数。
2. 在 `routers/` 下需要新建的路由文件源码。
3. `routers/chat.py` 中如何将新数据喂给大管家大脑。
4. 前端 `index.html` 中需要增补的 Vue 模板卡片和 Methods 方法。”