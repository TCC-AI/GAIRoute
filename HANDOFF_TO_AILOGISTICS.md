# GAIRoute 舊系統完整交接文件
# 供 YOU-JUE/AILogistics 新系統開發參考

> 本文件由 GAIRoute session 的 Claude Code 整理，涵蓋舊系統所有業務邏輯、資料結構、API 端點、計算公式。
> 新系統開發時，請以此文件作為功能對照與業務規則的唯一參考來源。

---

## 一、系統概述

- **系統名稱**：昶青智慧冷鏈物流 派車GAI數據管理系統
- **版本**：1.0.0
- **最後更新**：2025-06-27
- **架構**：單一 HTML 檔案（index.html, 14,219 行, 494.7KB）+ Google Apps Script 後端
- **部署**：前端 GitHub Pages / 後端 Google Apps Script

---

## 二、外部服務與 API Keys

| 服務 | 用途 | 識別碼/Key |
|------|------|-----------|
| Google Sheets (主) | 主資料庫 | `1miCf_ZTVyzmA1MNbbZM3uUdZjluk99gZUH5w2q_XWs8` |
| Google Sheets (外部) | GAI分析/託收託運/訂單 | `1YMYLTvbAZjUxtOrtavzOCSTt0U5NfZN7S825TQmflbc` |
| GAS 主端點 | 後端 API | `AKfycbxGALCbnR85DT_nTdXiFjWNS58GqSC-RV0Hr0EaineZyR7i-FmolJ0_wruah5VTfqHqBQ` |
| GAS 派車表端點 | 明日派車表 | `AKfycbxH0p4bVcrm89-uXGQVYTNcOznt3xpnicDovrCwbOi89pZan54iCqUPU064iJIEee0Q` |
| GAS 碳排端點 | 碳排圖表資料 | `AKfycbyZkX_tW_3vk1edpwf7fDNVMLB0tpUgvvRda90-lzJFVCrZUHvGaVH-7Gfp7idFDznR5g` |
| Google Maps | 路線規劃/地理編碼 | `AIzaSyDtP8oDCXfr0CPLyVv_WrAqmk2V9QgteLI` |
| OpenWeatherMap | 天氣資料 | `a4806bc2577c0c227f9acd8bccb566a5` |
| ChatGPT (GPT-5) | AI 對話助手 Lisa | 透過 GAS 代理呼叫 |

> **安全注意**：以上 API Keys 在舊系統中硬編碼於前端。新系統務必移至環境變數。

---

## 三、Google Sheets 工作表結構（資料庫 Schema）

### 主 Spreadsheet（SPREADSHEET_ID）

| 工作表名稱 | 用途 | 對應新系統 Model |
|-----------|------|-----------------|
| 管制名單 | 用戶帳號/密碼/權限 | `UserModel` |
| 主控台 | 儀表板統計資料 | `DashboardSnapshotModel` |
| 路線管理 | 路線 CRUD | `RouteModel` |
| GAI-精準智慧物流路徑規劃 | AI 路線優化 | `OptimizationModel` |
| 季節性路線 | 季節性路線 | `SeasonalRouteModel` |
| 載運管理 | 車輛載運 | `CargoAssignmentModel` |
| 即時監控 | 即時資料 | `RealtimeEventModel` |
| 系統設定 | 系統參數 | `SystemSettingModel` |
| 報表分析 | 操作日誌 | `OperationLogModel` |

### 外部 Spreadsheet（EXTERNAL_SPREADSHEET_ID）

| 工作表名稱 | 用途 | 前端引用的 sheetName |
|-----------|------|---------------------|
| 各路線板數 | 各路線每日板數統計 | `'各路線板數'` |
| GAI每日訂單分析 | AI 訂單分析報告 | `'GAI每日訂單分析'` |
| 託收託運回報 | 收送點清單（ChatGPT 查詢用） | `'託收託運回報'` |
| 託收託運回報_篩選 | 上次查詢的篩選結果 | `'託收託運回報_篩選'` |

### 「管制名單」工作表欄位（用戶資料）

```
帳號 (username)
密碼 (password)        ← 明文儲存！新系統改用 bcrypt hash
姓名 (name)
權限等級 (level)       ← s1/s2/s3/s4
角色 (role)
最後登入時間 (lastLoginTime)
```

---

## 四、權限系統（4 級制）

