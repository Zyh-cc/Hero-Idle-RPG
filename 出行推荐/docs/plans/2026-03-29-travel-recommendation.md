# 出行推荐 App 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建单文件网页 App，根据位置、天气和出行意图，通过 AI 推荐附近目的地并生成推荐理由。

**Architecture:** 单个 `index.html` 文件，无后端。依次调用浏览器 Geolocation API → 和风天气 API → 高德地图 POI API → AI Chat API，将结果渲染为推荐卡片列表。API Key 存入 localStorage，首次打开自动弹出设置面板。

**Tech Stack:** 原生 HTML5 + CSS3 + Vanilla JavaScript，无框架无构建工具。调用：高德地图 REST API、和风天气 API v7、DeepSeek/Claude/Qwen/Gemini Chat API。

---

## 文件结构

```
出行推荐/
├── index.html     # 全部代码：HTML 结构 + CSS 样式 + JS 逻辑
├── design.md      # 设计文档
└── docs/plans/    # 本计划文档
```

所有 JS 逻辑按模块分段注释，顺序为：
1. 常量与配置
2. localStorage 工具函数
3. 设置面板
4. 位置模块
5. 天气模块
6. 意图面板
7. POI 搜索模块
8. AI 推荐模块
9. 结果渲染模块
10. 主流程控制

---

## API 参考

### 高德地图
- **地理编码**（城市名 → 经纬度）：
  `GET https://restapi.amap.com/v3/geocode/geo?address={城市名}&key={key}`
  响应：`data.geocodes[0].location` → `"118.796877,31.98513"`

- **POI 周边搜索**：
  `GET https://restapi.amap.com/v3/place/around?key={key}&location={lng,lat}&radius={radius}&types={types}&output=json&offset=20`
  响应：`data.pois[]` 每项含 `name`, `type`, `distance`, `address`

### 和风天气
- **城市查询**（获取 locationId）：
  `GET https://geoapi.qweather.com/v2/city/lookup?location={城市名或经纬度}&key={key}`
  响应：`data.location[0].id`

- **实时天气**：
  `GET https://devapi.qweather.com/v7/weather/now?location={locationId}&key={key}`
  响应：`data.now` 含 `text`（天气描述）、`temp`、`windDir`、`windScale`、`humidity`

### AI 提供商

| 提供商 | Endpoint | Model | 特殊处理 |
|--------|----------|-------|---------|
| DeepSeek | `https://api.deepseek.com/v1/chat/completions` | `deepseek-chat` | 标准 OpenAI 格式 |
| Anthropic | `https://api.anthropic.com/v1/messages` | `claude-haiku-4-5-20251001` | 需加 `anthropic-version: 2023-06-01` header，响应在 `content[0].text` |
| 通义千问 | `https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions` | `qwen-turbo` | 标准 OpenAI 格式 |
| Gemini | `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key={key}` | — | body 格式不同，见 Task 7 |

### 活动标签 → 高德 POI 类型码

| 标签 | 高德 types 参数 |
|------|----------------|
| 🌿 自然风景 | `110000` |
| 🏛 历史文化 | `110000\|141200` |
| 🍜 美食 | `050000` |
| 🛍 购物 | `060000` |
| ☕ 休闲 | `080000\|110000` |
| （未选择时默认） | `110000\|080000` |

---

## Task 1: HTML 骨架 + 全局 CSS

**Files:**
- Create: `出行推荐/index.html`

