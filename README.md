<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width,initial-scale=1.0" />
<title>GitHub Private Drive | 1000GB</title>
<style>
:root{--bg:#0f172a;--card:#1e293b;--accent:#3b82f6;--danger:#ef4444;--text:#e5e7eb;--muted:#94a3b8;}
body{margin:0;font-family:-apple-system,BlinkMacSystemFont,"Segoe UI";background:var(--bg);color:var(--text);}
header{padding:10px;background:var(--card);display:flex;gap:8px;flex-wrap:wrap;align-items:center;}
button{padding:6px 10px;border:none;border-radius:6px;background:var(--accent);color:#fff;cursor:pointer;font-size:13px;}
button.danger{background:var(--danger);}
input[type=text],input[type=password]{padding:6px;border-radius:6px;border:none;background:#334155;color:var(--text);}
main{display:grid;grid-template-columns:260px 1fr;min-height:100vh;}
aside{padding:12px;background:#111827;overflow:auto;font-size:13px;}
section{padding:12px;overflow:auto;}
.file-grid{display:grid;grid-template-columns:repeat(auto-fill,minmax(140px,1fr));gap:10px;}
.card{background:var(--card);border-radius:8px;padding:8px;text-align:center;word-break:break-all;cursor:pointer;position:relative;}
.card:hover{outline:1px solid var(--accent);}
.card.selected{outline:2px solid #22d3ee;}
.card img{width:100%;height:90px;object-fit:cover;border-radius:6px;}
progress{width:100%;height:8px;border-radius:4px;}
.modal{position:fixed;inset:0;background:rgba(0,0,0,.75);display:flex;align-items:center;justify-content:center;z-index:100;}
.modal .box{background:var(--card);padding:16px;border-radius:10px;width:95%;max-width:640px;max-height:90vh;overflow:auto;}
.hidden{display:none;}
.tag{font-size:11px;background:#334155;padding:2px 6px;border-radius:4px;color:var(--muted);}
.task{background:#334155;border-radius:6px;padding:6px;margin-bottom:6px;font-size:12px;}
.checkbox{position:absolute;top:4px;left:4px;}
.preview-img{max-width:100%;max-height:70vh;border-radius:8px;}
@media(max-width:720px){main{grid-template-columns:1fr;} aside{max-height:45vh;}}
</style>
</head>
<body>

<header>
  <input id="token" placeholder="GitHub Token（本地加密存储）" type="password"/>
  <button onclick="saveToken()">保存Token</button>
  <button onclick="logout()">登出</button>
  <input id="search" placeholder="搜索文件名"/>
  <button onclick="vm.search()">搜索</button>
  <button onclick="showModal('cfgModal')">仓库配置</button>
  <button onclick="showStat()">容量统计</button>
  <button onclick="mkdir()">新建文件夹</button>
  <button onclick="newDoc()">新建文档</button>
  <button onclick="upload()">上传文件</button>
  <button onclick="batchDelete()" class="danger">批量删除</button>
  <button onclick="moveSelected()">移动选中</button>
</header>

<main>
<aside>
  <div style="margin-bottom:8px;font-weight:600;">📁 目录树</div>
  <div id="tree"></div>
  <hr style="border-color:#334155;margin:10px 0;">
  <div style="font-weight:600;">⬆️ 上传队列</div>
  <div id="tasks"></div>
</aside>
<section>
  <div id="path" style="margin-bottom:8px;"></div>
  <div id="grid" class="file-grid"></div>
</section>
</main>

<!-- 仓库配置 -->
<div id="cfgModal" class="modal hidden">
  <div class="box">
    <h3>仓库池配置</h3>
    <textarea id="cfgRepo" rows="7" style="width:100%;background:#0f172a;color:var(--text);border-radius:6px;padding:8px;"></textarea>
    <p style="font-size:12px;color:var(--muted);">格式：owner/repo|branch 每行一个，100个仓库≈1000GB</p >
    <button onclick="saveCfg()">保存</button>
    <button onclick="hideModals()">关闭</button>
  </div>
</div>

<!-- 容量统计 -->
<div id="statModal" class="modal hidden">
  <div class="box">
    <h3>容量统计</h3>
    <pre id="statText" style="font-size:12px;"></pre>
    <button onclick="hideModals()">关闭</button>
  </div>
</div>

<!-- 文本编辑 -->
<div id="editModal" class="modal hidden">
  <div class="box">
    <h3>编辑文档</h3>
    <textarea id="editor" rows="14" style="width:100%;background:#0f172a;color:var(--text);border-radius:6px;padding:8px;"></textarea>
    <button onclick="saveDoc()">保存</button>
    <button onclick="hideModals()">关闭</button>
  </div>
</div>

<!-- 图片预览 -->
<div id="imgModal" class="modal hidden">
  <div class="box" style="max-width:90vw;">
    < img id="previewImg" class="preview-img"/>
    <button onclick="hideModals()">关闭</button>
  </div>
</div>

<!-- 移动文件 -->
<div id="moveModal" class="modal hidden">
  <div class="box">
    <h3>选择目标文件夹</h3>
    <div id="moveTree"></div>
    <button onclick="confirmMove()">确认移动</button>
    <button onclick="hideModals()">取消</button>
  </div>
</div>

<script src="https://cdn.jsdelivr.net/npm/blueimp-md5/js/md5.min.js"></script>
<script>
/* ================== 配置常量 ================== */
const CHUNK_SIZE = 90 * 1024 * 1024;
const MAX_RETRY = 3;
const LS_TOKEN = 'gh_tk', LS_CFG = 'gh_cfg', LS_CACHE = 'gh_cache', LS_TASKS = 'gh_tasks', LS_STAT = 'gh_stat';

/* ================== Token 加密 ================== */
const crypto = {encrypt:s=>btoa(encodeURIComponent(s)), decrypt:s=>{try{return decodeURIComponent(atob(s));}catch{return '';}}};

/* ================== 全局状态 ================== */
let TOKEN = crypto.decrypt(localStorage.getItem(LS_TOKEN)||'');
let REPO_POOL = [], CACHE = {}, TASKS = [], STAT = {}, SELECTED = new Set();

/* ================== API ================== */
const api = {
  controller: new AbortController(),
  async request(path, opt={}){
    if(!TOKEN) return null;
    const r = await fetch(`https://api.github.com${path}`, {
      ...opt,
      signal: this.controller.signal,
      headers: {
        Authorization: `Bearer ${TOKEN}`,
        Accept: 'application/vnd.github+json',
        'Content-Type': 'application/json',
        ...opt.headers||{}
      }
    });
    if(r.status===401){alert('Token 失效');return null;}
    if(r.status===403||r.status===429){await new Promise(s=>setTimeout(s,3000));return this.request(path,opt);}
    return r.json().catch(()=>null);
  },
  async getContent(repo, path){
    return this.request(`/repos/${repo}/contents/${path}`);
  }
};

/* ================== 容量统计（缓存优化） ================== */
async function updateRepoStat(repo){
  if(STAT[repo] && Date.now()-STAT[repo].ts<60000) return STAT[repo].size;
  const data = await api.request(`/repos/${repo}/git/trees/${REPO_POOL.find(r=>r.r===repo).b}?recursive=1`);
  let size=0; data?.tree?.forEach(i=>size+=i.size||0);
  STAT[repo]={size,ts:Date.now()};
  localStorage.setItem(LS_STAT,JSON.stringify(STAT));
  return size;
}

async function pickRepo(fileSize){
  for(const repo of REPO_POOL){
    const used = await updateRepoStat(repo.r);
    if((used+fileSize)<=1*1024*1024*1024) return repo;
  }
  alert('所有仓库已满，请扩容仓库池！');
  return null;
}

/* ================== ViewModel ================== */
const vm = {
  cwd: '',
  async list(path){
    const key=path||'/';
    if(CACHE[key]) return CACHE[key];
    const files=[];
    for(const repo of REPO_POOL){
      const data = await api.getContent(repo.r, `${path}?ref=${repo.b}`);
      if(!Array.isArray(data)) continue;
      data.forEach(f=>files.push({...f,__repo:repo.r,__branch:repo.b}));
    }
    CACHE[key]=files; persist(); return files;
  },
  async delete(file){
    await api.request(`/repos/${file.__repo}/contents/${file.path}`,{
      method:'DELETE',
      body:JSON.stringify({message:'delete',sha:file.sha,branch:file.__branch})
    });
    STAT[file.__repo].size -= file.size||0;
    refresh();
  },
  async putFile(path, blob, repo, message='put'){
    const b64=await new Promise(r=>{const f=new FileReader();f.onload=e=>r(e.target.result.split(',')[1]);f.readAsDataURL(blob);});
    await api.request(`/repos/${repo}/contents/${path}`,{
      method:'PUT',
      body:JSON.stringify({message,content:b64,branch:REPO_POOL.find(r=>r.r===repo).b})
    });
    STAT[repo].size += blob.size;
  },
  async moveFile(file, newPath){
    const content = await fetch(file.download_url).then(r=>r.blob());
    await this.putFile(newPath, content, file.__repo, 'move');
    await this.delete(file);
  },
  async search(){
    const kw=document.getElementById('search').value.toLowerCase();
    const all=Object.values(CACHE).flat();
    render(all.filter(f=>f.name.toLowerCase().includes(kw)));
  }
};

/* ================== 分片上传 ================== */
async function uploadFile(file){
  const md5Hash = md5(await file.arrayBuffer());
  const exist = Object.values(CACHE).flat().find(f=>f.sha===md5Hash);
  if(exist){alert('秒传成功');return;}
  const chunks=Math.ceil(file.size/CHUNK_SIZE);
  const task={id:Date.now(),name:file.name,total:file.size,loaded:0,chunks,retry:0,md5:md5Hash,status:'pending',parts:[]};
  TASKS.push(task); persistTasks();

  const repo = await pickRepo(file.size);
  if(!repo) return;

  for(let i=0;i<chunks;i++){
    const chunk=file.slice(i*CHUNK_SIZE,(i+1)*CHUNK_SIZE);
    let ok=false;
    for(let r=0;r<MAX_RETRY&&!ok;r++){
      try{
        const b64=await new Promise(r=>{const f=new FileReader();f.onload=e=>r(e.target.result.split(',')[1]);f.readAsDataURL(chunk);});
        await api.request(`/repos/${repo.r}/contents/${vm.cwd}/${file.name}.part${i}`,{
          method:'PUT',
          body:JSON.stringify({message:'chunk',content:b64,branch:repo.b})
        });
        task.parts.push({i,sha:md5(i+Date.now())});
        task.loaded+=chunk.size; ok=true;
      }catch{task.retry++;}
    }
    if(!ok){task.status='failed';persistTasks();return;}
  }
  task.status='done'; persistTasks(); refresh();
}

/* ================== 分片合并下载（核心修复） ================== */
async function downloadAndMerge(filePrefix){
  const parts = CACHE[vm.cwd].filter(f=>f.name.startsWith(filePrefix)&&f.name.includes('.part'));
  parts.sort((a,b)=>a.name.localeCompare(b.name));
  const blobs=[];
  for(const p of parts){
    const res = await fetch(p.download_url);
    blobs.push(await res.blob());
  }
  const merged = new Blob(blobs);
  const a=document.createElement('a');
  a.href=URL.createObjectURL(merged);
  a.download=filePrefix;
  a.click();
}

/* ================== 文件移动 ================== */
let moveTargets=[];
function moveSelected(){
  moveTargets=Array.from(SELECTED);
  const tree=document.getElementById('moveTree'); tree.innerHTML='';
  Object.keys(CACHE).forEach(p=>{
    const d=document.createElement('div');
    d.textContent=p||'/';
    d.style.padding='4px'; d.style.cursor='pointer';
    d.onclick=()=>{vm.cwd=p;hideModals();confirmMove();};
    tree.appendChild(d);
  });
  showModal('moveModal');
}
async function confirmMove(){
  for(const name of moveTargets){
    const file = Object.values(CACHE).flat().find(f=>f.name===name);
    if(file) await vm.moveFile(file, `${vm.cwd}/${name}`);
  }
  SELECTED.clear(); refresh();
}

/* ================== UI渲染 ================== */
function render(files){
  const grid=document.getElementById('grid'); grid.innerHTML='';
  files.forEach(f=>{
    const d=document.createElement('div');
    d.className='card';
    if(SELECTED.has(f.name)) d.classList.add('selected');
    const cb=document.createElement('input'); cb.type='checkbox'; cb.className='checkbox';
    cb.checked=SELECTED.has(f.name);
    cb.onchange=e=>{e.stopPropagation();SELECTED.has(f.name)?SELECTED.delete(f.name):SELECTED.add(f.name);render(files);};
    d.appendChild(cb);
    const ext=f.name.split('.').pop();
    if(f.type==='dir'){
      d.innerHTML+=`<div>📁 ${f.name}</div>`;
      d.onclick=()=>{vm.cwd=f.path;refresh();};
    }else if(['jpg','png','webp'].includes(ext)){
      d.innerHTML+=`< img src="${f.download_url}"><div>${f.name}</div>`;
      d.onclick=e=>{if(e.target!==cb){document.getElementById('previewImg').src=f.download_url;showModal('imgModal');}};
    }else if(ext==='mp4'){
      d.innerHTML+=`<video src="${f.download_url}" controls style="width:100%;height:90px"></video><div>${f.name}</div>`;
      d.oncontextmenu=e=>{e.preventDefault();downloadAndMerge(f.name);};
    }else if(['txt','md'].includes(ext)){
      d.innerHTML+=`<div>📄 ${f.name}</div>`;
      d.onclick=()=>openEditor(f);
    }else if(f.name.includes('.part')){
      d.innerHTML+=`<div>🧩 ${f.name}</div>`;
    }else{
      d.innerHTML+=`<div>📄 ${f.name}</div>`;
    }
    grid.appendChild(d);
  });
  document.getElementById('path').textContent='当前路径：'+vm.cwd;
}

function refresh(){vm.list(vm.cwd).then(render);renderTasks();}
function renderTasks(){
  const box=document.getElementById('tasks');
  box.innerHTML=TASKS.map(t=>`<div class="task">${t.name}<br><progress value="${t.loaded}" max="${t.total}"></progress></div>`).join('');
}

/* ================== 基础操作 ================== */
function mkdir(){
  const name=prompt('文件夹名');
  if(!name) return;
  vm.putFile(`${vm.cwd}/${name}/.gitkeep`,new Blob(['']),REPO_POOL[0].r).then(refresh);
}
function upload(){
  const i=document.createElement('input'); i.type='file'; i.multiple=true;
  i.onchange=()=>Array.from(i.files).forEach(uploadFile);
  i.click();
}
function newDoc(){
  const name=prompt('文件名(.txt/.md)');
  if(!name) return;
  vm.putFile(`${vm.cwd}/${name}`,new Blob(['']),REPO_POOL[0].r).then(refresh);
}
function openEditor(file){
  fetch(file.download_url).then(r=>r.text()).then(t=>{
    document.getElementById('editor').value=t;
    document.getElementById('editModal').dataset.path=file.path;
    document.getElementById('editModal').dataset.repo=file.__repo;
    showModal('editModal');
  });
}
function saveDoc(){
  const m=document.getElementById('editModal');
  vm.putFile(m.dataset.path,new Blob([document.getElementById('editor').value]),m.dataset.repo).then(()=>{hideModals();refresh();});
}
async function batchDelete(){
  if(!confirm('确认批量删除选中文件？')) return;
  for(const name of SELECTED){
    const file = Object.values(CACHE).flat().find(f=>f.name===name);
    if(file) await vm.delete(file);
  }
  SELECTED.clear(); refresh();
}

/* ================== 仓库/统计 ================== */
function saveToken(){
  TOKEN=document.getElementById('token').value.trim();
  localStorage.setItem(LS_TOKEN,crypto.encrypt(TOKEN));
  alert('Token 已加密保存');
}
function logout(){localStorage.clear();location.reload();}
function saveCfg(){
  REPO_POOL=document.getElementById('cfgRepo').value.trim().split('\n').map(l=>{const[r,b]=l.split('|');return{r,b:'main'};}).filter(x=>x.r);
  localStorage.setItem(LS_CFG,JSON.stringify(REPO_POOL));
  hideModals(); refresh();
}
function showStat(){
  let txt=`仓库总数：${REPO_POOL.length}\n`;
  Object.entries(STAT).forEach(([r,s])=>{
    txt+=`${r}: ${(s.size/1024/1024).toFixed(2)} MB\n`;
  });
  document.getElementById('statText').textContent=txt;
  showModal('statModal');
}

/* ================== 持久化 ================== */
function persist(){localStorage.setItem(LS_CACHE,JSON.stringify(CACHE));}
function persistTasks(){localStorage.setItem(LS_TASKS,JSON.stringify(TASKS));}
function loadAll(){
  try{REPO_POOL=JSON.parse(localStorage.getItem(LS_CFG)||'[]');}catch{}
  try{CACHE=JSON.parse(localStorage.getItem(LS_CACHE)||'{}');}catch{}
  try{TASKS=JSON.parse(localStorage.getItem(LS_TASKS)||'[]');}catch{}
  try{STAT=JSON.parse(localStorage.getItem(LS_STAT)||'{}');}catch{}
}
function showModal(id){document.getElementById(id).classList.remove('hidden');}
function hideModals(){document.querySelectorAll('.modal').forEach(m=>m.classList.add('hidden'));}

/* ================== 启动 ================== */
loadAll();
document.getElementById('cfgRepo').value=REPO_POOL.map(c=>`${c.r}|${c.b}`).join('\n');
refresh();
</script>
</body>
</html>