```javascript
PERMISSIONS = {
    's4': {  // 管理者 — 全部功能
        pages: ['dashboard', 'route-management', 'route-optimization',
                'seasonal-routes', 'cargo-vehicle', 'real-time-monitor',
                'system-settings', 'reports', 'route-pallets']
    },
    's3': {  // 主要執行者 — 除了系統設定
        pages: ['dashboard', 'route-management', 'route-optimization',
                'seasonal-routes', 'cargo-vehicle', 'real-time-monitor',
                'reports', 'route-pallets']
    },
    's2': {  // 次要執行者 — 除了系統設定和報表
        pages: ['dashboard', 'route-management', 'route-optimization',
                'seasonal-routes', 'cargo-vehicle', 'real-time-monitor',
                'route-pallets']
    },
    's1': {  // 初階使用者 — 僅儀表板
        pages: ['dashboard']
    }
}
```

**新系統建議**：改為 RBAC（Role-Based Access Control），以 `role` 欄位搭配 `require_admin` / `require_role` 依賴注入。

---

## 五、所有 API 端點（GAS Backend Actions）

### 認證類
| Action | 參數 | 回傳 | 說明 |
|--------|------|------|------|
| `authenticateUser` | `{username, password}` | `{success, user: {username, name, role, level}}` | 登入驗證 |
| `updateLastLoginTime` | `{username}` | — | 更新最後登入時間 |

### 儀表板類（Dashboard）
| Action | 參數 | 回傳 | 說明 |
|--------|------|------|------|
| `getDashboardData` | — | `{todayDeliveries, vehicleUsage, onTimeRate, alertCount}` | 主控台基本資料 |
| `getRoutePalletSum` | `{targetSpreadsheetId}` | `number` | 當日各路線板數總和 |
| `getRoutePalletNonZeroPercentage` | `{targetSpreadsheetId}` | `number` | 非零路線百分比（車輛使用率） |
| `getRouteTableLastBValue` | `{targetSpreadsheetId}` | `number` | 路線表B欄最後一個值（運務指標） |
| `getRouteTableAlertValue` | `{targetSpreadsheetId}` | `number` | 路線表Alert值（異常警示） |

**儀表板 4 大指標對照**：
```
todayDeliveries  → getRoutePalletSum         → 「當日各路線板數」
vehicleUsage     → getRoutePalletNonZeroPercentage → 「車輛使用率 %」
onTimeRate       → getRouteTableLastBValue   → 「上日路線表B欄最後值」
alertCount       → getRouteTableAlertValue   → 「異常警示值」（> 2 時標紅）
```

### 路線管理類
| Action | 參數 | 回傳 | 說明 |
|--------|------|------|------|
| `getRoutes` | — | `Route[]` | 取得所有路線 |
| `addRoute` | `{route}` | — | 新增路線 |
| `editRoute` | `{routeId, route}` | — | 編輯路線 |
| `deleteRoute` | `{routeId}` | — | 刪除路線 |
| `searchRoutes` | `{keyword}` | `Route[]` | 搜尋路線 |
| `importRoutes` | `{routes}` | — | 匯入路線 |
| `exportRouteSheetAsExcel` | `{sheetName}` | `blob` | 匯出單一路線 Excel |
| `exportAllRouteTablesAsExcel` | — | `blob` | 匯出所有路線 Excel |
| `getRouteTableFromExternalSheet` | `{targetSpreadsheetId}` | `RouteData[]` | 從外部 Sheet 取路線資料 |
| `processRouteAddresses` | `{routeData}` | `ProcessedRoute[]` | 處理/地理編碼地址 |
| `getRoutePalletsData` | `{targetSpreadsheetId}` | `PalletData[]` | 取得各路線板數詳細資料 |
| `getRoutePalletNonZeroCount` | `{targetSpreadsheetId}` | `number` | 非零路線數量 |

### AI/ChatGPT 類
| Action | 參數 | 回傳 | 說明 |
|--------|------|------|------|
| `processChatGPTQuery` | `{userMessage, targetSpreadsheetId, sheetName}` | `{reply, mode, timestamp}` | AI 對話查詢 |
| `getLatestOrderAnalysis` | `{targetSpreadsheetId, sheetName}` | `[{A: time, B: content}]` | 最新訂單分析（GAI每日訂單分析表） |
| `getLatestOrderInfo` | `{targetSpreadsheetId}` | `{time, client, ...}` | 最新訂單資訊 |