- [ ] **Step 1: 创建 index.html，写入 HTML 结构和 CSS**

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>出行推荐</title>
  <style>
    :root {
      --bg: #1a1a2e;
      --surface: #1e1e2e;
      --surface2: #2a2a3e;
      --border: #3a3a5c;
      --accent: #6366f1;
      --accent2: #8b5cf6;
      --cyan: #06b6d4;
      --text: #e2e8f0;
      --text-muted: #64748b;
      --text-dim: #94a3b8;
      --radius: 14px;
    }
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
      background: var(--bg);
      color: var(--text);
      min-height: 100vh;
    }

    /* 顶部导航 */
    nav {
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 16px 24px;
      background: var(--surface);
      border-bottom: 1px solid var(--border);
    }
    nav h1 { font-size: 1.1em; font-weight: 700; }
    #settings-btn {
      background: var(--surface2);
      border: 1px solid var(--border);
      color: var(--text-dim);
      padding: 7px 14px;
      border-radius: 8px;
      cursor: pointer;
      font-size: 0.85em;
    }
    #settings-btn:hover { border-color: var(--accent); color: var(--text); }

    /* 主布局：三列 */
    main {
      display: grid;
      grid-template-columns: 1fr 1.1fr 1.3fr;
      gap: 28px;
      padding: 28px 24px;
      max-width: 1200px;
      margin: 0 auto;
    }
    @media (max-width: 768px) {
      main { grid-template-columns: 1fr; }
    }

    /* 步骤卡片 */
    .step-card {
      border-radius: 18px;
      overflow: hidden;
      border: 2px solid var(--border);
    }
    .step-header {
      padding: 18px 24px;
      font-weight: 700;
      font-size: 1.05em;
    }
    .step-body {
      padding: 24px;
      background: var(--surface);
      display: flex;
      flex-direction: column;
      gap: 20px;
    }
    .step-card:nth-child(1) { border-color: var(--accent); }
    .step-card:nth-child(1) .step-header { background: var(--accent); }
    .step-card:nth-child(2) { border-color: var(--accent2); }
    .step-card:nth-child(2) .step-header { background: var(--accent2); }
    .step-card:nth-child(3) { border-color: var(--cyan); }
    .step-card:nth-child(3) .step-header { background: var(--cyan); color: #0f172a; }

    /* 内部面板 */
    .panel {
      background: var(--surface2);
      border-radius: var(--radius);
      padding: 18px;
    }
    .panel-label {
      font-size: 0.75em;
      color: var(--text-muted);
      text-transform: uppercase;
      letter-spacing: 0.08em;
      margin-bottom: 10px;
    }

    /* 按钮 */
    .btn {
      border: none;
      border-radius: 10px;
      padding: 10px 16px;
      font-size: 0.9em;
      cursor: pointer;
      font-weight: 500;
      transition: opacity 0.15s;
    }
    .btn:hover { opacity: 0.85; }
    .btn-primary { background: var(--accent); color: white; width: 100%; }
    .btn-ghost {
      background: var(--surface2);
      border: 1px solid var(--border);
      color: var(--text-dim);
    }

    /* 输入框 */
    input[type="text"], input[type="password"], select {
      background: #22223a;
      border: 1px solid var(--border);
      border-radius: 10px;
      padding: 10px 14px;
      font-size: 0.88em;
      color: var(--text);
      width: 100%;
    }
    input:focus, select:focus {
      outline: none;
      border-color: var(--accent);
    }

    /* 标签 */
    .tag-group { display: flex; flex-wrap: wrap; gap: 8px; }
    .tag {
      background: var(--surface2);
      border: 1px solid var(--border);
      color: var(--text-muted);
      border-radius: 22px;
      padding: 7px 16px;
      font-size: 0.88em;
      cursor: pointer;
      transition: all 0.15s;
      user-select: none;
    }
    .tag.active {
      background: var(--accent2);
      border-color: var(--accent2);
      color: white;
      font-weight: 500;
    }

    /* 推荐卡片 */
    .result-card {
      border-radius: var(--radius);
      padding: 20px;
      border: 1px solid transparent;
    }
    .result-card-title {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-bottom: 6px;
    }
    .result-card-name { font-weight: 700; font-size: 1em; }
    .result-card-dist {
      border-radius: 8px;
      padding: 4px 10px;
      font-size: 0.78em;
      font-weight: 600;
    }
    .result-card-type { font-size: 0.8em; color: var(--text-muted); margin-bottom: 10px; }
    .result-card-reason { font-size: 0.86em; line-height: 1.6; }

    /* 结果卡片颜色循环 */
    .result-card:nth-child(3n+1) {
      background: #162532; border-color: #1e4a5e;
    }
    .result-card:nth-child(3n+1) .result-card-dist { background: #1e4a5e; color: var(--cyan); }
    .result-card:nth-child(3n+1) .result-card-reason { color: #7dd3c8; }
    .result-card:nth-child(3n+2) {
      background: #1e1a30; border-color: #3b2e6e;
    }
    .result-card:nth-child(3n+2) .result-card-dist { background: #2d1f5e; color: #a78bfa; }
    .result-card:nth-child(3n+2) .result-card-reason { color: #c4b5fd; }
    .result-card:nth-child(3n) {
      background: #1e2a1a; border-color: #3a5e2e;
    }
    .result-card:nth-child(3n) .result-card-dist { background: #2a3e1e; color: #86efac; }
    .result-card:nth-child(3n) .result-card-reason { color: #bbf7d0; }

    /* 状态提示 */
    .status-text { font-size: 0.8em; margin-top: 8px; }
    .status-success { color: #7c7cff; }
    .status-error { color: #f87171; }
    .status-info { color: var(--text-muted); }

    /* 加载动画 */
    .loading { text-align: center; padding: 24px; color: var(--text-muted); font-size: 0.9em; }
    .loading::after {
      content: '';
      display: inline-block;
      width: 16px; height: 16px;
      border: 2px solid var(--border);
      border-top-color: var(--accent);
      border-radius: 50%;
      animation: spin 0.8s linear infinite;
      margin-left: 8px;
      vertical-align: middle;
    }
    @keyframes spin { to { transform: rotate(360deg); } }

    /* 设置弹窗 */
    .modal-overlay {
      position: fixed; inset: 0;
      background: rgba(0,0,0,0.7);
      display: flex; align-items: center; justify-content: center;
      z-index: 100;
    }
    .modal-overlay.hidden { display: none; }
    .modal {
      background: var(--surface);
      border: 1px solid var(--border);
      border-radius: 20px;
      padding: 32px;
      width: 480px;
      max-width: 95vw;
      max-height: 90vh;
      overflow-y: auto;
    }
    .modal h2 { margin-bottom: 24px; font-size: 1.1em; }
    .form-group { margin-bottom: 18px; }
    .form-group label {
      display: block;
      font-size: 0.82em;
      color: var(--text-dim);
      margin-bottom: 6px;
    }
    .modal-footer {
      display: flex;
      gap: 10px;
      margin-top: 24px;
    }
    .btn-save { background: var(--accent); color: white; flex: 1; }
  </style>
</head>
<body>

  <nav>
    <h1>🗺 出行推荐</h1>
    <button id="settings-btn" onclick="openSettings()">⚙ 设置</button>
  </nav>

  <main>
    <!-- 第 1 步：位置 & 天气 -->
    <div class="step-card">
      <div class="step-header">第 1 步 · 位置 &amp; 天气</div>
      <div class="step-body">
        <div class="panel">
          <div class="panel-label">📍 当前位置</div>
          <div style="display:flex;flex-direction:column;gap:8px;">
            <button class="btn btn-ghost" onclick="getGPS()">🎯 自动定位</button>
            <div style="text-align:center;font-size:0.78em;color:var(--text-muted);">— 或手动输入 —</div>
            <div style="display:flex;gap:8px;">
              <input type="text" id="city-input" placeholder="如：南京市江宁区" />
              <button class="btn btn-ghost" style="white-space:nowrap;" onclick="geocodeCity()">确定</button>
            </div>
          </div>
          <div id="location-status" class="status-text status-info"></div>
        </div>
        <div class="panel">
          <div class="panel-label">🌤 今日天气</div>
          <div id="weather-display">
            <button class="btn btn-ghost" onclick="fetchWeather()" style="width:100%;">获取天气</button>
          </div>
          <div id="weather-status" class="status-text status-info"></div>
          <div id="manual-weather-toggle" style="margin-top:10px;font-size:0.8em;color:var(--accent);cursor:pointer;display:none;" onclick="toggleManualWeather()">⚙ 手动设置天气</div>
          <div id="manual-weather-panel" style="display:none;margin-top:10px;">
            <select id="manual-weather-text">
              <option value="晴">☀️ 晴</option>
              <option value="多云">⛅ 多云</option>
              <option value="阴">☁️ 阴</option>
              <option value="小雨">🌧 小雨</option>
              <option value="大雨">⛈ 大雨</option>
              <option value="雪">❄️ 雪</option>
            </select>
            <div style="display:flex;gap:8px;margin-top:8px;">
              <input type="text" id="manual-temp" placeholder="温度 如 22" style="width:80px;" />
              <span style="align-self:center;color:var(--text-muted);">°C</span>
              <button class="btn btn-ghost" onclick="applyManualWeather()">确定</button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 第 2 步：出行想法 -->
    <div class="step-card">
      <div class="step-header">第 2 步 · 出行想法</div>
      <div class="step-body">
        <div>
          <div class="panel-label">🚗 出行方式</div>
          <div class="tag-group" id="transport-tags">
            <div class="tag" data-value="开车" onclick="selectTransport(this)">🚗 开车</div>
            <div class="tag" data-value="步行" onclick="selectTransport(this)">🚶 步行</div>
            <div class="tag" data-value="公交" onclick="selectTransport(this)">🚌 公交</div>
            <div class="tag" data-value="骑行" onclick="selectTransport(this)">🚲 骑行</div>
          </div>
        </div>
        <div>
          <div class="panel-label">🎯 活动类型（可多选）</div>
          <div class="tag-group" id="activity-tags">
            <div class="tag" data-value="自然风景" data-types="110000" onclick="toggleActivity(this)">🌿 自然风景</div>
            <div class="tag" data-value="历史文化" data-types="110000|141200" onclick="toggleActivity(this)">🏛 历史文化</div>
            <div class="tag" data-value="美食" data-types="050000" onclick="toggleActivity(this)">🍜 美食</div>
            <div class="tag" data-value="购物" data-types="060000" onclick="toggleActivity(this)">🛍 购物</div>
            <div class="tag" data-value="休闲" data-types="080000|110000" onclick="toggleActivity(this)">☕ 休闲</div>
          </div>
        </div>
        <div>
          <div class="panel-label">💬 补充说明（可选）</div>
          <input type="text" id="user-text" placeholder="如：想找个安静的地方，不要人太多..." />
        </div>
        <button class="btn btn-primary" onclick="getRecommendations()" id="recommend-btn">
          ✨ 获取 AI 推荐
        </button>
      </div>
    </div>

    <!-- 第 3 步：推荐结果 -->
    <div class="step-card">
      <div class="step-header">第 3 步 · 推荐结果</div>
      <div class="step-body" id="results-container">
        <div style="text-align:center;padding:40px 0;color:var(--text-muted);font-size:0.9em;">
          完成左侧设置后点击「获取 AI 推荐」
        </div>
      </div>
    </div>
  </main>

  <!-- 设置弹窗 -->
  <div class="modal-overlay hidden" id="settings-modal">
    <div class="modal">
      <h2>⚙ 设置</h2>
      <div class="form-group">
        <label>AI 提供商</label>
        <select id="ai-provider">
          <option value="deepseek">DeepSeek（推荐，国内直连）</option>
          <option value="anthropic">Anthropic Claude</option>
          <option value="qwen">通义千问</option>
          <option value="gemini">Google Gemini</option>
        </select>
      </div>
      <div class="form-group">
        <label>AI API Key</label>
        <input type="password" id="ai-key" placeholder="sk-..." />
      </div>
      <div class="form-group">
        <label>高德地图 API Key</label>
        <input type="password" id="amap-key" placeholder="高德开放平台 Web服务 Key" />
      </div>
      <div class="form-group">
        <label>和风天气 API Key</label>
        <input type="password" id="qweather-key" placeholder="和风天气开发版 Key" />
      </div>
      <div class="form-group">
        <label>搜索半径（km，默认 20）</label>
        <input type="text" id="search-radius" placeholder="20" />
      </div>
      <div class="modal-footer">
        <button class="btn btn-ghost" onclick="closeSettings()">取消</button>
        <button class="btn btn-save" onclick="saveSettings()">保存</button>
      </div>
    </div>
  </div>

  <script>
  // ============================================================
  // 1. 全局状态
  // ============================================================
  const state = {
    location: null,    // { lng, lat, name }
    weather: null,     // { text, temp, windDir, windScale, humidity }
    transport: null,   // string
    activities: [],    // string[]
    poiTypes: '',      // 高德 types 参数
  };
  </script>
</body>
</html>
```

- [ ] **Step 2: 在浏览器中打开验证**

打开 `出行推荐/index.html`，检查：
- 顶部导航栏显示"🗺 出行推荐"和"⚙ 设置"按钮
- 三列布局正常，各有彩色标题栏
- 标签可点击（暂无逻辑）
- 设置弹窗尚未出现（JS 还未写）

- [ ] **Step 3: 提交**

```bash
cd "E:/小玩意"
git add 出行推荐/index.html
git commit -m "feat: 出行推荐 - HTML骨架与全局CSS"
```

---

## Task 2: localStorage 工具 + 设置面板逻辑

**Files:**
- Modify: `出行推荐/index.html`（在 `<script>` 中追加）

- [ ] **Step 1: 追加 localStorage 工具函数和设置面板逻辑**

在 `// 1. 全局状态` 区块后追加：

```javascript
// ============================================================
// 2. localStorage 工具
// ============================================================
const KEYS = {
  aiProvider: 'tr_ai_provider',
  aiKey: 'tr_ai_key',
  amapKey: 'tr_amap_key',
  qweatherKey: 'tr_qweather_key',
  searchRadius: 'tr_search_radius',
  initialized: 'tr_initialized',
};

function getSetting(key, fallback = '') {
  return localStorage.getItem(key) ?? fallback;
}
function setSetting(key, value) {
  localStorage.setItem(key, value);
}

// ============================================================
// 3. 设置面板
// ============================================================
function openSettings() {
  document.getElementById('ai-provider').value = getSetting(KEYS.aiProvider, 'deepseek');
  document.getElementById('ai-key').value = getSetting(KEYS.aiKey);
  document.getElementById('amap-key').value = getSetting(KEYS.amapKey);
  document.getElementById('qweather-key').value = getSetting(KEYS.qweatherKey);
  document.getElementById('search-radius').value = getSetting(KEYS.searchRadius, '20');
  document.getElementById('settings-modal').classList.remove('hidden');
}

function closeSettings() {
  document.getElementById('settings-modal').classList.add('hidden');
}

function saveSettings() {
  setSetting(KEYS.aiProvider, document.getElementById('ai-provider').value);
  setSetting(KEYS.aiKey, document.getElementById('ai-key').value.trim());
  setSetting(KEYS.amapKey, document.getElementById('amap-key').value.trim());
  setSetting(KEYS.qweatherKey, document.getElementById('qweather-key').value.trim());
  setSetting(KEYS.searchRadius, document.getElementById('search-radius').value || '20');
  setSetting(KEYS.initialized, '1');
  closeSettings();
}

// 首次打开自动弹出设置面板
window.addEventListener('DOMContentLoaded', () => {
  if (!getSetting(KEYS.initialized)) {
    openSettings();
  }
});

// 点击遮罩关闭
document.getElementById('settings-modal').addEventListener('click', (e) => {
  if (e.target === e.currentTarget) closeSettings();
});
```

- [ ] **Step 2: 验证**

刷新页面（清除 localStorage 后）：
- 首次打开自动弹出设置面板
- 填写并保存后关闭，再刷新不再弹出
- 点击"⚙ 设置"重新打开，之前填写的值仍在

清除 localStorage 方法：浏览器开发者工具 → Application → Local Storage → Clear All。

- [ ] **Step 3: 提交**

```bash
git add 出行推荐/index.html
git commit -m "feat: 出行推荐 - 设置面板与localStorage"
```

---

## Task 3: 位置模块（GPS + 手动输入 + 地理编码）

**Files:**
- Modify: `出行推荐/index.html`

- [ ] **Step 1: 追加位置模块**

```javascript
// ============================================================
// 4. 位置模块
// ============================================================
function setLocationStatus(msg, type = 'info') {
  const el = document.getElementById('location-status');
  el.textContent = msg;
  el.className = `status-text status-${type}`;
}

function setLocation(lng, lat, name) {
  state.location = { lng, lat, name };
  setLocationStatus(`✓ 已定位：${name}`, 'success');
  // 定位成功后自动获取天气
  fetchWeather();
}

async function getGPS() {
  if (!navigator.geolocation) {
    setLocationStatus('浏览器不支持定位，请手动输入', 'error');
    return;
  }
  setLocationStatus('定位中...', 'info');
  navigator.geolocation.getCurrentPosition(
    async (pos) => {
      const { longitude: lng, latitude: lat } = pos.coords;
      // 反地理编码获取城市名
      const amapKey = getSetting(KEYS.amapKey);
      if (!amapKey) {
        setLocation(lng, lat, `${lat.toFixed(4)},${lng.toFixed(4)}`);
        return;
      }
      try {
        const res = await fetch(
          `https://restapi.amap.com/v3/geocode/regeo?location=${lng},${lat}&key=${amapKey}&output=json`
        );
        const data = await res.json();
        const name = data.regeocode?.addressComponent?.district
          || data.regeocode?.formatted_address
          || `${lat.toFixed(4)},${lng.toFixed(4)}`;
        setLocation(lng, lat, name);
      } catch {
        setLocation(lng, lat, `${lat.toFixed(4)},${lng.toFixed(4)}`);
      }
    },
    (err) => {
      setLocationStatus('定位失败，请手动输入城市名', 'error');
    }
  );
}

async function geocodeCity() {
  const city = document.getElementById('city-input').value.trim();
  if (!city) return;
  const amapKey = getSetting(KEYS.amapKey);
  if (!amapKey) {
    setLocationStatus('请先在设置中填写高德 API Key', 'error');
    openSettings();
    return;
  }
  setLocationStatus('查询中...', 'info');
  try {
    const res = await fetch(
      `https://restapi.amap.com/v3/geocode/geo?address=${encodeURIComponent(city)}&key=${amapKey}&output=json`
    );
    const data = await res.json();
    if (data.geocodes && data.geocodes.length > 0) {
      const [lng, lat] = data.geocodes[0].location.split(',').map(Number);
      setLocation(lng, lat, city);
    } else {
      setLocationStatus('未找到该地址，请换个写法', 'error');
    }
  } catch {
    setLocationStatus('地理编码请求失败，请检查网络', 'error');
  }
}
```

- [ ] **Step 2: 验证**

在浏览器中：
1. 点击"🎯 自动定位"→ 允许定位权限，应显示当前城市名
2. 在输入框输入"南京市江宁区"点确定，应显示"✓ 已定位：南京市江宁区"
3. 无高德 Key 时，输入城市名应提示打开设置

- [ ] **Step 3: 提交**

```bash
git add 出行推荐/index.html
git commit -m "feat: 出行推荐 - 位置模块（GPS+手动+地理编码）"
```

---

## Task 4: 天气模块（和风天气 API + 手动选择）

**Files:**
- Modify: `出行推荐/index.html`

- [ ] **Step 1: 追加天气模块**

```javascript
// ============================================================
// 5. 天气模块
// ============================================================
// 和风天气图标映射
const WEATHER_ICONS = {
  '晴': '☀️', '多云': '⛅', '阴': '☁️',
  '小雨': '🌧', '中雨': '🌧', '大雨': '⛈', '暴雨': '⛈',
  '雷阵雨': '⛈', '小雪': '❄️', '中雪': '❄️', '大雪': '❄️',
  '雾': '🌫', '霾': '🌫',
};

function getWeatherIcon(text) {
  for (const [k, v] of Object.entries(WEATHER_ICONS)) {
    if (text.includes(k)) return v;
  }
  return '🌡';
}

function renderWeather(w) {
  const icon = getWeatherIcon(w.text);
  document.getElementById('weather-display').innerHTML = `
    <div style="display:flex;align-items:center;gap:16px;">
      <div style="font-size:3em;line-height:1;">${icon}</div>
      <div>
        <div style="color:#fbbf24;font-weight:700;font-size:1.1em;">${w.text} · ${w.temp}°C</div>
        <div style="font-size:0.78em;color:var(--text-muted);margin-top:4px;">
          湿度 ${w.humidity}% · ${w.windDir} ${w.windScale}级
        </div>
      </div>
    </div>
  `;
  document.getElementById('manual-weather-toggle').style.display = 'block';
}

function setWeatherStatus(msg, type = 'info') {
  const el = document.getElementById('weather-status');
  el.textContent = msg;
  el.className = `status-text status-${type}`;
}

async function fetchWeather() {
  const qKey = getSetting(KEYS.qweatherKey);
  if (!qKey) {
    setWeatherStatus('未填写和风天气 Key，请手动设置天气', 'error');
    document.getElementById('manual-weather-toggle').style.display = 'block';
    return;
  }
  if (!state.location) {
    setWeatherStatus('请先获取位置', 'error');
    return;
  }
  setWeatherStatus('获取天气中...', 'info');
  try {
    // 步骤1：查 locationId
    const loc = `${state.location.lng},${state.location.lat}`;
    const geoRes = await fetch(
      `https://geoapi.qweather.com/v2/city/lookup?location=${encodeURIComponent(loc)}&key=${qKey}`
    );
    const geoData = await geoRes.json();
    if (!geoData.location || geoData.location.length === 0) throw new Error('城市未找到');
    const locationId = geoData.location[0].id;

    // 步骤2：查实时天气
    const wxRes = await fetch(
      `https://devapi.qweather.com/v7/weather/now?location=${locationId}&key=${qKey}`
    );
    const wxData = await wxRes.json();
    if (wxData.code !== '200') throw new Error('天气 API 错误');
    const now = wxData.now;
    state.weather = {
      text: now.text,
      temp: now.temp,
      windDir: now.windDir,
      windScale: now.windScale,
      humidity: now.humidity,
    };
    renderWeather(state.weather);
    setWeatherStatus('', 'info');
  } catch (e) {
    setWeatherStatus(`天气获取失败 (${e.message})，请手动设置`, 'error');
    document.getElementById('manual-weather-toggle').style.display = 'block';
  }
}

function toggleManualWeather() {
  const panel = document.getElementById('manual-weather-panel');
  panel.style.display = panel.style.display === 'none' ? 'block' : 'none';
}

function applyManualWeather() {
  const text = document.getElementById('manual-weather-text').value;
  const temp = document.getElementById('manual-temp').value || '20';
  state.weather = { text, temp, windDir: '微风', windScale: '1-2', humidity: '—' };
  renderWeather(state.weather);
  document.getElementById('manual-weather-panel').style.display = 'none';
  setWeatherStatus('已手动设置天气', 'success');
}
```

- [ ] **Step 2: 验证**

1. 有和风 Key 时：定位后应自动获取天气，显示图标 + 温度 + 湿度
2. 无 Key 时：提示手动设置，展开下拉选择器后点确定，天气显示更新
3. 点击"⚙ 手动设置天气"可切换展开/收起

- [ ] **Step 3: 提交**

```bash
git add 出行推荐/index.html
git commit -m "feat: 出行推荐 - 天气模块（和风API+手动选择）"
```

---

## Task 5: 意图面板（标签交互 + 状态管理）

**Files:**
- Modify: `出行推荐/index.html`

- [ ] **Step 1: 追加意图面板逻辑**

```javascript
// ============================================================
// 6. 意图面板
// ============================================================
function selectTransport(el) {
  document.querySelectorAll('#transport-tags .tag').forEach(t => t.classList.remove('active'));
  el.classList.add('active');
  state.transport = el.dataset.value;
}

function toggleActivity(el) {
  el.classList.toggle('active');
  const active = [...document.querySelectorAll('#activity-tags .tag.active')];
  state.activities = active.map(t => t.dataset.value);
  // 合并所有选中标签的 POI 类型码（去重）
  const types = new Set();
  active.forEach(t => t.dataset.types.split('|').forEach(c => types.add(c)));
  state.poiTypes = types.size > 0 ? [...types].join('|') : '110000|080000';
}
```

- [ ] **Step 2: 验证**

1. 点击出行方式标签：只有一个高亮（单选）
2. 点击活动类型标签：可多选，再点取消
3. 打开浏览器控制台，执行 `console.log(state)` 确认 transport、activities、poiTypes 字段正确更新

- [ ] **Step 3: 提交**

```bash
git add 出行推荐/index.html
git commit -m "feat: 出行推荐 - 意图面板标签交互"
```

---

## Task 6: 高德 POI 周边搜索

**Files:**
- Modify: `出行推荐/index.html`

- [ ] **Step 1: 追加 POI 搜索模块**

```javascript
// ============================================================
// 7. POI 搜索模块
// ============================================================
async function searchPOI(radiusKm) {
  const amapKey = getSetting(KEYS.amapKey);
  if (!amapKey) throw new Error('请先填写高德 API Key');
  const { lng, lat } = state.location;
  const radius = radiusKm * 1000; // 转米
  const types = state.poiTypes || '110000|080000';

  const res = await fetch(
    `https://restapi.amap.com/v3/place/around?key=${amapKey}` +
    `&location=${lng},${lat}&radius=${radius}&types=${encodeURIComponent(types)}` +
    `&output=json&offset=20`
  );
  const data = await res.json();
  if (data.status !== '1') throw new Error(`高德 POI 搜索失败: ${data.info}`);
  return data.pois || [];
}

function formatPOIList(pois) {
  return pois.slice(0, 20).map((p, i) => {
    const distKm = (Number(p.distance) / 1000).toFixed(1);
    return `${i + 1}. ${p.name} | ${p.type.split(';')[0]} | ${distKm}km`;
  }).join('\n');
}
```

- [ ] **Step 2: 验证**

在浏览器控制台手动测试（需先设置好高德 Key 并获取位置）：

```javascript
searchPOI(20).then(pois => {
  console.log('POI 数量:', pois.length);
  console.log('前3条:', pois.slice(0, 3).map(p => p.name));
  console.log('格式化:\n', formatPOIList(pois));
});
```

预期：输出若干地点名称，formatPOIList 返回"1. 牛首山 | 风景名胜 | 12.3km"格式的字符串。

- [ ] **Step 3: 提交**

```bash
git add 出行推荐/index.html
git commit -m "feat: 出行推荐 - 高德POI周边搜索"
```

---

## Task 7: AI 推荐模块（多 Provider）

**Files:**
- Modify: `出行推荐/index.html`

- [ ] **Step 1: 追加 AI 推荐模块**

```javascript
// ============================================================
// 8. AI 推荐模块
// ============================================================
function buildPrompt(poiList) {
  const w = state.weather;
  const transport = state.transport || '不限';
  const activities = state.activities.length > 0 ? state.activities.join('、') : '不限';
  const userText = document.getElementById('user-text').value.trim();
  return `你是一个出行推荐助手。根据以下信息，从候选地点列表中挑选最合适的3-5个，并为每个生成一句推荐理由（30字以内，结合天气和用户意图）。

当前信息：
- 位置：${state.location.name}
- 天气：${w.text}，${w.temp}°C，${w.windDir}${w.windScale}级
- 出行方式：${transport}
- 活动偏好：${activities}
- 补充说明：${userText || '无'}

候选地点（高德POI搜索结果，格式：序号. 名称 | 类别 | 距离km）：
${poiList}

请返回 JSON 格式，不要有其他文字：
[{"name": "地点名", "reason": "推荐理由", "rank": 1}, ...]`;
}

async function callAI(prompt) {
  const provider = getSetting(KEYS.aiProvider, 'deepseek');
  const apiKey = getSetting(KEYS.aiKey);
  if (!apiKey) throw new Error('请先在设置中填写 AI API Key');

  if (provider === 'anthropic') {
    const res = await fetch('https://api.anthropic.com/v1/messages', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'x-api-key': apiKey,
        'anthropic-version': '2023-06-01',
      },
      body: JSON.stringify({
        model: 'claude-haiku-4-5-20251001',
        max_tokens: 1024,
        messages: [{ role: 'user', content: prompt }],
      }),
    });
    const data = await res.json();
    if (data.error) throw new Error(data.error.message);
    return data.content[0].text;
  }

  if (provider === 'gemini') {
    const res = await fetch(
      `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${apiKey}`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          contents: [{ parts: [{ text: prompt }] }],
        }),
      }
    );
    const data = await res.json();
    if (data.error) throw new Error(data.error.message);
    return data.candidates[0].content.parts[0].text;
  }

  // DeepSeek 和 通义千问 均为 OpenAI 兼容格式
  const endpoints = {
    deepseek: 'https://api.deepseek.com/v1/chat/completions',
    qwen: 'https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions',
  };
  const models = { deepseek: 'deepseek-chat', qwen: 'qwen-turbo' };
  const res = await fetch(endpoints[provider], {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${apiKey}` },
    body: JSON.stringify({
      model: models[provider],
      messages: [{ role: 'user', content: prompt }],
      temperature: 0.7,
    }),
  });
  const data = await res.json();
  if (data.error) throw new Error(data.error.message);
  return data.choices[0].message.content;
}

function parseAIResponse(text) {
  // 从 AI 响应中提取 JSON（可能包含 markdown 代码块）
  const match = text.match(/\[[\s\S]*\]/);
  if (!match) throw new Error('AI 返回格式异常');
  return JSON.parse(match[0]);
}
```

- [ ] **Step 2: 验证**

在浏览器控制台手动测试：

```javascript
const testPrompt = buildPrompt('1. 牛首山 | 风景名胜 | 12.0km\n2. 将军山公园 | 公园 | 8.5km\n3. 方山 | 风景名胜 | 15.2km');
console.log('Prompt 预览:\n', testPrompt.substring(0, 200));
callAI(testPrompt).then(raw => {
  console.log('AI 原始响应:\n', raw);
  console.log('解析结果:', parseAIResponse(raw));
});
```

预期：`parseAIResponse` 返回包含 `name`、`reason`、`rank` 字段的数组。

- [ ] **Step 3: 提交**

```bash
git add 出行推荐/index.html
git commit -m "feat: 出行推荐 - AI推荐模块（DeepSeek/Claude/Qwen/Gemini）"
```

---

## Task 8: 结果渲染 + 主流程 + 错误降级

**Files:**
- Modify: `出行推荐/index.html`

- [ ] **Step 1: 追加结果渲染和主流程**

```javascript
// ============================================================
// 9. 结果渲染
// ============================================================
function renderResults(recommendations, pois) {
  const container = document.getElementById('results-container');
  if (recommendations.length === 0) {
    container.innerHTML = '<div style="text-align:center;padding:40px;color:var(--text-muted);">未获取到推荐，请尝试扩大搜索范围</div>';
    return;
  }
  // 将 AI 推荐结果与 POI 原始数据合并（通过名称匹配距离）
  const poiMap = {};
  pois.forEach(p => { poiMap[p.name] = (Number(p.distance) / 1000).toFixed(1); });

  container.innerHTML = recommendations
    .sort((a, b) => a.rank - b.rank)
    .map(item => {
      const dist = poiMap[item.name] ? `${poiMap[item.name]} km` : '—';
      return `
        <div class="result-card">
          <div class="result-card-title">
            <div class="result-card-name">${item.name}</div>
            <div class="result-card-dist">${dist}</div>
          </div>
          <div class="result-card-type">📍 ${item.reason ? '推荐' : '附近地点'}</div>
          <div class="result-card-reason">${item.reason || ''}</div>
        </div>
      `;
    }).join('');
}

function renderFallbackResults(pois) {
  const container = document.getElementById('results-container');
  container.innerHTML = `
    <div style="background:#2a1a1a;border:1px solid #7f1d1d;border-radius:10px;padding:14px;margin-bottom:16px;font-size:0.82em;color:#fca5a5;">
      ⚠ AI 推荐暂不可用，以下为附近地点列表
    </div>
  ` + pois.slice(0, 5).map(p => `
    <div class="result-card">
      <div class="result-card-title">
        <div class="result-card-name">${p.name}</div>
        <div class="result-card-dist">${(Number(p.distance)/1000).toFixed(1)} km</div>
      </div>
      <div class="result-card-type">📍 ${p.type.split(';')[0]}</div>
    </div>
  `).join('');
}

// ============================================================
// 10. 主流程
// ============================================================
async function getRecommendations() {
  // 前置检查
  if (!state.location) {
    alert('请先获取位置（自动定位或手动输入）');
    return;
  }
  if (!state.weather) {
    alert('请先获取天气（点击"获取天气"或手动设置）');
    return;
  }

  const btn = document.getElementById('recommend-btn');
  const container = document.getElementById('results-container');
  btn.disabled = true;
  btn.textContent = '推荐中...';
  container.innerHTML = '<div class="loading">正在搜索附近地点</div>';

  const radius = Number(getSetting(KEYS.searchRadius, '20'));

  try {
    // Step 1: POI 搜索
    let pois;
    try {
      pois = await searchPOI(radius);
    } catch (e) {
      container.innerHTML = `
        <div style="text-align:center;padding:30px;color:var(--text-muted);">
          <div style="color:#f87171;margin-bottom:12px;">${e.message}</div>
          <button class="btn btn-ghost" onclick="getRecommendations()">重试</button>
        </div>`;
      return;
    }

    if (pois.length === 0) {
      container.innerHTML = `
        <div style="text-align:center;padding:30px;color:var(--text-muted);">
          <div style="margin-bottom:12px;">附近 ${radius}km 内未找到相关地点</div>
          <button class="btn btn-ghost" onclick="expandSearch()">扩大至 50km 搜索</button>
        </div>`;
      return;
    }

    // Step 2: AI 推荐
    container.innerHTML = '<div class="loading">AI 正在分析推荐</div>';
    const poiList = formatPOIList(pois);
    const prompt = buildPrompt(poiList);

    let recommendations;
    try {
      const raw = await callAI(prompt);
      recommendations = parseAIResponse(raw);
    } catch (e) {
      // AI 失败降级：直接展示 POI 列表
      renderFallbackResults(pois);
      return;
    }

    renderResults(recommendations, pois);

  } finally {
    btn.disabled = false;
    btn.textContent = '✨ 获取 AI 推荐';
  }
}

async function expandSearch() {
  const oldRadius = getSetting(KEYS.searchRadius, '20');
  setSetting(KEYS.searchRadius, '50');
  await getRecommendations();
  setSetting(KEYS.searchRadius, oldRadius);
}
```

- [ ] **Step 2: 端到端验证**

完整流程测试（需要真实 API Key）：
1. 填写所有 Key → 点击自动定位 → 等待天气加载 → 选标签"开车 + 自然风景" → 输入"想随便转转" → 点击"获取 AI 推荐"
2. 应依次显示"正在搜索附近地点" → "AI 正在分析推荐" → 结果卡片
3. 每张卡片有地点名、距离、AI 推荐理由

**降级测试（无需真实 Key）：**
- 关闭 AI Key → 应显示 POI 列表 + "AI 推荐暂不可用"横幅
- 选一个偏僻位置 + 无匹配类型 → 应显示"未找到"提示 + "扩大至 50km"按钮

- [ ] **Step 3: 最终提交**

```bash
git add 出行推荐/index.html
git commit -m "feat: 出行推荐 - 结果渲染、主流程与错误降级"
```

---

## 自检：Spec 覆盖确认

| Spec 需求 | 对应 Task |
|-----------|----------|
| GPS + 手动输入位置 | Task 3 |
| 高德地理编码（城市名→经纬度） | Task 3 |
| 和风天气 API + 手动天气备选 | Task 4 |
| 出行方式单选标签 | Task 5 |
| 活动类型多选标签 | Task 5 |
| 可选自由文本 | Task 1 (HTML) |
| 高德 POI 周边搜索 | Task 6 |
| AI 推荐（4个 Provider 可切换） | Task 7 |
| 推荐结果卡片列表 | Task 8 |
| 设置面板（Key + 提供商 + 半径） | Task 2 |
| localStorage 持久化 | Task 2 |
| 首次打开自动弹出设置 | Task 2 |
| GPS 失败降级 | Task 3 |
| 天气 API 失败降级 | Task 4 |
| POI 无结果 + 扩大搜索 | Task 8 |
| AI 失败降级展示 POI 列表 | Task 8 |