**AI 回應模式（mode）**：
- `quick_answer` ⚡ 快速回答 — 預設規則匹配
- `ai_analysis` 🤖 AI分析 — ChatGPT 深度分析
- `fallback` 💡 備用回應 — API 失敗時的後備

### 路線優化類
| Action | 參數 | 回傳 | 說明 |
|--------|------|------|------|
| `optimizeRoutes` | `{target, vehicleLimit, timeWindow}` | `OptResult` | 路線優化 |
| `applyOptimization` | `{optimizationId}` | — | 採用優化結果 |

**優化參數**：
- `target`: `'time'`（最短時間）/ `'distance'`（最短距離）/ `'cost'`（最低成本）
- `vehicleLimit`: 車輛數上限（整數）
- `timeWindow`: `{start, end}` 時間窗口

**優化結果**：
```javascript
{
    optimizationId: 'OPT' + timestamp,
    beforeDistance: 285.6,    // km
    afterDistance: 234.2,     // km
    savedDistance: 51.4,      // km
    improvementPercent: 18    // %
}
```

### 收送點/配送類
| Action | 參數 | 回傳 | 說明 |
|--------|------|------|------|
| `getDeliveryPoints` | `{date}` | `DeliveryPoint[]` | 取指定日期收送點清單 |
| `setSpecifiedDateToExternalSheet` | `{date}` | — | 寫入指定日期到外部 Sheet |
| `getFilteredDeliveryReport` | `{targetSpreadsheetId, sheetName}` | `Row[]` | 篩選後的配送報告（有重試機制：最多 3 次，2.5-5 秒間隔） |
| `getLastQueryData` | `{targetSpreadsheetId, sheetName}` | `Row[]` | 上次查詢結果（託收託運回報_篩選） |
| `getLatestDeliveryReport` | `{targetSpreadsheetId}` | `Row[]` | 最新配送快照 |

### 碳排類（獨立 GAS 端點）
| 端點 | 參數 | 回傳 | 說明 |
|------|------|------|------|
| `getCarbonChartData` (GET) | `startDate, endDate` | `{dates[], dispatch[], operation[], storage[], revenue[]}` | 碳排圖表資料 |

### 路線預排三模式（3 種派車策略）
| Action | 參數 | 回傳 | 說明 |
|--------|------|------|------|
| `getCurrentDayRouteTable` | `{targetSpreadsheetId}` | `RouteCard[]` | 模式 A：老師傅模態（經驗導向） |
| `getCurrentDayRouteTableB` | `{targetSpreadsheetId}` | `RouteCard[]` | 模式 B：高裝載模態（最大化車輛利用率） |
| `getCurrentDayRouteTableC` | `{targetSpreadsheetId}` | `RouteCard[]` | 模式 C：低碳排模態（環保優先） |

### 緊急調派與即時監控類
| Action | 參數 | 回傳 | 說明 |
|--------|------|------|------|
| `emergencyDispatch` | `{type, location, description, priority}` | `{emergencyId, message, estimatedResponse}` | 緊急調派 |
| `realTimeReroute` | `{routeId, waypoints}` | — | 即時改路 |
| `rerouteAllVehicles` | — | — | 全車隊改路 |

**緊急類型**：`accident`（交通事故）/ `breakdown`（車輛故障）/ `urgent`（緊急配送）/ `weather`（天氣異常）
**優先級**：`high` / `medium` / `low`

### 明日建議派車表（獨立 GAS 端點）
| Action | 參數 | 回傳 | 說明 |
|--------|------|------|------|
| `getTomorrowDispatchTable` | `{}` | `data[]` | 明日建議派車表 |

### 其他
| Action | 參數 | 回傳 | 說明 |
|--------|------|------|------|
| `initializeSheets` | — | `{message}` | 初始化工作表 |

---

## 六、前端 API 呼叫統一格式

```javascript
// 所有 API 呼叫都走這個模式
async apiCall(action, data = {}) {
    const payload = {
        action,           // 對應 GAS 的 switch-case
        data,             // 業務參數
        timestamp: Date.now(),
        user: currentUser.username || 'anonymous'
    };

    const response = await fetch(CONFIG.GAS_URL, {
        method: 'POST',
        headers: { 'Content-Type': 'text/plain' },
        body: JSON.stringify(payload),
        mode: 'cors'
    });

    const result = JSON.parse(await response.text());
    // result = { success: boolean, data: any, error?: string }
    return result.data;
}
```

**新系統對照**：每個 `action` 對應一個 FastAPI endpoint：
- `authenticateUser` → `POST /api/auth/login`
- `getDashboardData` → `GET /api/dashboard`
- `getRoutes` → `GET /api/routes`
- `optimizeRoutes` → `POST /api/optimization/optimize`
- `processChatGPTQuery` → `POST /api/ai/chat`
- 以此類推

---

## 七、碳排計算公式

### 4 個資料集
| 資料集 | 單位 | 說明 |
|--------|------|------|
| 派車GAI (dispatch) | 公噸 | 派車碳排 |
| 運務GAI (operation) | (省)小時 | 運務效率 |
| 託收托運GAI (storage) | 公斤 | 託收托運碳排 |
| 營業額 (revenue) | 元 (TWD) | 營業額 |

### 統計計算函數
```python
# 新系統 Python 等效實作

def calculate_total(data: list[float]) -> float:
    """總計"""
    valid = [v for v in data if v is not None and not math.isnan(v)]
    return sum(valid)

def calculate_average(data: list[float]) -> float:
    """平均值"""
    valid = [v for v in data if v is not None and not math.isnan(v)]
    return sum(valid) / len(valid) if valid else 0

def calculate_std_dev(data: list[float]) -> float:
    """標準差"""
    valid = [v for v in data if v is not None and not math.isnan(v)]
    if not valid:
        return 0
    avg = calculate_average(valid)
    square_diffs = [(v - avg) ** 2 for v in valid]
    return math.sqrt(sum(square_diffs) / len(square_diffs))

def calculate_growth_rate(data: list[float]) -> float:
    """增長率（百分比）"""
    valid = [v for v in data if v is not None and not math.isnan(v)]
    if len(valid) < 2 or valid[0] == 0:
        return 0
    return ((valid[-1] - valid[0]) / valid[0]) * 100

def calculate_median(data: list[float]) -> float:
    """中位數"""
    valid = sorted([v for v in data if v is not None and not math.isnan(v)])
    if not valid:
        return 0
    mid = len(valid) // 2
    if len(valid) % 2 == 0:
        return (valid[mid - 1] + valid[mid]) / 2
    return valid[mid]
```

### 總碳排量計算
```python
total_carbon = {
    'total': dispatch.total * 1000 + operation.total + storage.total,  # 公斤
    'avg':   dispatch.avg * 1000 + operation.avg + storage.avg,
    'max':   dispatch.max * 1000 + operation.max + storage.max,
    'min':   dispatch.min * 1000 + operation.min + storage.min
}
# 注意：dispatch 單位是公噸，需 ×1000 轉公斤
```

### AI 洞察生成邏輯
```python
# 碳排效率 = 每千元營收的碳排量
carbon_per_revenue = (dispatch.total + operation.total + storage.total) / revenue.total * 1000

# 最大碳排來源判斷
max_source = max(dispatch.total * 1000, operation.total, storage.total)

# 整體趨勢
total_growth = dispatch.growth + operation.growth + storage.growth
# > 0: 上升趨勢, < 0: 下降趨勢, ≈ 0: 穩定

# 穩定度
avg_std_dev = (dispatch.std_dev + operation.std_dev + storage.std_dev) / 3
# < 10: 非常穩定, < 50: 穩定, ≥ 50: 波動大
```

---

## 八、儀表板快取機制

```python
# DashboardCache 實作規格

CACHE_KEY = 'dashboard_cache_v2'
DEFAULT_TTL = 30 * 60 * 1000  # 30 分鐘（毫秒）

cache_format = {
    'ts': timestamp,  # 儲存時間
    'data': {
        'pallets': route_pallet_sum,          # 各路線板數總和
        'nonzeroPct': non_zero_percentage,     # 非零路線百分比
        'lostB': route_table_last_b_value,     # 路線表B欄最後值
        'alert': route_table_alert_value       # 異常警示值
    }
}

# 快取策略：
# 1. 頁面載入 → 檢查快取是否有效（30分鐘內）
# 2. 有效 → 立即顯示快取資料
# 3. 無效/過期 → 4 個 API 並行呼叫 → 更新快取
```

**異常警示規則**：`alertValue > 2` 時顯示紅色警示樣式

---

## 九、明日派車表 AI 生成邏輯

### Prompt 建構
```
請依據以下收送點清單，依照「北、中、南、東」分區，
並依據每個地點的板數，規劃明天的派車路線
（每條路線請標示路線名稱、起點、各收送點及板數，格式如下）：

路線名稱 起點 第一收送點(板數) 第二收送點(板數) ...

收送點清單：
{name}（{area}區，{pallet}板）
...

請用 markdown 格式回覆，並分區顯示，每區多條路線，路線可用 <details> 收折。
```

### 收送點資料欄位對應
```python
name   = item.get('收送點') or item.get('地址') or item.get('地點') or item.get('名稱')
area   = item.get('區域') or item.get('分區')
pallet = item.get('板數') or item.get('板數量') or item.get('板')
```

---

## 十、路線資料結構

```python
# Route 物件
route = {
    'id': 'R001',
    'name': '台北市區線',         # 路線名稱/工作表名稱
    'startPoint': '台北車站',     # 起點
    'endPoint': '信義區',         # 迄點
    'stopCount': 8,               # 途經點數量
    'estimatedTime': 120,         # 預估時間（分鐘）
    'status': '啟用',             # 啟用/停用
    'totalDistance': 0,           # 總公里數
    'addresses': {                # 最多 20 個地址
        '地址1': '...',
        '地址2': '...',
        # ... up to 地址20
    }
}
```

### 板數資料結構（Route Pallets）
```python
# 每條路線的板數統計
pallet_data = {
    'route_name': '路線名稱',
    'pallet_count': 15,           # 板數
    'is_non_zero': True,          # 是否非零
    'last_updated': '2025-06-27'  # 最後更新時間
}

# 快取 TTL: 5 分鐘
```

---

## 十一、Google Maps 整合

### 使用的服務
- **Directions Service**：路線規劃（起點→途經點→終點）
- **Geocoding Service**：地址轉座標
- **Distance Matrix**：距離/時間矩陣
- **Places Autocomplete**：地址自動完成
- **Traffic Layer**：即時路況
- **Transit Layer**：大眾運輸
- **Bicycling Layer**：自行車道

### 預設座標（台北）
```python
DEFAULT_CENTER = {'lat': 25.0330, 'lng': 121.5654}  # 台北市中心
```

---

## 十二、天氣服務

### OpenWeatherMap API
```python
# 端點
url = f"https://api.openweathermap.org/data/2.5/weather?lat={lat}&lon={lng}&appid={API_KEY}&units=metric&lang=zh_tw"

# 回傳欄位
weather = {
    'temperature': float,      # 攝氏溫度
    'description': str,        # 天氣描述（中文）
    'visibility': float,       # 能見度
    'wind_speed': float,       # 風速
    'humidity': float,         # 濕度 %
    'feels_like': float        # 體感溫度
}
```

---

## 十三、AI 秘書 Lisa

### 角色設定
- **名稱**：TCC-GAI派車中心數位主任 Lisa
- **功能**：
  1. 路線規劃建議
  2. 配送狀態查詢
  3. 車輛調度分析
  4. 異常情況處理建議

### R2 角色動畫系統
- **圖片**：R0.png, R1.png, R2.png（在 repo 根目錄）
- **表情狀態**：
  - `normal`：一般待機
  - `thinking`：滑鼠懸停 / AI 處理中
  - `working`：發送查詢時
  - `happy`：收到回應時（持續 1 秒後恢復）

### 對話流程
```
用戶輸入 → R2 進入 working 狀態
    → apiCall('processChatGPTQuery', {userMessage, targetSpreadsheetId, sheetName})
    → 收到回應 → R2 進入 happy 狀態（2秒後恢復 normal）
    → 顯示回應 + mode 標籤（⚡快速回答 / 🤖AI分析 / 💡備用回應）
```

---

## 十四、系統健康檢查

### 4 項檢查
| 項目 | 方法 | 說明 |
|------|------|------|
| API | `app.checkConnection()` | GAS 後端連線 |
| Maps | `window.google.maps` 存在 | Google Maps SDK |
| Weather | 呼叫 OpenWeatherMap | 天氣服務可用 |
| Storage | localStorage 讀寫測試 | 本地儲存 |

### 健康度判定
```python
percentage = passed_checks / total_checks * 100
if percentage >= 80: return 'healthy'
if percentage >= 60: return 'warning'
return 'critical'
```

**執行頻率**：每 5 分鐘自動檢查

---

## 十五、離線模式與 Mock Data

當網路斷線時：
1. `isOnline = False`
2. `USE_MOCK_DATA = True`
3. 所有 API 呼叫回傳 mock 資料
4. 顯示離線圖示

### Mock 資料預設值
```python
mock_dashboard = {
    'palletSum': 163,
    'nonZeroPercentage': 10,
    'lastBValue': 7436.04,
    'alertValue': 2.01
}

mock_optimization = {
    'beforeDistance': 285.6,
    'afterDistance': 234.2,
    'savedDistance': 51.4,
    'improvementPercent': 18
}

mock_emergency = {
    'emergencyId': 'EMG' + timestamp,
    'message': '緊急調派已啟動',
    'estimatedResponse': '15分鐘內'
}
```

---

## 十六、前端頁面結構

| # | 頁面 ID | 名稱 | 功能狀態 |
|---|--------|------|---------|
| 1 | `dashboard` | 主控台 | ✅ 完整 |
| 2 | `gai-assistant` | GAI秘書 | ✅ 完整 |
| 3 | `route-optimization` | 路線優化 | ✅ 完整 |
| 4 | `route-management` | 路線管理 | ✅ 完整 |
| 5 | `route-pallets` | 各路線板數 | ✅ 完整 |
| 6 | `seasonal-routes` | 季節性路線 | 🚧 開發中 |
| 7 | `cargo-vehicle` | 載運管理 | 🚧 開發中 |
| 8 | `real-time-monitor` | 即時監控 | 🚧 開發中 |
| 9 | `system-settings` | 系統設定 | ⚠️ 僅 s4 權限 |
| 10 | `reports` | 報表分析 | ⚠️ 僅 s3+ 權限 |

---

## 十七、鍵盤快捷鍵

| 快捷鍵 | 功能 | 條件 |
|--------|------|------|
| `Ctrl+R` / `F5` | 重新整理儀表板 | 僅在 dashboard 頁面 |
| `Esc` | 關閉 Modal | 任何頁面 |
| `Ctrl+M` | 切換地圖視圖 | 僅在 dashboard 頁面 |

---

## 十八、記憶體監控

```python
# 每 60 秒檢查
# 如果 JS heap 使用量 > 80%：
#   1. 發出警告
#   2. 清理快取（CacheManager.clear()）
```

---

## 十九、匯出功能

| 功能 | 格式 | 檔名模式 |
|------|------|---------|
| 碳排統計 | CSV (UTF-8 BOM) | `{reportName}_{date}.csv` |
| 優化結果 | JSON | `優化結果_{date}.json` |
| 單一路線表 | Excel | 由 GAS 產生 |
| 全部路線表 | Excel | 由 GAS 產生 |
| 明日派車表 | HTML 表格匯出 | — |

---

## 二十、關鍵業務邏輯總結（給新系統開發者的 Checklist）

### 必須保留的核心邏輯
- [ ] 儀表板 4 大指標的計算方式（板數總和、非零%、B欄值、警示值）
- [ ] 碳排統計 7 項指標（total/avg/max/min/stdDev/median/growth）
- [ ] 碳排效率計算（每千元營收碳排量）
- [ ] 總碳排公式（dispatch×1000 + operation + storage）
- [ ] 異常警示閾值（> 2 標紅）
- [ ] 明日派車表 AI Prompt 格式（北中南東分區、含板數）
- [ ] AI 三種回應模式（quick_answer / ai_analysis / fallback）
- [ ] 4 級權限系統（s1~s4）
- [ ] 路線最多 20 個地址
- [ ] 快取 TTL（儀表板 30 分鐘、板數 5 分鐘）
- [ ] 離線 mock data 機制

### 可以改進的部分
- [ ] 密碼改用 bcrypt hash（舊系統明文）
- [ ] API Key 移至環境變數（舊系統硬編碼前端）
- [ ] 認證改用 JWT（舊系統每次傳帳密）
- [ ] 加入 Rate Limiting（舊系統無）
- [ ] 加入 CORS 白名單（舊系統 GAS 無法控制）
- [ ] 前端拆分為模組化檔案（舊系統單一 14K 行 HTML）
- [ ] 資料庫改用 PostgreSQL（舊系統 Google Sheets 10萬行上限）
- [ ] 加入自動化測試（舊系統 0 個測試）
- [ ] 季節性路線、載運管理、即時監控功能需完成開發
